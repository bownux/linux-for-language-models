# Chapter 3 — Reading the Machine

*Draft status: author draft, gate-checked; human verification pending. Outputs shown
are real, from the authoring machine on the day of writing, and are labeled where they
are machine-specific.*

## The screen was always a rendering

The tools this chapter replaces — `top`, `htop`, the graphical system monitors — do
not have privileged access to the machine's state. They read the same files you can
read, arithmetic the same deltas you can arithmetic, and then spend most of their code
on the part you cannot use: painting the result onto a screen, sixty times a minute,
for eyes. Beneath every dashboard is `/proc` — a pseudo-filesystem the kernel
synthesizes on demand, where every process is a directory, every subsystem publishes
its counters as small text files, and reading a file *is* the measurement. The
interactive tradition put a rendering between the operator and that filesystem. The
transcript tradition removes it: you read the source directly, one shot at a time, and
what would have been a glance at a gauge becomes a line of text in your record — which
is better than the gauge, because it is now evidence with a timestamp rather than a
memory of a needle's position.

The first file to know is the one whose rendering everyone has seen:

```bash no-run
cat /proc/loadavg
```

```output
38.90 39.57 37.57 50/5997 265358
```

That is the authoring machine, mid-book, and the numbers are worth reading closely
because they demonstrate the register's advantage. The three leading figures are the
load average over one, five, and fifteen minutes — the same numbers `top` puts in its
header, here without the tool. The fourth field, `50/5997`, is runnable threads over
total threads; the fifth is the PID most recently assigned, a rough odometer of
process churn. A load near 39 would be alarming on a laptop; on this machine — whose
`ps` output below shows several large model-inference servers resident — it is a
working day. (Scale before judgment, always: on a 2-CPU cloud instance the same
figure would mean twenty-fold oversubscription and a machine in real distress; on
a 1-CPU VPS it would mean the run-queue itself is thirty-nine deep, which is
harm, not headroom. The introduction shot later in this chapter reads the CPU
count for exactly this reason, and the pressure files below measure the distress
directly instead of inferring it.) The point of the example is the *reading*: a snapshot plus knowledge of
the machine's role produced a judgment, no repainting required. And because the
snapshot is text in a transcript, tomorrow's judgment can diff against it, which no
glance at a dashboard ever supported.

`/proc` has a sibling, `/sys`, the kernel's device and configuration tree; where
`/proc` answers *what is happening*, `/sys` mostly answers *what exists and how is it
configured* — block devices, network interfaces, hardware topology. The tools later in
this chapter (`lsblk` among them) are readers of `/sys` in the same sense that `top`
is a reader of `/proc`, and the same logic applies: the file tree is the truth; the
tool is a convenience over it; and when tool and truth disagree, the tree wins.

## The snapshot discipline

Chapter 1 classed `top` among the repainters — programs that assume a watcher — and
promised the snapshot sibling. For processes, the sibling is `ps`, and the discipline
is to ask it precise questions rather than accept its defaults. The `-eo` flags hand
you column selection; `--sort` hands you ordering at the source (recall chapter 2's
preference for source-side bounds); `head` caps the answer:

```bash no-run
ps -eo pid,comm,rss --sort=-rss | head -n 6
```

```output
    PID COMMAND           RSS
   2160 llama-server    21703068
  23868 llama-server    10723596
  24196 python          10217392
  24178 python          5587932
  24179 python          5538964
```

Machine-specific, dated, and honest: the five largest residents on the authoring
machine are two local language-model servers and three Python processes, the largest
holding about 21 GiB resident (RSS reports kibibytes here). One shot produced the
answer a human gets by opening `htop`, sorting by memory, and reading the top of the
table — except the shot's version is reproducible, greppable, and did not require a
terminal that can render a table. Every column `ps` offers is documented in its manual
page; the craft is to request exactly the columns your question needs, because (chapter
2 again) every extra column is transcript volume, and `%cpu` in particular is a trap
the next section defuses.

The snapshot discipline generalizes past processes. `watch df` is a repainting
superstition; `df` run again when you have a reason is the register's version, and the
transcript keeps both readings for comparison. The general form: *interactive
monitoring is repeated snapshots plus human short-term memory; transcript monitoring
is repeated snapshots plus an actual record.* You are not giving up monitoring by
losing the dashboard. You are trading a volatile display for a durable one, and the
trade is in your favor for every question except one — the truly continuous watch, a
thing you genuinely cannot do, and for which the honest answers are the machine's own
recording instruments: counters that accumulate (below), logs that persist (chapter
4), and, where real vigilance is needed, an alerting system configured to do the
watching, which is an *interactive* human's tool too, because humans also sleep.

