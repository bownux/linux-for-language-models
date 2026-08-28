# Chapter 2 — One Shot, One Truth

*Draft status: author draft, gate-checked; human verification pending. Outputs shown
are real outputs from the authoring machine.*

## The number is the message

Every command you will ever run ends by handing the kernel a small integer, and that
integer is the most reliable sentence Linux will ever speak to you. The exit status is
not decoration on the output; it is the output's verdict. Text can be translated,
reworded between tool versions, reimplemented in another language with different
phrasing — chapter 1 closed on exactly such a seam, two implementations of `ls`
describing one missing file in two different sentences with one identical status. The
number's meaning, by contrast, is contract: zero is success, nonzero is failure, and
the shell's own manual pins the semantics. An operator who reads transcripts for a
living learns to ask the number first and treat the prose as commentary.

The contract has structure worth knowing precisely, because the structure carries
diagnosis. Statuses up to 125 belong to the program itself, and the best tools spend
them meaningfully. `grep` is the canonical example — a trichotomy, not a boolean:

```bash
printf "alpha\nbeta\n" | grep -q alpha;         echo "selected:   $?"
printf "alpha\nbeta\n" | grep -q missing;       echo "no match:   $?"
grep -q pattern /no/such/file 2>/dev/null;      echo "error:      $?"
```

```output
selected:   0
no match:   1
error:      2
```

Status 1 from `grep` is not an error. It is a successful search whose answer was *no*
— information, and often the information you wanted, as when you verify that a broken
setting is gone from a config file. Status 2 is the actual failure: the search could
not be conducted. Collapsing those two into "grep failed" is one of the register's
classic self-inflicted wounds, and it matters doubly under the strict-mode flags
discussed below, where an innocent "no" can abort a whole script if you have not
decided in advance which answer you expect.

Above the program's own range, the shell reserves statuses to report on programs it
could not run: 126 when the file exists but is not executable, 127 when the command
was not found at all. In transcript mode, 127 deserves reflex status — it means your
question never reached a tool, so the transcript's text (if any) describes a shell
problem, not a system problem. Beyond those, a command killed by a signal reports 128
plus the signal number: 137 is a SIGKILL (nine), very often the out-of-memory killer's
signature; 141 is a SIGPIPE (thirteen), which this chapter will produce on purpose in
a moment; and `timeout(1)` reports 124 for a command it had to cut off:

```bash
timeout 1 sleep 5
echo "status: $?"
```

```output
status: 124
```

That listing is also the first safety tool of the book. Chapter 1 catalogued the traps
that hang a shot forever — pagers, prompts, stdin-blockers. `timeout` converts all of
them from *shot that never returns* into *status 124 after a bound you chose*, and in
an environment where a hung command costs a whole turn plus a harness timeout you did
not choose, wrapping anything remotely doubtful in `timeout` is not paranoia but
budgeting. The habit generalizes: a well-composed shot has a known worst case — in
time (`timeout`), in volume (`head`, below), and in consequence (chapter 6) — before
it is dispatched.

One more property of the number completes the contract: in a pipeline, there are
several numbers, and by default the shell hands you only the last. `false | true` is a
success by default. The `PIPESTATUS` array and the `pipefail` option (below) exist to
recover the rest. Keep that in mind through the next section, because the two streams
and the several statuses interact.

## Two streams, two audiences

A process is born holding three file descriptors, and the register's second discipline
is to respect the difference between the two it writes: standard output is for the
*answer*; standard error is for *commentary about the attempt* — progress, warnings,
complaints. The convention is old, near-universal, and precisely what makes one-shot
composition possible: because `df`'s answer and `df`'s complaints travel different
pipes, an `awk` downstream parses the answer without ever seeing the complaints.

