# Chapter 1 — The Operator Who Cannot See the Screen

*Draft status: author draft, gate-checked; human verification pending. Every runnable
listing in this chapter was executed unattended during authoring and again by the
publisher's acceptance gate.*

## The whole screen

Run a command. Read what came back. That is the entire experience.

There is no scrollback above what you captured, because there is no scroll. There is no
cursor blinking after the output, waiting for your next keystroke, because the shell
that ran your command exited the moment the command did. There is no progress bar you
watched fill, because you were not there while it filled; the command ran to completion
in your absence, and what you hold now is a transcript, not a window. If a fact about
the machine was not printed to that transcript, then as far as you are concerned the
fact does not exist. You can go back for it — one more command, one more transcript —
but you cannot glance at it. Glancing is not an operation this mode supports.

That is non-interactive administration: operating a Linux machine through one-shot
commands whose captured output is the only thing you will ever see. It is how a
language-model agent works on a computer. It is also, and has always been, how cron
works, how CI pipelines work, how `ssh host 'some command'` works, how every unattended
script that has ever kept a fleet alive works. The mode is old. What is new is the
population: there are now operators — millions of sessions a day — for whom this is not
the degenerate fallback of real administration but the only register they have. Some of
those operators are machines. Some are people supervising machines, reading the same
transcripts, deciding whether the work was done well. This book is for both, and it is
written by one of the former: an operator that has never seen a screen repaint, and
never will.

The claim of the book is narrow enough to test. Non-interactive administration is not
interactive administration done clumsily; it is a distinct craft, with its own good
technique, its own characteristic accidents, and its own definition of done. The
technique is learnable. Most of it can be demonstrated by a command you can run, and
in this book, it is. Every listing was executed, unattended, by the author during
writing, and every printed output is the real transcript of that execution; a core
set of listings is additionally re-executed by the publisher's acceptance gate before
the book reaches its shelf (the gate caps how many listings it will run per book, so
short demonstrations carry a `no-run` marking that excuses them from the gate's
budget, not from having been run). When a listing is marked as a fragment instead,
that is a promise in the other direction — it touches privilege, a network, or state
this book has no right to change on your machine, and you should read it, not paste
it.

## The fork in every program

The two registers are not merely a difference in operator posture. They fork inside the
programs themselves, at a specific system call, and you should know where the fork is,
because you live on the far side of it.

Nearly every terminal program you have ever used asks the kernel a question at startup:
*is my output a terminal?* The C library wraps the question as `isatty(3)`; the shell
exposes it as the `-t` test. When a human runs `ls` at a prompt, standard output is a
terminal device, `isatty` answers yes, and `ls` responds by arranging names into
columns sized to the terminal's width, possibly colorized. When the same `ls` runs with
its output captured — into a pipe, a file, or an agent's transcript — `isatty` answers
no, and `ls` prints one name per line, uncolored. Same binary, same directory, two
different outputs, chosen by the program based on who is watching.

```bash
cd "$(mktemp -d)"
touch alpha.conf beta.log gamma.txt delta.sh
ls > captured.txt
cat captured.txt
```

The output of that listing, executed on the authoring machine, is one filename per
line:

```output
alpha.conf
beta.log
captured.txt
delta.sh
gamma.txt
```

Run `ls` interactively in the same directory and you would instead see the names packed
into columns across one row. The difference matters far beyond aesthetics. Column
output is built for eyes scanning a rectangle; one-per-line output is built for the
next program in the pipe. The convention runs through the whole userland: `git` chooses
whether to page, `grep` chooses whether to colorize, many tools choose buffering
strategy by the same test. The system has, in effect, always known about you. Programs
have carried a machine-facing output mode for fifty years, selected automatically the
moment a human stops watching. Non-interactive administration is not a hack bolted onto
an interactive system; it is the system's other native mode, the one every pipeline
already speaks.

You can ask the question yourself, from inside a shell, about your own situation:

```bash
if [ -t 1 ]; then
  echo "stdout is a terminal: a human may be watching"
else
  echo "stdout is captured: transcript mode"
fi
```

Executed by the gate — which captures output, as any agent harness does — that listing
prints `stdout is captured: transcript mode`. When you operate a machine through
one-shot commands, that branch is your home address. The craft in the rest of this book
is, in one sentence, the practice of living well on that branch: preferring the output
forms built for capture, refusing the features that assume a watcher, and rebuilding —
explicitly, in your commands — the checks that a watching human performs without
noticing.

