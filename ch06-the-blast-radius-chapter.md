# Chapter 6 — The Blast Radius Chapter

*Draft status: author draft, gate-checked; human verification pending. Every
destructive mechanism in this chapter is demonstrated inside a scratch directory the
listing itself creates, or shown as a printed plan rather than an execution. That
caging is not stagecraft; it is the chapter's own doctrine, practiced on itself.*

## Finality, as an engineering requirement

Chapter 1 named the cost: a one-shot operator's last influence over a command ends at
dispatch. There is no watching the first deletions scroll past, no Ctrl-C at the
moment doubt arrives, no glance at the prompt's directory before pressing enter —
every safety mechanism interactive humans actually rely on turns out to live *during*
execution, in the seconds this register does not experience. What remains is
composition time, and this chapter is about spending it. The doctrine in one line:
**a destructive command must be made safe before it runs, because nothing can make it
safe while it runs.** The good news, argued through chapter 3, is that the
overwhelming majority of administration is reading, which needs none of this
chapter. The discipline concentrates on the minority of shots that write, remove, or
reach outside the machine — and because they are a minority, the register can afford
to make each one carry guards that would feel ceremonious at an interactive prompt.
That asymmetry — free reads, ceremonial writes — is the whole risk posture of a
well-run non-interactive operator.

## The dominant accident: the space in the name

Ask interactive administrators about catastrophic commands and they describe typos.
The register's dominant accident class is different and more mechanical: **an
expansion changed the command between composition and execution.** The shell rewrites
your command before running it — variables expand, the results split into words on
whitespace, the words glob against the filesystem — and each rewrite is a place where
the command you composed and the command that runs can diverge. The canonical
specimen, caged in a scratch directory:

```bash
cd "$(mktemp -d)"
f="release notes.txt"
touch "$f" notes.txt
rm $f 2>&1
printf "%s\n" *
```

```output
rm: cannot remove 'release': No such file or directory
release notes.txt
```

Read the transcript as the four-question routine demands, because it is a small
horror story. The unquoted `$f` split into two words, `release` and `notes.txt`. The
first named nothing — hence the error line. The second named an *innocent bystander*,
which `rm` removed without comment. The final line is the listing's own survivor
roll: the target, `release notes.txt`, stands untouched — and `notes.txt`, created
two commands earlier, is not in the list, because it no longer exists. One unquoted
expansion produced a command that failed at its purpose *and* destroyed something
unrelated, with an error message pointing at neither fact. The cure costs two characters: `rm "$f"` names one file, spaces included,
always. The rule admits no judgment calls: **every expansion is quoted —
`"$var"`, `"$(cmd)"`, `"$@"` — unless splitting is the explicit, commented intent.**
Not because every variable will contain a space, but because the operator composing
the shot cannot see the value at composition time, and this register has no other
moment to check. A mechanical rule for a mechanical accident; `shellcheck` enforces
it (its finding SC2086 is precisely this), and running ShellCheck over any script
before dispatch is the register's equivalent of proofreading — a static gate for the
class of accident no runtime gate can catch in time.

The expansion accident has a second, worse form: the variable that expands to
*nothing*. Chapter 2 previewed it; here is its anatomy, demonstrated as a printed
plan rather than an execution, which is how a plan this dangerous should first exist:

```bash no-run
prefix=""
echo "would run: rm -rf ${prefix}/cache"
```

```output
would run: rm -rf /cache
```

An empty `prefix` — unset, typo'd, or emptied by a failed substitution upstream —
and the composed path collapses to an absolute path at the filesystem root. Every
veteran of this register knows an incident of this shape; the best-documented
public specimen shipped in Steam's Linux client, whose startup script ran
`rm -rf "$STEAMROOT/"*` and, on machines where the variable came up empty,
deleted every file the user could reach — bug report in the references, preserved
complete with its disbelieving comment thread. The defenses stack. `set -u` aborts on the
unset case. The shell's own `${prefix:?prefix is unset}` expansion makes the check
inline and fatal without strict mode. Better than both, because it also catches the
*set-but-wrong* case: test the composed path's existence and shape before acting on
it — a directory that should contain cache files can be required to exist and to
match a pattern you assert (`[ -d "$prefix/cache" ] && [[ "$prefix" == /srv/* ]]`)
before any destructive verb sees it. That is proof-of-target, and a later section
makes it doctrine.

## When filenames attack

The third rewrite, globbing, has two failure shapes of its own. The first: a glob
that matches nothing *passes itself through as a literal word* by default —
`*.conf` in an empty directory becomes the string `*.conf`, handed to your command
as if it were a filename, and whether that is harmless depends entirely on the
command. The second is nastier — filesystem content becomes command syntax:

```bash
cd "$(mktemp -d)"
touch -- -l data.log
echo "ls * sees:"
ls * 2>&1 | head -n 1
echo "ls -- * sees:"
ls -- * 2>&1
```

```output
ls * sees:
-rw-r--r-- 1 luis luis 0 Aug 27 22:07 data.log
ls -- * sees:
-l
data.log
```

A file named `-l` exists in the directory. The glob expands it along with everything
else, `ls` receives it *as an option*, and the first invocation silently becomes
`ls -l data.log` — a long listing of the other file, with the dash-file itself
vanished from the report. Any tool accepting options is vulnerable, and with `rm` or
`chmod` the smuggled option can be `-r` or `--no-preserve-root`. Two idioms close
the hole completely, and the listing shows the first: `--`, the near-universal
end-of-options marker, after which everything is an operand. The second is prefixing
relative globs with `./` (`rm ./*` — a name cannot begin with a dash if it begins
with `./`). One of the two belongs in every command whose operands come from a glob
or a variable; which one is taste, their absence is exposure.

Silent semantic reversal has one more famous residence: tools whose *argument order*
is meaning, where a plausible reordering runs and does something categorically
different. The specimen is `find`, whose command line is not options-then-operands
but a little program evaluated left to right:

```bash
cd "$(mktemp -d)"
touch a.tmp b.txt
echo "misordered:"
find . -type f -print -name "*.tmp"
echo "correct:"
find . -type f -name "*.tmp" -print
```

```output
misordered:
./b.txt
./a.tmp
correct:
./a.tmp
```

With `-print` *before* the name filter, the action fires for every file and the
filter, evaluated after, filters nothing. Substitute `-delete` for `-print` — a
substitution operators make exactly once — and the misordering deletes every file
under the tree instead of the `.tmp` files; same shape, and `-delete` even disables
some of `find`'s own safety refusals. The register's habit: any `find` that will
carry a destructive action is composed first with `-print` in the action's position,
dispatched, and its output *read as the plan* — then, and only then, re-dispatched
with the action swapped in. Which generalizes into the chapter's central move.

## Rehearsal: the dry run

Interactive operators sometimes rehearse; the register *must*, because rehearsal is
the only place its mistakes can be caught before they are permanent. The instruments
are the dry-run modes the serious tools all carry, and the doctrine is to treat them
not as reassurance but as the source of the plan you then execute. The worked
specimen is `rsync`, whose `--delete` — essential for true synchronization, feared
for good reasons — is exactly the kind of verb that deserves a rehearsal:

```bash
cd "$(mktemp -d)"
mkdir -p src dst
printf "1\n" > src/a.txt
printf "2\n" > dst/stale.txt
rsync -rn -v --delete src/ dst/ 2>&1 | head -n 6
echo "dst after the dry run:"
ls dst
```

```output
sending incremental file list
deleting stale.txt
a.txt

sent 66 bytes  received 32 bytes  196.00 bytes/sec
total size is 2  speedup is 0.02 (DRY RUN)
stale.txt
```

The `-n` run names its victims — `deleting stale.txt` — and the final `ls` proves the
victim still breathes: the rehearsal was pure prophecy, zero action. The transcript
now contains the plan, reviewable by the operator or its supervisor, and the
execution step is the identical command minus one letter. Note also what the
rehearsal quietly validated: rsync's infamous trailing-slash semantics (`src/` means
*contents of* src; `src` means *the directory itself* — one character, two different
resulting trees). The dry run renders that decision visible before it is real, which
is the general principle: **a dry run converts semantics you believe into semantics
you have read.** The register's rehearsal shelf is well stocked — `patch --dry-run`
(chapter 5), `apt-get -s` and its cousins for package operations, `systemd-analyze`
verbs for units and calendars (chapter 4), `--check` flags across configuration
tools — and where a tool has none, the `find -print`-then-swap pattern builds one.
The habit has a cost — one extra turn per destructive operation — and chapter 1's
economics prices it correctly: the turn is the cheapest thing this register has, and
it buys down the one cost that cannot be refunded.

## Proof of target, narrowness, and the verb that shows its work

Rehearsal checks the plan; proof-of-target checks the world. Before a destructive
verb runs, the transcript should already contain evidence that its operand is the
thing intended — existence, type, and a property no wrong target would share:

```bash
cd "$(mktemp -d)"
mkdir -p build
printf "artifact\n" > build/out.bin
[ -d build ] && [ -e build/out.bin ] \
  && rm -v build/out.bin \
  && [ ! -e build/out.bin ] && echo "removal verified"
```

```output
removed 'build/out.bin'
removal verified
```