## Rates need two samples

Here is the trap in `ps -o %cpu`, and it is worth this whole section because the
underlying mistake — treating an accumulated total as a current rate — recurs across
every counter in `/proc`. The kernel does not track "CPU percentage right now"; it
tracks cumulative time each CPU has spent in each state since boot, in the first lines
of `/proc/stat`. A percentage is a *rate*, and a rate needs an interval: two readings
of the accumulator, a known gap between them, and a subtraction. `top` does exactly
this between its repaints — its CPU column is the delta between the frame you see and
the frame before it. `ps`, having no previous frame, reports something else entirely:
the process's CPU time divided by its lifetime — a career batting average, not the
current inning. A process that burned an hour of CPU yesterday and sleeps today still
shows a healthy-looking `%cpu`. Operators who did not know this have restarted the
wrong service on the strength of it.

The register's answer is to take the two samples yourself, which costs one `sleep` and
a subtraction:

```bash
read -r _ u1 n1 s1 i1 rest < /proc/stat
sleep 1
read -r _ u2 n2 s2 i2 rest < /proc/stat
busy=$(( (u2 + n2 + s2) - (u1 + n1 + s1) ))
idle=$(( i2 - i1 ))
echo "cpu busy: $(( 100 * busy / (busy + idle) ))% over 1s"
```

```output
cpu busy: 57% over 1s
```

The first four fields after the `cpu` label are user, nice, system, and idle time, in
clock ticks; two reads a second apart make the delta, and the delta makes an honest
percentage — 57 percent busy across all cores of the authoring machine during that
particular second, a number consistent with the load average shown earlier. (A
production version would fold in the iowait and irq fields that follow; the manual
page for `proc(5)` documents the full row. The four-field version stays within a
teachable line and errs by at most the small slices those states occupy.)

The pattern is the important export: **counter, gap, counter, subtract.** Network
bytes in `/proc/net/dev`, disk sectors in `/proc/diskstats`, interrupts, context
switches — the kernel publishes nearly everything as accumulators, and any "per
second" figure you have ever seen was two samples in a trench coat. In transcript
mode, taking the samples explicitly has a side benefit: the interval is in your
record. A dashboard's "12 MB/s" answers *when? averaged over what?* with a shrug; your
version answers precisely, because you chose the gap and wrote it down. When a rate
matters enough to act on, take a longer gap or several short ones — a single
one-second sample can catch a freak spike or miss one, a caveat the last section of
this chapter returns to.

One honesty note on the arithmetic itself: the interval in that listing is
approximate, not exact. `sleep 1` guarantees *at least* a second, the two file reads
are not instantaneous, and on a loaded machine the scheduler can add jitter between
them — so the true gap might be 1.02 seconds while the subtraction assumes 1.00,
overstating the rate by the same couple of percent. For a triage read that error is
noise; for a rate you will act on or record, shrink it structurally: lengthen the gap
(the error is fixed overhead, so ten seconds of gap makes it ten times smaller), or
capture the clock *with* each sample — read `/proc/uptime` in the same breath as the
counter and divide by the measured gap rather than the intended one. The two reads
themselves need no synchronization beyond this — each read of a `/proc` counter file
is internally consistent — the uncertainty lives entirely in the gap's length, which
is why measuring the gap, rather than trusting it, closes the question.

## Memory: read the answer the kernel already computed

`/proc/meminfo` is the machine's memory ledger, and it is the site of the register's
most durable misreading. The file's first line, `MemTotal`, and second, `MemFree`,
seduce every newcomer into the subtraction `used = total - free` — which on any
healthy Linux machine reports near-exhaustion, because the kernel deliberately spends
otherwise-idle memory on disk cache and reclaims it on demand. `MemFree` is not
"memory not currently allocated to a process." Process-backed pages, file cache, and
buffers are all allocated; they live under other keys (`Cached`, `Buffers`, the
anon/file breakdowns). `MemFree` counts only pages on the allocator's free lists —
the kernel documentation defines it as the sum of the zones' free pages — so a
well-run kernel keeps `MemFree` low on purpose: idle pages are wasted pages. The
number that answers the question people actually have — *how much memory could
applications obtain before the machine starts to struggle* — is `MemAvailable`,
an estimate the kernel itself computes and publishes precisely because the naive
subtraction misled a generation of monitoring scripts; the kernel documentation for
`/proc/meminfo` says as much in nearly those words.