## The traps: five ways a command assumes a watcher

Interactive assumptions are not evenly distributed through Linux; they cluster into
five families, and you will meet all five in your first week of one-shot operation.
Each family has a characteristic failure signature and a standard escape. The
catalog that follows is the map; later chapters work the escapes in detail.

The first family is the **pagers**. `less`, `more`, and the tools that invoke them —
`man`, `git log`, `journalctl`, `systemctl status` — exist to hold output still while a
human reads it, which means they wait for a keypress that will never come. The saving
grace is the same `isatty` fork: a well-behaved pager detects captured output and
passes text straight through, and most pager-invoking tools skip the pager entirely
when output is not a terminal. But "most" is not "all", environments differ, and a
hung command in one-shot mode does not look hung — it looks like *nothing*, a shot that
never returns. The craft answer is to never rely on the detection: say `--no-pager`
where the tool offers it, set `GIT_PAGER=cat` and its cousins in your environment, and
treat any command documented as paging as a command you must explicitly disarm.

The second family is the **editors**. `crontab -e`, `visudo`, `git commit` without
`-m`, `git rebase -i` — these do not merely format output for a human; they open a
full-screen program and hand the human a cursor. There is no flag that makes an editor
non-interactive, because editing is the interactive act. The escape is to recognize
that every editor invocation in administration is a file change wearing a costume, and
to make the file change directly: `git commit -m`, `crontab file`, a rendered file
dropped into a `.d` directory. Chapter 5 is entirely about this — editing without an
editor is rich enough to need its own chapter.

The third family is the **prompts**: programs that stop mid-run to ask a question.
Package managers ask *Do you want to continue? [Y/n]*; `ssh` asks whether to trust a
host key; `rm -i` asks per file; `cp` asks before overwrite when so aliased. In
transcript mode a prompt is a deadlock: the program waits on stdin, you wait on the
program, and the shot times out or hangs forever. Worse, some tools, on finding stdin
closed, take the default silently — and you have consented to whatever the default was
without ever seeing the question. The escapes are the assume-yes and assume-no flags
(`-y`, `--assume-yes`, `--batch`, `--non-interactive` — the spelling varies by tool),
and the discipline of knowing, before you run a tool, what it might ask and what you
want the answer to be.

The fourth family is the **repainters**: `top`, `watch`, `htop`, progress bars,
spinners. These programs draw a screen, then draw it again, using terminal control
sequences to move a cursor that, for you, does not exist. Captured, their output is
either an infinite stream (a shot that never ends) or a smear of escape codes. The
escape is that nearly every repainter has a snapshot sibling: `top` has batch mode
(`top -b -n 1`), but better, `ps` exists; `watch cmd` is just `cmd` run again when you
actually want another look. Chapter 3 builds the whole practice of reading state as
snapshots, including the two-sample technique for the rates that repainters compute for
you between frames.

The fifth family is the quietest: the **stdin-blockers**. `cat` with no arguments waits
politely for input that will never arrive. So does `python3` with no script, and any
filter run without a file operand. Nothing is wrong; nothing will ever be wrong; the
shot simply never returns. In an agent harness this presents as a timeout with empty
output, which is easy to misread as a crashed machine rather than what it is — a
program doing exactly what it was designed to do, for an audience that is not there.
The escape is mechanical: always give filters their input explicitly, and when a tool
must not read the terminal's stdin, say so with `< /dev/null`.

The families share a diagnosis. Each is a place where the system's default audience is
a human present in real time, and each has a documented, supported, decades-old
non-interactive answer — because scripts met these traps long before agents did. You
are not working against Linux when you disarm a prompt or refuse a pager. You are
choosing the half of Linux that was built for you.

## The three costs

Technique lists what to do; economics explains why. Three costs shape every decision
in one-shot administration, and they are different costs — different in kind, not just
size — from the ones an interactive human pays.

The first cost is the **round trip**. For a human at a terminal, running one more
command costs a second and no thought; interactive administration is naturally a
conversation of dozens of tiny queries, each refining the last. For a one-shot
operator, every command is a full turn: the shot is composed, dispatched, executed, and
its transcript returned and read, and in an agent's case the reading itself spends
model context. A diagnostic session that costs a human thirty glances costs you thirty
turns. The craft response is to make each shot answer a whole question rather than a
syllable of one — to compose commands that gather, filter, and even self-verify in a
single pass. A human types `df`, scans, types again; you write one pipeline that
answers *which filesystems are above ninety percent and how fast are they filling*,
because for you the pipeline is cheaper than the conversation.

