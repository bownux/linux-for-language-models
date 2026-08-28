# Response to v1 findings — Linux for Language Models

Scope: pass-2 critic reviews A (gpt-oss-20b), B (gemma-4-31b), C (qwen2.5-7b,
chunked chapter-by-chapter). Critics A and B recorded no blocking findings; critic C
filed four categorized findings on chapter 3 without blocking/suggestion labels, and
this response conservatively treats all four as blocking. Every finding is answered
below; every suggestion is adopted or declined with a reason. All v2 changes are
confined to `ch03-reading-the-machine.md` (plus the re-measured `manifest.json` and
the fresh `pass1-report.json`); the diff `v1..v2` is the complete inventory.

## Blocking findings

### Critic A
None recorded. No blocking items to answer.

### Critic B
None recorded. No blocking items to answer.

### Critic C-1 — ch3, "Memory: read the answer the kernel already computed" (Factual Error)
**Finding:** the statement that `MemFree` measures memory doing "nothing" is
incorrect; `MemFree` measures memory not currently allocated to any process, which
can include memory used for caching.

**Answer: fixed in wording, rebutted in substance.** The finding's own definition is
the incorrect one: memory in use for caching is allocated and is accounted under
`Cached` and `Buffers`, not under `MemFree`. The kernel's `/proc/meminfo`
documentation (book reference 3) defines `MemFree` as total free RAM — the sum of
the zones' free-list pages — and the finding's closing sentence ("not a factual
error in the kernel's documentation") concedes the documentation does not support
it. The chapter's substantive claims (`used = total - free` is the wrong
arithmetic; `MemAvailable` is the field to read) stand, and critic B's fact-check
sample verified the `MemAvailable` claim independently. The original gloss "memory
doing nothing" was, however, loose enough to invite exactly this misreading, so v2
replaces it with the explicit definition stated in both directions: `MemFree` is
*not* "memory not currently allocated to a process" — process-backed pages, file
cache, and buffers are allocated and live under other keys — it counts only pages
on the allocator's free lists. Location: ch3, "Memory" section; see the v1..v2
diff.

### Critic C-2 — ch3, "The introduction shot" (Padding)
**Finding:** the uptime detail adds unnecessary complexity; host, kernel, and
memory would suffice.