```bash
awk -F'[: ]+' '/^MemTotal|^MemAvailable/ {printf "%s %.1f GiB\n", $1, $2/1048576}' /proc/meminfo
```

```output
MemTotal 125.1 GiB
MemAvailable 60.8 GiB
```

The authoring machine again: 125 GiB fitted, 61 GiB genuinely obtainable — while
`MemFree` at the same moment stood far lower, the gap being cache doing useful work.
The shot embodies the section's rule: **when the kernel publishes a computed answer,
read the answer; do not re-derive it worse.** The same rule retires several other
folk formulas — swap arithmetic, dirty-page guesswork — each of which has a
`meminfo` field computed by the people who wrote the allocator. The transcript-mode
operator's edge here is again the record: `MemAvailable` sampled in every diagnostic
shot builds, for free, the time series that distinguishes "this machine is sized
tight" from "something is leaking", a distinction a single glance can never make.

## Pressure: the kernel's own verdict on scarcity

Load, busy percentages, and `MemAvailable` all measure *supply*; the question
underneath most performance complaints is about *suffering* — is anything actually
waiting? Modern kernels answer that question directly, through the pressure stall
information files, and the answer belongs in this chapter because it is another
computed verdict of the `MemAvailable` kind — arguably the best three files in
`/proc` for a one-shot triage:

```bash
for res in cpu memory io; do
  f=/proc/pressure/$res
  if [ -r "$f" ]; then
    printf "%-6s %s\n" "$res" "$(head -n 1 "$f")"
  else
    printf "%-6s pressure interface not available\n" "$res"
  fi
done
```

```output
cpu    some avg10=0.00 avg60=0.00 avg300=0.04 total=366454226
memory some avg10=0.00 avg60=0.00 avg300=0.00 total=35693827
io     some avg10=0.02 avg60=0.17 avg300=0.09 total=1547742828
```

Each `some` line reports the percentage of time, averaged over ten, sixty, and
three hundred seconds, during which *at least one task sat stalled* waiting for
that resource. The authoring machine, mid-book: effectively zero everywhere, a
touch of I/O wait in the last minute — the kernel's own statement that, load
average of thirty-nine notwithstanding, nothing on the machine is starving. That
is the reading to internalize: chapter-opening load figures counted *demand*;
pressure measures *harm*, and the two diverge exactly when intuition most needs
correcting (sixty-four cores absorb enormous demand without harm; a two-core
cloud instance shows harm at load figures that look innocent). The pre-averaged
windows also spare the two-sample dance for a first look — the kernel maintained
the rate for you, at three horizons, which is why a pressure read plus a
`MemAvailable` read makes the cheapest credible answer to "is this machine
struggling right now". The listing's guard clause is not decoration: the
interface requires a reasonably modern kernel and can be compiled or booted out,
so the honest shot prints an affirmative "not available" — chapter 2's rule,
already at work — rather than letting absence impersonate health.

## The JSON turn

Column scraping — the `awk '{print $4}'` idiom this book has already used — carries a
quiet fragility: it binds your shot to a tool's *visual layout*, which was never a
contract. Columns get added, widths shift, a mount point with a space in it splits
one field into two, and the shot keeps succeeding while meaning something else. The
system's toolmakers know this, and over the last decade the major system utilities
have grown a machine-first answer: native JSON output. `lsblk -J`, `ip -j`, `ss
--json`, `findmnt -J`, `systemctl`'s `show` and `--output=json` modes — the pattern
(util-linux, iproute2, and systemd converged on it independently) is that the tool
that owns the data serializes it with named keys, and the reader addresses fields by
name rather than by position:

```bash no-run
lsblk -J -o NAME,TYPE,SIZE,MOUNTPOINT | python3 -c '
import json, sys
for dev in json.load(sys.stdin)["blockdevices"]:
    print(dev["name"], dev["type"], dev["size"], dev.get("mountpoint") or "-")'
```