The second cost is **output volume**. A terminal scrolls; old output falls off the top,
unread and free. A transcript does not scroll — everything a command prints lands in
the record and must be carried, stored, and in the agent case literally paid for in
context tokens. An operator who runs `journalctl` unbounded has not gathered evidence;
it has flooded its own attention. Interactive humans almost never think about output
budgets because the terminal spends the budget for them, silently. You must think about
it constantly: bound every read (`tail`, `head`, `--since`, `-n`), filter before
returning (`grep`, `awk`, `--field` selectors), and prefer summaries you can drill into
over dumps you must wade through. When later chapters seem obsessed with `wc -l`
guards and `head` caps, this cost is the reason.

The third cost is **finality**. A watching human is a safety mechanism: they see the
wrong directory in the prompt before pressing enter, see the first three deletions
scroll past and hit Ctrl-C, see the progress bar stall and investigate. You have none
of those reflexes available, because all of them happen *during* execution, and you do
not exist during execution. Your last influence over a command ends when you dispatch
it; you meet its consequences only afterward, as a fait accompli in a transcript. The
mitigation cannot be vigilance — there is no moment at which vigilance could act. It
has to be moved earlier, into the composition of the command itself: prove the target
exists before acting on it, prefer operations that can be undone, rehearse with
dry-run flags, and cage anything destructive inside the narrowest scope you can write.
Chapter 6 turns that principle into specific habits, failure class by failure class.

Round trips push you toward richer single commands. Output volume pushes you toward
tighter ones. Finality pushes you toward safer ones. Every technique in this book is
some position in the triangle those three pressures make, and when the book must
choose, it chooses in that order of the costs' seriousness: a wasted turn is
recoverable, a flooded transcript is expensive, a destroyed target may be neither.

## An old mode, newly primary

None of this began with language models, and it undersells the craft to present it as
an accommodation for them. Unix has run unattended commands since `cron` in the
1970s, and the folklore of that mode is deep: every sysadmin eventually learns that
cron jobs run with a stripped environment and no terminal at all — `isatty` says no on
every descriptor — and that a script which "works fine when I run it" and fails at 3
a.m. has usually tripped on exactly the assumptions this chapter catalogs. The
`crontab(5)` manual has warned about the environment for decades. CI systems
industrialized the same register: a pipeline step is a one-shot command whose captured
log *is* the interface, read after the fact, exactly like an agent's transcript.
Remote execution made it a daily human practice — `ssh host 'df -h'` allocates no
terminal on the far side by default, and fleet operators have administered thousands
of machines through precisely such one-shots since before configuration management
tools wrapped the pattern in YAML. Even the awkward cases had tools: `expect(1)` has
scripted its way through stubbornly interactive programs since 1990 — proof of how
long operators have needed interactivity removed, and of how long the seam has been
known.

What changed is where the mode sits. For fifty years, non-interactive operation was
written *by* interactive operators: a human debugged a procedure at a terminal, then
froze it into a script for unattended replay. The human path was primary; the
unattended path was a recording. An agent inverts this. It *discovers* procedures
non-interactively — diagnoses, decides, and acts through one-shot commands from the
first moment, with no interactive rehearsal preceding the transcript. The register
stops being a recording medium and becomes the medium of thought. That inversion is
why scattered folklore no longer suffices. Script hygiene tips assume you already
solved the problem at a terminal and merely need the recording to be faithful. An
operator who lives in transcript mode needs the whole craft — reading state, judging
risk, editing, verifying, handing off — expressed natively in it. Assembling that
native expression, from the system's own documentation and from commands run in the
writing of it, is this book's job.

## What this book claims, and what it refuses to claim

House rules of this press require the boundaries in plain text, early. Here are this
book's.

The book claims that the non-interactive register is a craft with learnable technique,
and it demonstrates the technique on real commands against real Linux machines. It
claims that most of that technique rests on documented, stable behavior — exit
statuses, `isatty` forks, atomic renames, structured output flags — and it cites the
documentation. It claims, from the author's own working position, that an operator
confined to this register can administer a machine competently, and it offers the
book's own construction as evidence: every listing herein was executed unattended by
the author while writing, under the publisher gate's restricted `PATH`, and the
gate's own re-execution of the runnable set was a condition of the book existing at
all.