In transcript mode you are usually handed both streams, but *how* they arrive is your
choice, and the choice is worth making deliberately. Merged (`2>&1`), you get a single
chronological story — right for debugging, where the complaint's position among the
output lines is itself evidence. Separated (`2>errors.txt`, or captured independently
by your harness), you get a parseable answer channel and a quarantined commentary
channel — right for composition, where a warning printed mid-table must not reach your
parser. What you must never do is leave the merge to habit, because the merge is the
number-one source of parsers eating prose. A tool that got more talkative in a new
version — a deprecation warning, a TLS notice — breaks a merged-stream parser at a
random future date, through no change of yours.

The merge syntax carries a famous ordering subtlety that a one-shot operator has no
interactive opportunity to debug, so learn it once, here. Redirections apply left to
right: `cmd > file 2>&1` first points stdout at the file, then points stderr at
"wherever stdout points now" — both land in the file. Reversed, `cmd 2>&1 > file`
points stderr at "wherever stdout points *now*" — the terminal or capture pipe — and
only then moves stdout to the file: the streams end up split, the file missing the
commentary. Both spellings look plausible; only one says what you probably meant.
When you want everything a command emitted, in order, in one place, the idiom is
`cmd > out.txt 2>&1`, and no other arrangement of those tokens is its synonym.

The two-audience rule also governs your own emissions. When your shot is itself a
small script — an `awk` program, a loop — put the answer on stdout and route your own
diagnostics to stderr (`echo "warning" >&2`), because the next operator to build on
your shot will parse it exactly as you parse `df`. In this register you are not only a
consumer of the convention; you are a link in it.

## Determinism: the same shot must mean the same thing

An interactive human re-runs a flaky command and shrugs. A transcript-mode operator
comparing today's output to yesterday's needs the differences to be *signal*, and that
requires stripping the environment's fingerprints from the output. Three fingerprints
account for most of the noise.

The first is locale. A surprising amount of "what did the command say" is
locale-dependent: sort orders, decimal separators, month names, even which column a
tool aligns. The classic demonstration is `sort`, whose ordering under a language
locale interleaves cases and can differ between systems, but under `LC_ALL=C` is the
one ordering every machine on earth agrees on — raw byte order:

```bash
printf "banana\nApple\ncherry\n" | LC_ALL=C sort
```

```output
Apple
banana
cherry
```

Uppercase letters sort before lowercase in byte order, so `Apple` leads — a result
some language locales would reverse. Neither ordering is wrong; the point is that only
one of them is *pinned*. The GNU sort documentation itself warns that locale collation
produces surprising results and recommends `LC_ALL=C` when byte-stable ordering is
wanted. The register's rule: any shot whose output you will parse, diff, or join
against another shot's output gets `LC_ALL=C` — usually as a prefix on the one command
that needs it, so the pin is visible in the transcript rather than hidden in
environment setup you would have to remember happened.

The second fingerprint is time. `date` with no arguments answers in local time with a
localized format — pleasant on a screen, poison in a ledger, because "today" formatted
in one machine's timezone does not join against another's. The register writes
timestamps in UTC, in ISO-8601, always: `date -u +%Y-%m-%dT%H:%M:%SZ`. The two extra
flags cost nothing at composition time and save an entire class of
off-by-one-timezone confusions at reading time, which for you is the only time there
is.

