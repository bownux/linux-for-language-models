# Chapter 4 — Services Without a Status Screen

*Draft status: author draft, gate-checked; human verification pending. This chapter's
worked postmortem examines a real failed unit on the authoring machine, with its real
outputs; nothing in it was staged.*

## Status is a poster; show is a socket

`systemctl status` is one of the most-typed commands on any systemd machine, and it is
a human display through and through: colored dots, a tree of processes, the last few
log lines inlined, all of it paged when the output runs long, none of its layout
promised to stay put between versions. The transcript-mode operator's counterpart is
`systemctl show` — the same facts as `KEY=VALUE` pairs, every key documented, no
pager, no color, and a `-p` flag that selects exactly the properties your question
needs:

```bash no-run
systemctl show -p Description,ActiveState,SubState,MainPID systemd-journald.service
```

```output
Description=Journal Service
ActiveState=active
SubState=running
MainPID=1326
```

The shape should feel familiar by now: it is the porcelain rule from chapter 2 wearing
systemd's uniform. `ActiveState` is the coarse answer (`active`, `failed`,
`inactive`); `SubState` refines it (`running`, `dead`, `exited` — a service can be
`active (exited)` legitimately, as oneshot units are); `MainPID` hands you the number
that unlocks all of chapter 3's per-process reads. The property list runs to hundreds
— `systemctl show` with no `-p` dumps them all, and one unbounded dump per unfamiliar
unit type is a reasonable investment to learn what is on offer. The properties this
book leans on most: `Result` and `ExecMainStatus` (how the last run ended — the unit's
own memory of its exit), `NRestarts` (how many times systemd has already picked the
service back up, a number that turns "it's running" into "it's crash-looping"),
`ExecMainStartTimestamp` (running *since when* — freshness matters when a restart is
part of the story), and `FragmentPath` plus `DropInPaths` (which files on disk define
this unit — the bridge to chapter 5's editing).

The economics repay a comparison. `status` spends its output on being glanceable;
`show -p` spends nothing it was not asked for. Five properties cost five lines, land
already parseable (`grep '^ActiveState='` or a shell `while IFS== read` loop), and
diff cleanly against the same five properties in yesterday's transcript. In a register
that pays per line carried, the poster is a luxury and the socket is the tool.

## Exit codes as sensors

Alongside `show`, systemctl carries a family of commands designed for scripts first
and eyes second — predicates whose real answer is the exit status, with the printed
word as a courtesy. `is-active` answers zero only for an active unit; `is-enabled`
answers for the boot configuration; `is-failed` answers zero when the unit *is*
failed — the predicate affirms its own name, so a zero from `is-failed` is bad news
delivered in good grammar. The one to run first, on any machine you have just been
handed, asks the whole system:

```bash no-run
systemctl is-system-running
echo "verdict status: $?"
```

```output
degraded
verdict status: 1
```

That is the authoring machine, answering honestly: `degraded` means the system is up
but at least one unit has failed, and the nonzero status makes the answer usable by a
script without parsing the word. (The healthy answer is `running`, status 0; a machine
mid-boot answers `starting`.) The measured pair — word for the transcript, number for
the branch — is the two-audience discipline of chapter 2 built directly into the
tool, and it makes the next question mechanical: *which unit?*

```bash no-run
systemctl list-units --failed --no-legend --no-pager --plain
```

```output
gpu-power-cap.service loaded failed failed Cap GPU power limits (RTX PRO 6000 -> 500W) to prevent PSU transient trips
```

One line, one culprit, real: a unit that exists to cap the power draw of the machine's
GPU — the same GPU that serves the inference processes chapter 3's `ps` found — so
that transient spikes do not trip the power supply. The three flags on that shot are
the register's standard systemctl seasoning and worth fixing as a habit:
`--no-legend` strips the header and footer rows that exist for eyes, `--no-pager`
disarms the chapter 1 trap explicitly rather than trusting isatty detection, and
`--plain` flattens the decorative tree characters that would otherwise salt the first
column. What remains parses on whitespace: unit, load state, active state, sub-state,
then the free-text description.

## A real postmortem, one shot at a time

The failed unit above is this chapter's case, worked with the machine's actual
evidence in the order a transcript-mode operator gathers it. The confirmation shot
comes first, because a sweep's output may be minutes old by the time you act on it:

```bash no-run
systemctl is-failed gpu-power-cap.service
echo "confirmed failed: $?"
```

```output
failed
confirmed failed: 0
```

Zero from `is-failed`: the predicate affirms. Next, the unit's own memory of what
happened, from the properties chosen for exactly this question:

```bash no-run
systemctl show gpu-power-cap.service -p Result,ExecMainStatus,ExecMainStartTimestamp,NRestarts
```

```output
Result=exit-code
NRestarts=0
ExecMainStartTimestamp=Mon 2026-08-24 12:57:51 PDT
ExecMainStatus=2
```

Four lines carrying a complete preliminary story. `Result=exit-code` says the failure
mode was the process's own exit, not a timeout, a signal, or a watchdog.
`ExecMainStatus=2` gives the exit status of the main process — and chapter 2's
contract reading applies to services exactly as to shots: status 2 is the tools'
customary "misuse or real error", distinctly not a clean refusal. `NRestarts=0` says
systemd did not retry — either restart policy is off or the failure predates any
retry budget. And the timestamp places the event: `uptime -s` on the same machine
reports boot at `12:57:37` the same day, so the service tried once, fourteen seconds
into boot, failed, and has been failed for the three days since. No log has been read
yet; two `systemctl` reads produced when, how, how often, and how badly.

The log should be next, and the log is where the case turns into a lesson this book
could not have staged better:

```bash no-run
journalctl -u gpu-power-cap.service --no-pager -n 8 -o short-iso 2>&1 | tail -n 8
echo "status: $?"
```

```output
-- No entries --
status: 0
```

An empty answer, delivered with a success status. Chapter 2 called emptiness the most
ambiguous sentence a transcript can contain, and here is the ambiguity with stakes:
*no entries* could mean the process wrote nothing before dying — plausible for a
script failing at its first line — or it could mean something else entirely. The
shape check catches it: a process that exited with status 2 *almost always* said
something on stderr first, and stderr from services lands in the journal. Evidence
missing that should exist is itself evidence. The resolving read costs one shot:

```bash
id -nG | tr " " "\n" | awk '/^(systemd-journal|adm|root|wheel)$/ {n++} END {print n+0}'
```

```output
1
```

One qualifying group — and on inspection it is `wheel`, which grants sudo eligibility
but *not* journal access. On a systemd machine, the system journal is readable only by
root and members of groups like `systemd-journal` and `adm`; an unprivileged
`journalctl` quietly shows only the user's own journal, and for a system unit that
means: no entries, status 0, a calm face on a permission boundary. The trap is worth
the italics: **the journal does not say "permission denied"; it says "nothing here",
and the difference between those sentences is a wrong diagnosis.** The operator's
resolution is explicit privilege — `sudo journalctl -u gpu-power-cap.service`, a
fragment here by this book's rules — or membership in `systemd-journal`, a one-time
grant that makes every future diagnostic read cheaper and is the standard provisioning
choice for exactly this book's reader. The case closes with the unprivileged
evidence in hand: unit failed at boot, exit status 2, no retries, logs unreadable
from this identity — and, per chapter 1's boundary discipline, a finding that names
what it could not see is a finished finding, not a failed one.

## The journal, bounded and structured

When you do hold journal access, `journalctl` is the machine's flight recorder, and
everything chapter 2 said about bounding and structure applies with force, because
the journal is effectively bottomless. The bounding flags come first in every
composed read: `-u <unit>` scopes to a service; `--since` and `--until` take both
timestamps and English (`--since "1 hour ago"`, `--since today`); `-n` caps the line
count; `-p err` and friends filter by priority, so a first look at a sick machine is
often `journalctl -p err --since "1 hour ago" -n 50`. Output format is the second
choice: `-o short-iso` replaces the default's localized month names with sortable
ISO timestamps (chapter 2's determinism rule); `-o cat` strips metadata entirely,
right when a service's raw stderr is the object of study; and `-o json` emits one
JSON object per entry, with every field the journal indexes — the message, the unit,
the PID, the priority, the monotonic timestamp — addressable by name, chapter 3's
JSON turn applied to logs.

One journal facility is so precisely shaped for this register that it reads as if
designed for it: the cursor. Every entry carries an opaque position token, and
`--cursor-file=FILE` makes a read *start where the last read using that file ended*,
writing the new position back when done. A transcript-mode operator monitoring a
service across turns — an agent checking a deploy each visit, a cron'd health report
— reads with a cursor file and receives exactly the entries that arrived since its
last look: no overlap to deduplicate, no gap to worry over, no "tail and hope" — and
the file itself is durable state of the kind chapter 8 will formalize, a bookmark the
next turn's operator (you, remembering nothing) inherits from this one.

```bash fragment
# Incremental read: each invocation returns only what is new since the last one.
journalctl -u myservice.service --cursor-file="$HOME/.cache/myservice.cursor" \
  --no-pager -o short-iso
```

## Units on disk: where a service's definition lives

Every read so far has queried systemd's memory; the definition it remembers came from
files, and the bridge between the two is a pair of properties this book's editing
chapter will depend on:

```bash no-run
systemctl show systemd-journald.service -p FragmentPath,UnitFileState
```

```output
FragmentPath=/usr/lib/systemd/system/systemd-journald.service
UnitFileState=static
```

`FragmentPath` is the answer to *which file defines this unit* — asked constantly,
guessed incorrectly almost as often, because unit files legitimately live in several
places with a precedence order: the distribution installs under `/usr/lib/systemd/
system`, local administration overrides under `/etc/systemd/system`, and runtime
generators synthesize under `/run`. A unit can also be modified without replacing its
file at all, through drop-in directories — `<unit>.d/*.conf` fragments that override
individual settings — and those appear in the sibling property `DropInPaths`. The
one-shot rule: never reason from where a unit file *should* be; ask `FragmentPath`
and `DropInPaths`, and read what they name. `systemctl cat <unit>` performs exactly
that assembly for you — the file plus every drop-in, concatenated with their paths as
comments — and earns a place in the diagnostic sequence right after `show` (with
`--no-pager`, faithfully; it is a chapter 1 pager tool otherwise).

`UnitFileState` closes a distinction that bites operators who conflate it with
`ActiveState`: `enabled` and `disabled` describe *boot wiring*, not present activity.
A unit can be active yet disabled (started by hand, will vanish at reboot — the
classic "it worked until the maintenance window" incident, laid dormant weeks in
advance) or enabled yet inactive (crashed, and nothing noticed). `static`, as above,
means the unit has no install section at all and is wired by dependency rather than
by choice. The pairing to check when handing a machine back — chapter 8 will insist
— is ActiveState *and* UnitFileState together: what is true now, and what will be
true after the next reboot, are separate facts with separate flags.

## The user manager, and the empty-environment trap

systemd machines run a second, less famous constellation: per-user managers, started
at login, controlling units under `~/.config/systemd/user/` — the natural home for
an unprivileged operator's own services and timers, and therefore for much of what
this book's reader will actually deploy. The commands are the same with `--user`
appended; the trap is how that flag fails in exactly the environments this book's
operators inhabit. Measured on the authoring machine, from a deliberately stripped
environment of the kind cron jobs, CI steps, and agent harnesses live in:

```bash no-run
systemctl --user is-active default.target 2>&1
echo "status: $?"
```

```output
Failed to connect to user scope bus via local transport: $DBUS_SESSION_BUS_ADDRESS and $XDG_RUNTIME_DIR not defined (consider using --machine=<user>@.host --user to connect to bus of other user)
status: 1
```

The user manager is running; the *command cannot find it*, because the rendezvous
happens over a session bus whose address lives in environment variables that
interactive logins export and stripped environments do not. The repair is one
variable, constructed from facts already in hand:

```bash no-run
XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user is-active default.target 2>&1
echo "status: $?"
```

```output
active
status: 0
```

Same command, same machine, opposite verdict — the pair is this book's cleanest
specimen of a rule chapter 1 stated abstractly: in the non-interactive register, the
*environment is part of the question*, and an error message about connection is not
evidence that the thing you asked about is down. (The generalization: `sudo` also
strips environment; the difference between "the service is broken" and "my shot
could not reach the service" is checked by asking who failed — connection errors
implicate the asker.) One companion fact completes the user-manager picture: by
default, a user's manager — and every service under it — stops when their last
session ends, which for an operator deploying long-running work from an ssh one-shot
means the work dies at hangup. The grant that changes this is lingering
(`loginctl enable-linger <user>`, privileged, a fragment by this book's rules),
which keeps the user manager alive from boot; it is the single systemd fact most
often missing from "my service vanished when I logged out" incidents.

## Changing state, with proof

Reading services is unprivileged; changing them is not, so this section is fragments
by the book's own rules — but the *shape* of a state-changing shot matters more than
its privilege. The naive change is `systemctl restart myservice.service`, dispatched
alone, its silence on success read as good news. The register's version carries its
proof, chapter 2's ask-and-verify with service semantics:

```bash fragment
sudo systemctl restart myservice.service \
  && systemctl is-active myservice.service \
  && systemctl show myservice.service -p ExecMainStartTimestamp,NRestarts
```

Three answers in one transcript: the restart's own status, the predicate confirming
the unit settled active rather than flapping, and the timestamp proving the running
process is *new* — because a restart that silently failed to kill the old process is
a known failure shape, and freshness is the property the timestamp checks. For a
service with a real interface, one more link belongs on the chain: the functional
probe — `curl --fail` against its health endpoint, a query against its socket —
because "systemd considers it active" and "it answers" are different facts, and the
second is the one the machine's users experience. Two operational footnotes complete
the pattern: after editing any unit file, `systemctl daemon-reload` must precede the
restart, or systemd restarts the service under the *old* definition while the new
one sits unread on disk — a mismatch between disk and memory that produces the
register's most confusing five minutes; and `enable --now` is the idiom that both
starts a service and persists it across boots, the two halves of "turn it on" that
`start` alone quietly leaves separate.

## Reading a boot you did not attend

Chapter 3's introduction shot flagged short uptimes as findings, and services are
where a reboot's consequences surface — the disabled-but-active unit vanishing, the
enabled-but-broken one failing on schedule, fourteen seconds in, like this
chapter's case study. systemd ships a dedicated read for the boot it performed
while nobody watched:

```bash no-run
systemd-analyze
echo "status: $?"
```

```output
Startup finished in 1min 4.263s (firmware) + 3.054s (loader) + 3.130s (kernel) + 1.933s (initrd) + 8.669s (userspace) = 1min 21.051s
graphical.target reached after 8.499s in userspace.
status: 0
```

The authoring machine's last boot, decomposed by stage — and the transcript reads
itself: userspace took under nine seconds, while the *firmware* spent a leisurely
minute before Linux existed at all, which is exactly the kind of fact that
redirects a "boots are slow" investigation away from every service on the machine.
The refinement, `systemd-analyze blame --no-pager`, ranks individual units by
startup cost (on this machine, the household's own report generator tops the list
at 22 seconds — unprivileged, honest, and slightly embarrassing for the
household). Both reads work from the ordinary seat.

One caution transfers from the permission lesson. `journalctl --list-boots`
enumerates the boots the journal can show *to you* — on the authoring machine it
reports the current boot's entries beginning hours after the actual boot time
`uptime -s` states, because the unprivileged view opens where the user's own
first process began logging, not where the kernel did. The boot list, like every
journal read, is a view through an identity; reconcile it against `uptime -s`
(whose source is `/proc`, identity-blind) before concluding anything about when
or how often a machine restarted.

## What a service costs, asked the same way

One more family of `show` properties completes the reading toolkit, because
"running" and "running away" look identical from `ActiveState`. systemd tracks each
service inside its own control group, and the accounting surfaces as properties —
which means resource questions get asked in the same porcelain dialect as
everything else in this chapter:

```bash no-run
systemctl show systemd-journald.service -p MemoryCurrent,TasksCurrent,CPUUsageNSec
```

```output
MemoryCurrent=72110080
CPUUsageNSec=141585922000
TasksCurrent=1
```

The journal daemon on the authoring machine: about 69 MiB resident, one task, 141
seconds of accumulated CPU over the boot. Two of chapter 3's disciplines transfer
directly. `CPUUsageNSec` is an accumulator, so a *rate* needs the counter-gap-
counter treatment — two reads a minute apart, subtracted, turn "141 seconds since
boot" into "how hard is it working now". And the properties beat their `ps`
equivalents for the same reason `MemAvailable` beat the folk formula: the cgroup
figure covers the *whole* unit — every process the service spawned, including the
short-lived ones sampling misses — because the accounting is structural, not
snapshot. The pairing to watch in the wild: `TasksCurrent` climbing across
transcripts is a leak of processes; `NRestarts` climbing is a crash loop;
`MemoryCurrent` climbing without either is the service itself remembering too
much. Three counters, three different conversations with whoever maintains the
service — and all of them one `show` away, unprivileged.

## Timers: the scheduler that answers questions

The unattended scheduling this book's mode descends from — chapter 1's cron heritage
— has, on systemd machines, a native successor with far better transcript manners.
The authoring machine makes the point bluntly: it has no `crontab` binary at all
(measured during the writing of this chapter — `command -v crontab` answers nothing),
and its scheduled work is timer units:

```bash no-run
systemctl list-timers --no-pager --no-legend --plain | head -n 4 \
  | awk '{for (i=1; i<=NF; i++) if ($i ~ /\.timer$/) {print $1, $2, $3, $i; break}}'
```

```output
Fri 2026-08-28 06:05:00 rog-life-report-morning-brief-dream.timer
Fri 2026-08-28 08:05:00 rog-life-report-morning-brief-deliver.timer
Fri 2026-08-28 13:12:39 systemd-tmpfiles-clean.timer
Mon 2026-08-31 00:19:41 fstrim.timer
```

The awk scan for the field ending in `.timer`, rather than a fixed column number, is
a scar with a story. This listing's first draft selected the unit name by position —
field eleven — and worked; run again an hour later, it printed the word `ago`,
because `list-timers` renders elapsed time in human units, and a timer's "15h ago"
had become "3 days" somewhere in the table, changing the field count of its row.
Human-layout output does not merely *risk* drifting between versions, as the
porcelain rule warns; it can drift between *invocations*, and the register's
defense, when no `--json` or porcelain mode is on offer, is to anchor on the shape
of the wanted value itself rather than on where it stood. (Newer systemd does offer
`--output=json` for exactly this table; the anchor trick is for the tools and
versions that do not.)

Real again, and quietly personal: alongside the distribution's own maintenance
timers run two belonging to the operator's household automation — a machine this
book's author shares with other unattended operators, all of them scheduled through
the same mechanism. The transcript advantages over classic cron are exactly the
themes of this chapter. A timer is a unit, so the whole read toolkit applies:
`list-timers` answers *when next and when last* — a question crontab files simply
cannot answer, since cron persists no last-run record — and the scheduled job's
output lands in the journal under the service's own name, not in a root mailbox
nobody reads. The failure of a scheduled job is a *failed unit*, visible to this
chapter's first sweep, rather than a silence. And the schedule itself lives in a
file with `OnCalendar=` syntax that `systemd-analyze calendar '...'` will dry-run
for you — chapter 6's rehearsal principle available for time itself:

```bash fragment
# Will this expression fire when I believe it will? Ask before installing it.
systemd-analyze calendar "Mon..Fri 06:05" --iterations=3
```

For the reader on a cron machine, the classic discipline still holds — `crontab -l`
to read, environment pinned inside the job, output redirected somewhere durable —
but the migration logic points one way: the register runs on evidence, and of the
two schedulers, only one keeps records.

A machine's services, read without a status screen, changed only with proof, and
scheduled by a mechanism that remembers — that is the operational half of the
system. What remains before the dangerous chapters is the substrate everything
configures itself through: files, edited by an operator with no editor. That is
chapter 5.