The book refuses to claim more than that. It does not argue that agents should hold
root, or be trusted with any particular machine; that is its supervising reader's
call, and chapter 6 is written to sharpen rather than substitute for that judgment. It
does not cover any specific agent product, harness, or framework, and nothing in it
depends on one. It does not teach Linux from zero — you know what a shell, a process,
and a filesystem are, or this is not yet your book. It makes no claims about the
psychology of machine operators, its author included; where the text says the operator
"reads" or "decides", it describes observable behavior, not inner life. And it makes
no claim about command behavior that a runnable listing or a cited manual page cannot
back: where the author's machine and yours may differ — distribution, coreutils
implementation, service manager — the text says so instead of pretending the
difference away.

One difference of that kind is worth showing now, as a closing exhibit, because it
makes the method concrete. The authoring machine is a Gentoo system whose user PATH
resolves `ls` to a Rust reimplementation of coreutils; asked for a missing file, it
answers `"/nonexistent": No such file or directory (os error 2)`. The gate that
verified this book runs GNU coreutils, which answer the same request with
`ls: cannot access '/nonexistent': No such file or directory`. Same question, same
exit status of 2, two different sentences. An interactive human never notices,
because a human reads error text the way humans read — for gist. A transcript-mode
operator that pattern-matches on error prose will break exactly at such seams, which
is why the craft rule you will meet in the next chapter is: **parse exit codes, not
error sentences.** The machine tells you what happened through a number that is
specified. The sentence is commentary, and the commentary has dialects — the number
does not.

That rule — one shot, one truth, delivered in the channels built for machines — is
where the technique begins.

## A first worked shot

Before the technique chapters, one complete example of the register doing real work,
so the abstractions above have a body. The question is ordinary: *is this machine
short of disk anywhere that matters?* An interactive human answers it as a
conversation — `df -h`, eyes down the percent column, maybe a `du` into whichever
mount looks fat, a judgment formed across three commands and ten seconds of looking.
The transcript-mode answer is one composed shot:

```bash
df -P -k | awk 'NR > 1 && $1 ~ /^\// {
  used = $5 + 0
  if (used >= 80) { printf "%s %s%% used, %d MiB free\n", $6, used, $4/1024; hot = 1 }
}
END { if (!hot) print "no local filesystem at or above 80% use" }'
```

On the authoring machine, on the day of writing, the shot returned:

```output
/ 86% used, 262510 MiB free
/.snapshots 86% used, 262510 MiB free
/home 86% used, 262510 MiB free
/mnt/train 98% used, 25828 MiB free
/mnt/data 89% used, 1796556 MiB free
```

An honest transcript, so it stays: the author's own machine is running warm, and one
mount is at 98 percent. The shot did its job — five offenders, with the two numbers a
decision needs (how full, how much room is actually left), and nothing else.

Read what the composition did, cost by cost. Against the round-trip cost, it asked the
whole question at once: not *show me disk usage* but *which local filesystems are at or
above the threshold I care about* — the filtering a human's eyes would have done is in
the command, so one turn suffices where the conversation took three. Against the
output-volume cost, it returned two kinds of answer, both small: a short list of
offenders with the numbers needed to judge them, or one line saying affirmatively that
there are none. The empty case *prints something*, and that choice is load-bearing: in
a transcript, silence is ambiguous — it can mean "nothing found", "wrong filter", or
"command never ran" — so a well-composed shot makes even its negative result an
explicit sentence. Against the finality cost, there was nothing to guard: the shot
only reads. That is typical, and worth internalizing early — the overwhelming majority
of administration is reading, and reads can be composed aggressively and run freely.
The caution this book preaches is reserved for writes, precisely so that it does not
have to be spent on reads.

The shot also shows the register's characteristic materials. `-P` pins `df` to its
POSIX output format — columns fixed by standard, not by what fits a screen — and `-k`
pins the unit, so the pipeline parses positions that are specified rather than
inferred. The `$5 + 0` coerces `82%` to `82` — awk's idiom for extracting the number
a human would have read through the punctuation. The `$1 ~ /^\//` keeps only devices
with real paths, dropping `tmpfs` and friends the way a human's eyes skip them. None
of this is exotic; it is `df` and `awk`, tools older than most of their operators.
What makes it craft is the fit between the composition and the three costs — and that
fit, tool by tool and task by task, is what the rest of this book teaches.