Four small assertions braid chapter 2's ask-and-verify into the destructive case:
the target's existence proven before, the verb's own verbose narration (`-v` — the
register always asks destructive tools to narrate) during, and the absence proven
after. Alongside proof stands *narrowness*: the destructive verb receives the most
specific operand the task permits. Full explicit paths over `cd`-then-relative
(the `cd` that fails while the `rm` proceeds is a classic compound accident — and
the reason chapter 2 recommended strict mode's abort-on-error for scripts);
one named target over a glob where one target is meant; a glob over a recursive
flag where a glob suffices; `-maxdepth` on any `find` that does not intend the
abyss. Blast radius is measured at composition time by a simple question: *if every
name in this command resolved to the worst plausible value, what is the largest
thing that could disappear?* Narrowness is the practice of making that answer
small enough to survive.

## The reversibility ladder

The final recalibration is choosing verbs by their undo channel. Interactive humans
grade operations by effort; the register grades them by *recoverability*, and the
grades are a ladder worth internalizing. At the top, operations that carry their own
undo: `mv` within a filesystem (rename back), the drop-in file (delete it), the
atomic swap staged beside a kept original (swap back). In the middle, operations
recoverable with preparation: overwrites preceded by a copy (`cp -a target
target.bak.$(date -u +%Y%m%dT%H%M%SZ)` — timestamped, so repeated preparations do
not overwrite each other's insurance), deletions rehearsed and logged so that at
minimum *what was lost* is known. At the bottom, the truly one-way verbs: `rm`,
`>` truncation, `rsync --delete` unrehearsed, `dd` onto a device, database drops —
gone is gone, on a timescale that matters to the incident.

The ladder's use is substitution pressure: before dispatching a bottom-rung verb,
ask which higher rung reaches the same goal. The strongest everyday substitution is
quarantine — `mv` the doomed thing into a dated graveyard directory instead of
removing it. The operation is total (the directory is clean, the goal achieved),
reversible for as long as the graveyard is retained, and *cheap* — a rename costs
nothing regardless of size. Deletion becomes a scheduled, boring event (a timer
purging graveyards older than some retention), decoupled from the operational moment
where mistakes live. An operator that quarantines by default and deletes on a
schedule has converted its worst accident class into a recoverable inconvenience —
which is the entire ambition of this chapter, applied structurally rather than shot
by shot.

## The verbs that leave the machine

The reversibility ladder has a floor below its bottom rung, and it is not on the
filesystem. Commands that *communicate* — send the email, post to the API, push
the release, publish the message to the queue — are irreversible in a way even
`rm` is not: deletion destroys state you held, while communication creates state
in systems you do not hold. There is no quarantine directory for a sent
notification; the copy that matters now lives in someone else's inbox, cache, or
audit log, beyond every undo channel this chapter built. The register treats such
verbs as a class of their own, with three rules. They are never composed into
retry loops or strict-mode chains casually — a "retry on failure" wrapped around a
send can deliver twice, and chapter 6's read-before-retry rule applies with the
reading done on the *far* system where possible (did the message arrive? does the
API's idempotency key say this request was already seen?). Idempotency keys, where
the far side offers them, are the outward world's version of the guarded append,
and using them is not optional politeness but the only mechanism that makes an
outward retry safe at all. And the rehearsal principle inverts into staging:
where a dry run cannot exist (few mail systems offer one), the rehearsal is a
*real* send against a target you own — the test inbox, the sandbox API, the
staging channel — with the production dispatch composed as the same command with
one variable changed, and that variable proven, proof-of-target style, before the
shot goes out. An operator that would rehearse a local `rsync --delete` three
times and then improvise a production API call has ranked its risks by
familiarity rather than by blast radius; this section exists to reverse that
ranking.

## Working where others work

The blast radius of a command includes operators it collides with. A one-shot
operator is rarely alone on a machine: humans hold sessions, timers fire (chapter
4's real machine runs several), other agents may be mid-task in the same trees.
Two consequences follow, one about your writes, one about everyone else's.

Your writes need private ground. Predictable scratch paths — `/tmp/work`,
`/tmp/out.txt` — are collisions waiting (two operators, one path, interleaved
writes) and, on shared machines, a classic attack surface: a hostile process that
pre-creates the predictable name as a symlink redirects your write onto any file
your identity can damage. The system's answer is `mktemp`, which this book's
listings have used since chapter 1 and which earns its formal introduction here:

```bash
w=$(mktemp -d)
echo "workspace: $w"
[ -d "$w" ] && echo "exists, mode $(stat -c %a "$w")"
```

```output
workspace: /tmp/tmp.NwLRJZiNbQ
exists, mode 700
```

Unpredictable name, created atomically, mode `700` — private by construction. The
habit: every scratch need goes through `mktemp`; the only names you write outside
scratch are the deliberate, guarded targets of the task itself.

Against everyone else's writes, the instrument is mutual exclusion, and the shell
has a real one — `flock(1)`, an advisory lock on a file descriptor, atomic in the
kernel, released automatically when the holding process exits (an important property
for an operator whose process *will* exit, cleanly or not — no crash leaves a stale
lock held):

```bash
cd "$(mktemp -d)"
exec 9> job.lock
if flock -n 9; then echo "lock acquired"; else echo "another operator holds the lock"; fi
( exec 8> job.lock
  flock -n 8 && echo "second acquisition: succeeded" || echo "second acquisition: refused" )
```

```output
lock acquired
second acquisition: refused
```

The subshell plays the second operator, and the kernel refuses it while the first
descriptor lives. `-n` makes the attempt non-blocking — the register's correct
default, since a shot that queues invisibly behind a lock is a chapter 1 hang with
extra steps; better to learn *refused* in one turn and decide, than to wait
silently. Any procedure that must not run twice concurrently — a deploy, a
migration, a graveyard purge — opens with this pattern, and the lock file's name
becomes part of the procedure's documented interface, which is precisely how the
system cron daemons and package managers already guard themselves (the familiar
"could not get lock" from a package manager is this same mechanism, experienced
from the outside).

## The retry, and what must be read before it

Chapter 5 built idempotence into edits; this chapter's version of the question is
harsher: a destructive or compound shot *failed midway, or timed out with its fate
unknown* — what now? The interactive reflex, run-it-again, is exactly wrong as a
reflex, because a compound operation that died mid-flight has left the world in a
state that neither the before nor the after picture describes, and the retry was
composed against the before picture. The register's rule: **after a failed write,
the next shot is a read.** Reconstruct which stage completed from evidence — which
files exist, which staged artifacts are present, what the service's timestamps say
— then resume from the stage the evidence proves, not from the top:

```bash
cd "$(mktemp -d)"
mkdir -p deploy
printf "v2\n" > deploy/app.txt.new
ls deploy
echo "staged file present: retry resumes at validate-and-swap, not at generation"
```

```output
app.txt.new
staged file present: retry resumes at validate-and-swap, not at generation
```

A toy, but the shape is the real discipline: chapter 5's atomic-swap pipeline,
interrupted, is *legible* — the presence of the `.new` file tells the returning
operator exactly where death occurred, and the resume point follows from the
evidence rather than from hope. That legibility was purchased at composition time,
by building the procedure from stages whose completion leaves distinct marks. The
general design rule for anything long: make each stage's completion *observable*
(a staged file, a moved marker, a logged line), so that the mid-flight state reads
as a position rather than as wreckage — chapter 8 formalizes the same idea as the
change ledger. And when a timed-out shot's fate is genuinely unknowable — the
classic being a remote command whose connection died after dispatch — the read
comes *first* and decides whether there is anything to retry at all: the operation
may have completed beautifully, in your absence, like everything else in this
register does.

## Privilege: the discipline of remaining small

Everything above bounds what a command *does*; privilege bounds what it *can* do,
and the register's rule is inherited from decades of automation practice with the
stakes newly raised: **operate at the lowest privilege that answers the question,
and escalate per-command, not per-session.** Chapters 3 and 4 were written
deliberately from an unprivileged seat — processes read, services diagnosed, a
failed unit triaged to its exit status, all without root; the one boundary hit (the
journal's group wall) was itself diagnosable from below. That is representative:
reading rarely needs privilege, and the shots that do escalate should each carry
`sudo` visibly — the transcript then shows exactly which actions ran elevated,
an audit trail chapter 8 will want, against a `sudo -i` session in which every
subsequent accident, expansion included, is a root accident. `sudo -l` reads the
current identity's actual grants — worth one shot on any new machine, since it is
the authoritative answer to *what can this seat do*, and (chapter 4's lesson
recurring) `sudo` strips most of the environment, so a command that behaves
differently under it has usually lost a variable, not gained a bug. For the
supervising reader designing the seat itself: the strongest blast-radius decision
is made before any command runs, in what the operator's account can reach at all —
filesystem permissions, group memberships, a sudoers file listing specific commands
rather than `ALL`. The guards in this chapter are how a careful operator behaves;
the account is how careful the machine remains when, someday, a shot is composed
without them.

Quoted expansions, guarded emptiness, disarmed filenames, rehearsed deletions,
proven targets, reversible verbs, and a small seat: none of it is exotic, and all
of it is composition-time work, because composition time is all this register has.
What safety cannot be compressed into is a checklist recited once — it is a set of
defaults, and defaults are cheap. The next chapter takes the same doctrine onto the
one substrate where even reads have side effects and nothing stays still: the
network.