The third fingerprint is the audience fork itself, and here the craft is to prefer
formats that are *documented as stable* over formats that merely look parseable.
The ecosystem's clearest naming of this idea is git's: `git status` is a human
display, explicitly subject to change between versions, while `git status
--porcelain` is a wire format the documentation promises to keep stable for scripts.
Many tools have such a mode under many spellings — `--porcelain`, `-P` on `df`
(POSIX-pinned columns, used in chapter 1's worked shot), `--json` on a growing set of
system tools (chapter 3 makes heavy use of these). The general rule: when a tool
offers a machine format, the machine format is yours. The human display was never a
contract, and parsing it means your shot's meaning can be changed by someone else's
cosmetic commit.

## Bounding the shot

Chapter 1 named output volume as the register's second cost; here is the mechanics of
paying it. The blunt instruments are `head` and `tail`, and the habit of *always*
capping any command whose output size you cannot predict: an unfamiliar log, a
recursive listing, a `find` over a tree of unknown depth. A cap is not merely about
politeness to your own attention — an unbounded dump can push the fact you needed out
of a truncated capture buffer, so the cap is what guarantees the *relevant* part
arrives. The refined instruments are the tools' own bounds — `grep -m 1` stops at the
first match rather than scanning to the end; `journalctl -n 50 --since` bounds by
count and time at the source; `find -maxdepth` refuses the abyss before descending
into it. Prefer the source-side bound where it exists: `head` discards output after it
was produced, while `-m`, `-n`, and `--since` prevent the work itself.

Capping a pipeline, though, springs one of the register's best-hidden traps, and you
should meet it on your own terms rather than in production. When `head` has taken its
fill it exits, the pipe closes, and the producer still writing into that pipe is
killed by SIGPIPE — which, per the 128-plus-signal rule, is status 141:

```bash
set -o pipefail
seq 1000000 | head -n 1
echo "pipeline status: $?"
```

```output
1
pipeline status: 141
```

The answer — the first line — arrived perfectly. The pipeline's status says a
component died of signal 13, because under `pipefail` the pipeline reports any
component's failure, and `seq`, mid-write into a closed pipe, was in fact killed.
Nothing malfunctioned; producer-dies-when-consumer-leaves is exactly how pipe
plumbing is meant to economize. But an operator running under strict mode (next
section) will see the shot *fail* — and a script will abort — on a pipeline that did
its job. The escapes, in order of preference: bound at the source instead of piping
to `head` (`sed 1q`, `grep -m`, `-n` flags) so no producer is left writing; or accept
and inspect the status knowing 141-with-good-output is benign in this specific shape;
or drop `pipefail` for that one pipeline. What you may not do is let the first
surprise 141 teach you to stop using caps or to stop reading statuses — both lessons
would be exactly backward.

## Strict mode, and its fine print

The preamble `set -euo pipefail` appears at the top of most modern shell scripts, and
you should know both why it earned that position and where its promises end, because
the register leans on it harder than interactive use ever did. `-e` (errexit) aborts
the script when a command fails un-checked; `-u` (nounset) makes expansion of an
unset variable an error instead of a silent empty string; `pipefail` you have just
met. Together they convert a script from "keeps going regardless, damage compounding"
to "stops at the first surprise" — and in a mode with no human watching the damage
compound, stopping early is the correct default. `-u` in particular defuses one of
the most catastrophic accident shapes in all of shell: `rm -rf "$prefix/cache"` with
`prefix` unset is, without `-u`, a cheerful attempt to delete `/cache`; with `-u`, it
is an aborted script and an error message. Chapter 6 dissects that accident class in
detail; strict mode is its first line of defense.

The fine print is that `-e` is a blunt instrument with documented dull spots, and the
register's operators must know them rather than trust the flag as a talisman. A
command's failure does not trigger errexit when the command sits in a tested position
— the condition of an `if`, the left side of `&&` or `||` — which is usually what you
want (that grep status 1 stays usable) but means a *misspelled* command in those
positions also sails on. Failures inside command substitution in an assignment can
escape notice entirely:

```bash no-run
set -e
result=$(false; echo "kept going")
echo "after: $result"
```

```output
after: kept going
```

The `false` failed; the substitution's status is that of its *last* command, the
`echo`; the assignment succeeded; strict mode saw nothing. The craft consequences are
two. Inside scripts, check the statuses you actually care about explicitly — `x=$(cmd)
|| exit 1`, or test the result's shape (`[ -n "$x" ]`) rather than assuming errexit
guarded the assignment. And in single composed shots — one pipeline, no state — skip
the incantation and read `$?` yourself; strict mode is a script's discipline, and a
one-liner wears it mostly as costume. `-u`, by contrast, has no such dull spots and
belongs everywhere; its measured failure mode on the authoring machine is loud and
immediate:

```bash no-run
bash -c 'set -u; echo "$not_defined"' 2>&1
echo "child status: $?"
```

```output
bash: line 1: not_defined: unbound variable
child status: 127
```

(The precise nonzero number varies with how the shell was invoked; the contract you
rely on is *nonzero, before the expansion is used* — the difference between an aborted
shot and a deleted `/cache`.)

## Ask and verify in the same shot

The single most compounding habit in the register is this: a shot that changes the
machine carries its own check, and prints the check's result as its final output. Not
because your tools are especially untrustworthy, but because in transcript mode the
alternative is to *assume* — and chapter 1's finality cost means you will not be
present to notice a wrong assumption until something built on it fails. The pattern
at its smallest:

```bash
cd "$(mktemp -d)"
printf "retries = 5\n" > service.conf
grep -c "^retries = 5$" service.conf
```

```output
1
```

The write happened; the read-back proves it; the `1` is the proof, in the transcript,
where it now exists as evidence rather than as hope. The pattern scales up through
`&&` chains — `mkdir -p target && test -d target && echo "target ready"` — and, for
anything with a service on the other end, through a functional probe rather than a
structural one: after changing a config, the verifying read is not "is my line in the
file" but "does the service now answer the way the change intended" (chapters 4 and 7
build those probes). The register's phrasing of the principle: **a change without a
printed verification is, to every future reader of the transcript including you, a
rumor.** Chapter 8 grows this habit into the evidence-block convention that closes a
whole piece of work; it starts here, one `grep -c` at a time.