```output
sda disk 14.6T -
sdb disk 0B -
nvme1n1 disk 1.8T -
nvme0n1 disk 1.8T -
```

The authoring machine's disks: a large rotational drive, an empty card-reader slot
(`0B` — an honest artifact worth leaving in, since your parsers must survive such
entries too), and two NVMe devices whose partitions, children in the JSON tree, are
omitted here for space. Three properties make the JSON form worth its verbosity.
Names instead of positions: a future `lsblk` adding a column cannot silently shift
your field. Explicit nulls: an empty mount point arrives as `null`, not as a missing
column that re-numbers its neighbors — the exact accident that breaks whitespace
scraping. And a real parser: `python3` is present on effectively every machine this
book's reader will touch, and `json.load` plus a loop replaces a class of `awk`
fragility with a language that has actual data structures. (Where it is installed,
`jq` is the field's dedicated instrument for exactly this — terser than the loop
above, worth knowing, and chapter 5 uses it for a one-line edit; `python3` carries
the listings here because it is effectively always present, which for one-shot work
beats elegance.) The register's rule of precedence follows: **JSON flag if the tool has one; documented stable format
(`--porcelain`, `-P`) if not; positional scraping only against formats a standard
pins, and never against human-layout output you do not control.**

Two honest caveats. First, availability: the JSON flags are newer than the tools, and
a machine past its distribution's support window may carry an `lsblk` without `-J`;
the fallback order above is a gradient, not a cliff. Second, reach: some of the
richest JSON emitters live in `sbin` directories — `ip -j` chief among them — and
minimal `PATH`s (cron's, constrained sandboxes', this book's own gate) may not reach
them. On the authoring machine `ip` resolves at `/usr/bin/ip`; on the gate's Ubuntu
runner it does not resolve within the gate's `PATH` at all, which is why the `ip`
listings in chapter 7 are labeled fragments rather than runnable. The seam is itself
the lesson: *which tools your shot can reach is part of your machine's state*, and
`command -v tool` is the one-shot read that answers it before a 127 answers it for
you.

## Processes up close

The `/proc` directory of a single process is the register's microscope, and three of
its files answer most of the questions a stuck or mysterious process provokes.
`cmdline` holds the process's exact argument vector, NUL-separated — the truth behind
`ps`'s sometimes-truncated `COMMAND` column:

```bash
tr "\0" " " < /proc/$$/cmdline; echo
```

```output
bash /tmp/oailly-gate-la2ln9dv/listing.sh
```

The output is the gate's own execution of this very listing — the process examining
itself, which is also this book's provenance model in miniature. Note the `$$` where
you might have expected the more famous `/proc/self`. The first draft of this listing
used `self`, and its transcript read, absurdly, `tr \0` — because a redirection is
opened by the forked child *after* the fork, so `self` resolved to the child that was
about to become `tr`, and the listing examined the examiner. `$$` expands to the
shell's own PID before any forking, and asks the intended question. The trap is a
pure specimen of this register's failure style: nothing errored, the output looked
plausible at a glance, and only reading the answer against the question exposed it —
the shape check from chapter 2's reading routine, earning its keep. Alongside `cmdline` sit
`cwd`, a symlink to the process's current directory — the first question for any
process writing files "somewhere" — and `environ`, the environment it was born with,
NUL-separated like `cmdline`, and the fastest way to learn which proxy, locale, or
credential path a misbehaving service actually received, as opposed to what its unit
file intended. Deeper files repay acquaintance: `status` for a readable summary
including memory and thread counts, `fd/` for every open file descriptor (a directory
listing that has solved a thousand "what is holding this file open" mysteries).

The permission rule: you may read these files for your own processes; other users'
processes, and much of `fd/`, require matching identity or privilege, and a
`Permission denied` here is the system working, not an obstacle to route around.
Chapter 6 takes up the discipline of operating below root; the reading habits of this
chapter are deliberately chosen to live comfortably there.

## The counters between the samples

The counter-gap-counter pattern promised earlier deserves one full worked instance
beyond CPU, because network throughput is the question it answers most often in
practice. `/proc/net/dev` is the kernel's per-interface ledger: one row per
interface, cumulative received bytes in the second column, transmitted bytes in the
tenth, both counting since the interface came up. Two reads and a subtraction make
the throughput figure that bandwidth dashboards render:

```bash
r1=$(awk 'NR > 2 {rx += $2} END {print rx}' /proc/net/dev)
sleep 1
r2=$(awk 'NR > 2 {rx += $2} END {print rx}' /proc/net/dev)
echo "ingress: $(( (r2 - r1) / 1024 )) KiB/s across all interfaces"
```

```output
ingress: 319 KiB/s across all interfaces
```

The authoring machine, drawing a modest stream during the write. The `NR > 2` skips
the file's two header lines — position-based, which the previous section just warned
about, and defensible here only because `proc(5)` documents this layout as an
interface; even so, a reader on an unfamiliar kernel checks the header once before
trusting the columns. Summing all interfaces is the deliberate choice for a
first-look shot: it cannot miss traffic on an interface you forgot existed, and a
follow-up shot can always split by row once the total says something is moving.
That two-shot rhythm — cheap aggregate first, targeted breakdown second — spends
round trips the way chapter 1's economics recommends: the second turn is bought only
when the first turn's answer justifies it.

The same two-column subtraction against `/proc/diskstats` yields per-device I/O
rates, with one refinement worth knowing: field 10 of that file (milliseconds spent
doing I/O) is the raw material of the "utilization" figure `iostat` renders, and a
delta there that approaches the sampling interval means the device was busy nearly
the whole gap — the single most useful one-number answer to *is this disk the
bottleneck*.

The two `awk` reads are sequential, not simultaneous. Packets (or sector completions)
can land in the few milliseconds between them, and a loaded scheduler can stretch
that further. That is a real race — `/proc` has no transaction that would freeze
both samples — but it is also why the gap is a full second, or ten, rather than two
back-to-back reads. The error is bounded by however much moved during the *read
overhead*, not during the intended interval. Lengthening the gap, or capturing
`/proc/uptime` beside each sample as the CPU section already recommended, shrinks
the race the same way. Do not try to lock the two reads together; make the gap large
enough that the race is noise.

## The file as a fact

Processes and counters are half of a machine's observable state; files are the other
half, and the register reads them with the same preference for precise questions.
The workhorse is `stat`, which answers with exactly the fields you request:

```bash no-run
cd "$(mktemp -d)"
printf "data\n" > f.txt
stat -c "%n %s bytes, mode %a, modified %y" f.txt
```

```output
f.txt 5 bytes, mode 644, modified 2026-08-27 22:00:09.705241936 -0700
```

Size, permissions, and modification time are the triage triple: together they answer
*is this the file I think it is, can the process that needs it read it, and has
anything touched it lately* — three of the five questions in most configuration
mysteries. (`stat`'s `-c` formats are GNU spellings; the flag set differs on BSD
userlands, one more reason the book's listings declare the platform they ran on.)
The habit to unlearn is answering these questions by parsing `ls -l`, whose output
was designed for eyes, varies with locale and version, and mangles unusual
filenames; `ls` remains the right tool for *seeing* a directory, and the wrong tool
for extracting facts from one.

At directory scale, the precise question is usually temporal — *what changed
recently?* — and `find` answers it in one bounded shot. In a scratch tree seeded
with two files touched two hours ago and two written now:

```bash
cd "$(mktemp -d)"
mkdir -p etc logs
touch -d "2 hours ago" etc/old.conf logs/old.log
printf "x\n" > etc/fresh.conf
printf "y\n" > logs/today.log
find . -type f -mmin -60 -printf "%TY-%Tm-%TdT%TH:%TM %p\n" | LC_ALL=C sort -k2
```

```output
2026-08-27T22:00 ./etc/fresh.conf
2026-08-27T22:00 ./logs/today.log
```

The old files are correctly absent; the fresh ones arrive timestamped and sorted
under a pinned locale. Pointed at `/etc` with a bound of minutes-since-the-incident,
this shape of shot is the fastest first move in "it worked yesterday" forensics —
and pointed at a tree you are *about* to modify, it snapshots the before-state your
chapter 8 handoff will want. The `-printf` timestamp format is chapter 2's
determinism rule applied: ISO-shaped, sortable as text, and immune to the month-name
localization that makes default `find` and `ls` timestamps unjoinable across
machines.

## The introduction shot

The chapter's reads compose into a ritual worth naming: the first shot an operator
dispatches on any machine it has not met — or has not met *recently*, which for an
operator without persistent memory may be every machine, every session. Identity,
scale, and age, in one bounded transcript:

```bash
. /etc/os-release 2>/dev/null
echo "host: $(uname -n) | kernel: $(uname -r) | os: ${PRETTY_NAME:-unknown}"
echo "cpus: $(nproc) | mem: $(awk '/^MemTotal/ {printf "%.0f GiB", $2/1048576}' /proc/meminfo) | up: $(awk '{printf "%.1f days", $1/86400}' /proc/uptime)"
echo "sampled: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

```output
host: RogGentoo | kernel: 6.18.31-gentoo-dist | os: Gentoo Linux
cpus: 64 | mem: 125 GiB | up: 3.4 days
sampled: 2026-08-28T05:15:58Z
```

The authoring machine introduces itself: sixty-four CPUs, the 125 GiB the memory
section already met, three and a half days since boot, kernel and distribution
named exactly. Each field earns its place by changing what subsequent shots should
assume. The distribution decides package manager, service manager, and which
dialect seams (chapter 1's `ls` lesson) to expect. The uptime bounds every "since
boot" accumulator this chapter reads — a rate computed from counters is meaningless
without knowing the counters are 3.4 days deep — and a *surprisingly short* uptime
is itself a finding: the machine rebooted recently, and whatever you were sent to
diagnose may have started there. The CPU count calibrates load averages (the 38.9
that opened this chapter reads very differently over 64 cores than over 8 — about
sixty percent of capacity, not five hundred). And the closing UTC timestamp is
chapter 2's determinism rule applied to the transcript itself: every reading in
the session dates from somewhere, and the introduction shot is where the somewhere
is written down. `/etc/os-release` deserves its footnote: it is a *sourceable
file* by design — the distribution publishes its identity as shell variables,
machine-first, one more place the system turns out to have been expecting you.

## What a snapshot cannot know

The chapter closes on its own limits, because the snapshot discipline has a failure
mode and the honest version of this book names it. A snapshot is a point sample, and
point samples miss what happens between them: the process that spikes for two seconds
each minute, the disk that stalls only under a nightly job, the memory that climbs
for an hour and collapses before your read. Where an interactive human's dashboard
would also likely miss these — human attention is a sparse sampler too — the
transcript operator has three honest recourses. Sample deliberately: several reads at
noted intervals, chosen to bracket the suspected behavior, beat one read at an
arbitrary moment. Concretely, a bounded burst sampler is one loop:

```bash no-run
for i in 1 2 3 4 5 6; do
  printf "%s  io-some=%s  load1=%s\n" \
    "$(date -u +%H:%M:%S)" \
    "$(awk -F"avg10=" "NR==1 {split(\$2,a,\" \"); print a[1]}" /proc/pressure/io)" \
    "$(cut -d" " -f1 /proc/loadavg)"
  sleep 5
done
```

```output
16:21:32  io-some=2.16  load1=4.89
16:21:37  io-some=1.45  load1=4.98
16:21:42  io-some=0.79  load1=5.06
16:21:47  io-some=0.53  load1=5.13
16:21:52  io-some=0.29  load1=4.96
16:21:57  io-some=0.19  load1=4.89
```

Thirty seconds of the authoring machine, and the run happened to catch something a
single read would have flattened: an I/O pressure spike *in mid-decay* — 2.16
falling to 0.19 across six samples while the load average barely moved. One read at
16:21:32 would have said "I/O problem"; one read at 16:21:57 would have said "all
quiet"; the six together say "a burst just ended", which is a different diagnosis
from either. The sampler's design carries the section's rules in miniature: a fixed
count (never `while true` — chapter 1's hang), an interval chosen to bracket the
suspected behavior's timescale, a timestamp on every line so the record can be
correlated with logs afterward, and the whole thing cheap enough to run three of at
different intervals when you do not yet know the timescale you are hunting. Use the accumulators: the kernel's counters integrate what happened
*between* your samples — a delta in `/proc/diskstats` over ten minutes has seen every
I/O in the gap, including the burst your snapshots straddled. And use the machine's
own memory: the logging and journal infrastructure of chapter 4 is precisely the
machine watching itself continuously so that no operator, human or otherwise, has to
pretend a glance was a vigil.

Read once, read precisely, subtract when you need a rate, prefer the kernel's own
computed answers, address fields by name, and know what your sample cannot contain.
That is reading the machine. The next chapter points the same discipline at the part
of the machine that talks back: its services, and the journal where they confess.