**Answer: rebutted.** Uptime is not ornament in this chapter's system: every
`/proc` counter the chapter reads is an accumulator since boot, so the age of those
accumulators calibrates every rate derived from them ("a rate computed from
counters is meaningless without knowing the counters are 3.4 days deep"), and a
surprisingly short uptime is flagged in the same passage as an independent finding
(the machine rebooted recently), which chapter 4's boot-analysis section builds on.
Each field in the introduction shot carries a stated diagnostic consequence; that
is the passage's organizing principle. Cutting uptime would make the ritual cheaper
and the later arithmetic worse. The manuscript's mechanical padding battery
(compression, near-duplicate, scaffold-share, listicle detectors) reports zero
findings on this chapter in `pass1-report.json`. No cut.

### Critic C-3 — ch3, two-sample rate technique (Safety)
**Finding:** the counter-gap-counter method does not mention race conditions or the
need for reads taken within a known, short interval.

**Answer: fixed, at both instances of the technique.** v2 adds the measurement-
error model in two places. In "Rates need two samples" (the CPU instance): `sleep 1`
is a lower bound, the reads are not instantaneous, scheduler jitter stretches the
true gap, so the subtraction overstates the rate by the unmeasured overhead — with
the two structural mitigations (lengthen the gap, since the error is fixed
overhead; or capture the clock beside each sample via `/proc/uptime` and divide by
the measured gap rather than the intended one). In "The counters between the
samples" (the network/disk instance): the two reads are sequential, not
simultaneous; events can land between them and `/proc` offers no transaction that
freezes both samples; the error is bounded by movement during the *read overhead*,
not the intended interval, so the remedy is a gap large enough that the race is
noise — never an attempt to lock the reads. One clarification retained from the
chapter: each individual read of a `/proc` counter file is internally consistent,
so the uncertainty lives entirely in the gap's length. Locations: ch3, both
sections, in the v1..v2 diff.

### Critic C-4 — ch3, "What a snapshot cannot know" (Unclear)
**Finding:** the section ends without concrete examples of deliberate sampling.

**Answer: fixed.** v2 adds a worked bounded burst sampler to the closing section: a
six-sample, five-second-interval loop over `/proc/pressure/io` and the load
average, with its real captured transcript from the authoring machine — a run that
caught an I/O-pressure spike in mid-decay (2.16 → 0.19 over thirty seconds), used
to show how one early read, one late read, and the six together yield three
different diagnoses. The commentary states the sampler's design rules: fixed count
(never `while true`), interval matched to the suspected timescale, a timestamp per
line, and cheapness enough to run at several intervals when the timescale is
unknown. The listing is marked `no-run` under the book's declared three-class
marking, keeping the gate-executed set at 39 of the 40-listing budget; its
transcript is a real authoring-machine run per the book's marking discipline.
Location: ch3, closing section, in the v1..v2 diff.

## Suggestions

### Critic A
1. Quick-reference cheat-sheet of the 15-line discipline — **declined as a new
   section.** Chapter 8 already closes with "The one-page discipline": fifteen
   numbered lines, placed one section before the coda and written to be printable.
   Duplicating it would be summary-shadow under the press's own padding covenant.
2. Side-by-side `journalctl` vs `systemctl show` permissions table — **declined.**
   The permission difference is taught operationally in chapter 4's postmortem
   ("the journal does not say permission denied; it says nothing here") and
   recorded in the glossary's *journal* entry; pocket-tier density favors the
   worked case over a restating table.
3. FAQ on common pitfalls — **declined.** The book's structure deliberately
   locates each pitfall inside the technique it belongs to (empty journal in ch4,
   SIGPIPE/141 in ch2, `/proc/self` in ch3), with the glossary as the lookup path;
   a FAQ would restate chapter content.
4. Minimal idempotent deployment example (e.g., Docker-based) — **declined.**
   Chapter 5's "When the unit of edit is a directory" is that example in
   tool-neutral form (immutable release trees + atomic symlink flip, rollback as
   the same gesture backward); chapter 1's boundaries keep the book
   product-neutral, so an orchestrator-specific variant is out of scope by design.
5. Version-control / git-hooks ledger section — **declined.** Chapter 8 already
   develops commits-as-ledger (small commits at observable stages, messages that
   say why, clean status at handoff); hooks are a workstation workflow pattern the
   pocket tier cannot open without thinning the operator-focused material.

### Critic B
1. Active cross-reference hyperlinks in the digital edition — **adopted as a
   production note, not a manuscript change.** Cross-references are deliberately
   textual ("chapter 6") in the canonical Markdown so the platform renderer can
   linkify them at publication.
2. Low-spec contrast beside the 64-CPU loadavg reading — **adopted in v2.** The
   loadavg passage now carries both contrasts: a 2-CPU cloud instance (twenty-fold
   oversubscription) and a 1-CPU VPS (a run-queue thirty-nine deep — harm, not
   headroom), pointing at the introduction shot's CPU-count read and the pressure
   files as the direct measure of distress.
3. Mention `jq` beside the `python3` JSON loop — **adopted in v2.** "The JSON
   turn" now names `jq` as the field's dedicated instrument (and notes chapter 5's
   existing one-line `jq` edit), with the stated reason the listings stay on
   `python3`: universal presence beats elegance for one-shot work.

## Gate

Local Pass-1 re-run on this revision: PASS, 0 reject / 0 warn — measured body
25,900+ words (exact figure in `manifest.json`, chapter counts re-measured with the
gate's own counter), executed-listing budget 39/40. `pass1-report.json` from the
run is committed. History is append-only on top of the v1 SHA.