Verification composed into the shot also changes *failure* into information. When the
`&&` chain stops early, the transcript shows exactly which link broke — the mkdir, the
test, the probe — with no additional forensic turn spent. In a mode that pays per
round trip, a shot that localizes its own failure is not a nicety; it is the
difference between one turn and four.

## Disarming the environment

Last, the preamble that makes the rest possible. Chapter 1's trap families — pagers,
prompts, editors — are disarmed partly per-command (`--no-pager`, `-y`,
`--batch`) and partly, more durably, through the environment variables the tools
consult before deciding how to behave. A transcript-mode operator's session
environment should say, in every dialect the common tools understand, *no one is
watching; do not wait for anyone*:

```bash fragment
# The non-interactive preamble: set once per session, not per shot.
export PAGER=cat GIT_PAGER=cat SYSTEMD_PAGER=cat   # pagers: pass text through
export GIT_EDITOR=false                            # editors: fail fast instead of hanging
export DEBIAN_FRONTEND=noninteractive              # Debian-family installers: never prompt
export LC_ALL=C.UTF-8 TZ=UTC                       # pin collation, encoding, clock
```

The fragment marking is deliberate: this changes session state, and which variables
earn a place depends on the tools your machine actually runs — a systemd-less box
needs no `SYSTEMD_PAGER`; a Fedora box replaces the Debian line. The principle is
portable even where the spellings are not. Set the environment so that a *forgotten*
per-command flag degrades into safety (a pager that harmlessly cats, an editor that
fails instantly and visibly) rather than into a hang. Defense in both layers — the
environment as the net, explicit flags as the practice — because in this register a
hang and a catastrophe are nearer neighbors than they ever are at a terminal: both
end the turn with the machine's state unknown to you.

## The batch: several questions, one dispatch

The chapter has treated the shot as one command, but the round-trip economics of
chapter 1 point at a composition pattern this book's later chapters use constantly:
the *batch* — several independent reads dispatched as one shot, their answers
labeled so the transcript stays parseable. The shell's `;` separator is the whole
mechanism; the craft is in the labeling and the independence. Labeled, because six
commands' outputs concatenated without markers force the reader to guess where one
answer ends — so each section opens with an `echo` naming what follows, or each
line carries its own prefix (the introduction shot in chapter 3 and the layer
sweep in chapter 7 are both this pattern in the field). Independent, because `;`
runs every command regardless of predecessors' failures — which is precisely right
for a diagnostic sweep, where the third read failing must not cost you the
remaining four, and precisely wrong for a sequence with dependencies, which is
what `&&` is for. The choice between the two separators is therefore a statement
of intent: `&&` says *these stand or fall together*; `;` says *these are separate
questions sharing a stamp*. Mixing them by habit rather than intent produces the
two corresponding accidents — the sweep that silently stops reading after one
failure, and the dependent chain that barrels on past a failed precondition
(chapter 6 has opinions about the second). A last sizing rule keeps batches
honest: batch *reads* freely, but a shot should carry at most one *write*, so
that any failure in the transcript maps to at most one change to reason about —
the finality cost, budgeted one commitment at a time.

## Reading the transcript back

Composition is half the craft of the single shot; the other half is reading what came
back, and reading it in the right order. Operators new to the register read the way
humans read a screen — prose first, top to bottom, forming an impression. Operators
who have been burned read like this: status, then stderr, then the *shape* of stdout,
and only then its content.

Status first, because it reframes everything after it. The same stdout means different
things under status 0 and status 2 — a filtered list that arrives alongside status 2
is a *partial* list, produced before the failure, and treating it as complete is a
quiet corruption of everything downstream. Stderr second, because commentary explains
verdicts: a status 1 with `Permission denied` on stderr is a different investigation
from a status 1 with silence. Shape third — line count, field count, the presence of
the header you expected — because shape mismatches catch the wrong-question errors
that content reading misses: the command succeeded, the output parses, and it answers
a question adjacent to the one you meant to ask. A `grep` that returns nothing and
status 1 has answered *no matches*; but if you meant to search a different file, the
answer is truthful and useless, and only checking the shape of the invocation against
your intent catches it.

Empty output deserves its own paragraph, because in this register emptiness is the
most ambiguous sentence a transcript can contain. An empty result with status 0 from a
filter usually means *ran, found nothing* — but it can also mean the input was empty,
which is a different fact entirely. When the distinction matters, split it explicitly
in the composition: count the input and the matches separately, so the transcript
distinguishes "no hot filesystems among the twelve examined" from "zero filesystems
examined" — the second being a broken shot wearing a calm face. Chapter 1's worked
`df` shot printed an affirmative sentence for its empty case for exactly this reason.
The rule generalizes into one of the register's small signatures: **good shots say
"none", never just nothing.**

Numbers in a transcript deserve one final habit: distrust of unanchored plausibility.
An interactive human who typos `df` into reporting the wrong mount notices, because
the screen sits inside a context of intent. A transcript number — `86`, `262510` —
carries no such context unless the shot printed it: units, the identifier of the thing
measured, the threshold it was judged against. That is why the chapter 1 shot printed
`/mnt/train 98% used, 25828 MiB free` rather than `98`. Label everything at
composition time; at reading time, treat any bare number whose unit or subject you
cannot point to in the same transcript as unverified. This is cheap when the shot is
written and impossible to retrofit when the transcript is all that remains.

The four-question routine — *what was the status? what did stderr say? does the shape
match the question? does the content, labeled, answer it?* — takes seconds and is the
register's substitute for the peripheral vision a terminal gave for free. It also
composes forward: shots written by an operator who reads this way start carrying their
statuses, labels, and affirmative negatives on purpose, because the writer and the
reader are the same operator on different turns, and the writer learns to serve the
reader.

With the command's anatomy in hand — status first, streams separated, output pinned
and bounded, changes self-verifying, environment disarmed — the next question is what
to point it at. A machine's state is not a screenful of dashboards; it is a filesystem
of numbers that were always meant to be read one shot at a time. That reading is
chapter 3.
