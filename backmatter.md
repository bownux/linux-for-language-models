# Back Matter

## Glossary

- **accumulator** — a kernel counter that only grows (CPU ticks, bytes, sectors); rates are derived from two reads and a subtraction, never from one read.
- **atomic replace** — writing a complete new file beside the target and renaming it over the old one, so readers see only whole versions; rests on `rename(2)` atomicity within one filesystem.
- **batch (shot)** — several independent reads dispatched as one command line, each labeled, separated by `;` so one failure cannot suppress the rest.
- **bounded poll** — a wait implemented as check / interval / maximum, with an affirmative message on both success and exhaustion.
- **change ledger** — an append-only record, one line per write to the world, kept at the moment of each change; only writes get lines, and every write gets one.
- **cursor (journal)** — an opaque position token in systemd's journal; with `--cursor-file`, each read resumes exactly where the previous one ended.
- **drop-in** — a small file added to a `.d` directory that a configuration owner promises to include; the register's preferred alternative to editing files it does not own.
- **dry run** — a tool mode (`rsync -n`, `patch --dry-run`, `apt-get -s`) that reports what would happen without doing it; the register's rehearsal instrument.
- **evidence block** — a labeled final shot that re-verifies each claim of a task with fresh reads and prints the results as one unit.
- **evidence theater** — verification-shaped output that cannot fail: checks that measure nothing, or reprint what the script itself set.
- **exit status** — the integer a command hands the kernel at death; 0 success, 1–125 program-defined, 126/127 shell "could not run", 128+n death by signal n.
- **fragment (listing)** — a code listing marked not-to-execute because it needs privilege, a network, or state the book has no right to touch.
- **guarded append** — an append made conditional on its own absence (`grep -q || printf >>`), idempotent under retries.
- **here-document** — a multi-line literal fed to a command's stdin; a quoted delimiter suppresses expansion, an unquoted one makes the block a template.
- **idempotence** — the property that applying an operation twice equals applying it once; the register's defense against its own retries.
- **isatty** — the system query "is this descriptor a terminal?"; the fork at which programs choose their human-facing or machine-facing behavior.
- **journal** — systemd's structured, indexed log store; read with bounds (`-u`, `--since`, `-n`, `-p`) and formats (`-o short-iso`, `-o json`, `-o cat`).
- **listener** — a socket in LISTEN state; enumerable via `ss -tlnp` or decoded directly from `/proc/net/tcp`'s state column (`0A`).
- **MemAvailable** — the kernel's own estimate of memory obtainable without swapping; the correct answer to "how much memory is left", unlike `MemFree`.
- **one-shot** — a command dispatched non-interactively whose captured output is the operator's entire experience of its execution.
- **pager** — a program (`less`, `more`) that holds output for a keypress; a hang risk in transcript mode, disarmed with `--no-pager` or `PAGER=cat`.
- **pipefail** — shell option making a pipeline's status reflect any component's failure; interacts with SIGPIPE (status 141) under early-exiting consumers.
- **porcelain** — a tool's documented stable machine-output mode (`git status --porcelain`, `df -P`), as opposed to its human display.
- **pressure (PSI)** — `/proc/pressure/{cpu,memory,io}`: the fraction of time tasks stalled waiting for a resource; measures harm where load measures demand.
- **proof of target** — evidence, printed before a destructive verb runs, that its operand exists and is the thing intended.
- **quarantine** — moving a doomed file into a dated graveyard directory instead of deleting it; total in effect, reversible in fact.
- **repainter** — a program (`top`, `watch`, progress bars) that redraws a screen; replaced in this register by snapshots and accumulators.
- **reversibility ladder** — ranking of operations by undo channel: rename and drop-in at the top; `rm`, truncation, and unrehearsed `--delete` at the bottom.
- **scaffold** — preview/recap prose ("in this chapter…"); a padding pattern this book's own publisher rejects mechanically.
- **shot** — one dispatched command or pipeline plus its captured transcript; this book's unit of work.
- **snapshot** — a single point-in-time read of state (`ps`, `df`, one `/proc` read), as against a dashboard's continuous rendering.
- **strict mode** — `set -euo pipefail`; abort-on-surprise defaults for scripts, with documented dull spots around tested positions and command substitution.
- **transcript** — the captured record of a shot's output and status; in this register, the only place anything can be said to have been observed.
- **two-sample rate** — counter, gap, counter, subtract: the derivation behind every "per second" figure this book reports.
- **unbound output** — a read with no cap (`journalctl` bare, `find` without `-maxdepth`); a volume accident and, under capture, sometimes a hang.
- **vantage** — the host, identity, and moment from which an observation was made; part of the finding, stated or the finding is oversized.

## References

1. isatty(3), Linux man-pages. https://man7.org/linux/man-pages/man3/isatty.3.html
2. proc(5), Linux man-pages. https://man7.org/linux/man-pages/man5/proc.5.html
3. The Linux kernel's /proc filesystem documentation. https://docs.kernel.org/filesystems/proc.html
4. PSI — Pressure Stall Information, Linux kernel documentation. https://docs.kernel.org/accounting/psi.html
5. GNU Bash Reference Manual (exit status, pipelines, the set builtin, redirections, word splitting). https://www.gnu.org/software/bash/manual/bash.html
6. GNU Coreutils Manual (df, sort and locale collation, install, stat, timeout semantics). https://www.gnu.org/software/coreutils/manual/coreutils.html
7. POSIX.1-2017 (IEEE Std 1003.1-2017), The Open Group Base Specifications Issue 7 (shell command language, utility conventions, df -P). https://pubs.opengroup.org/onlinepubs/9699919799/
8. grep(1), Linux man-pages (exit status trichotomy). https://man7.org/linux/man-pages/man1/grep.1.html
9. timeout(1), GNU coreutils via Linux man-pages (status 124). https://man7.org/linux/man-pages/man1/timeout.1.html
10. signal(7), Linux man-pages (128+n convention, SIGPIPE). https://man7.org/linux/man-pages/man7/signal.7.html
11. pipe(7), Linux man-pages (pipe buffer behavior). https://man7.org/linux/man-pages/man7/pipe.7.html
12. ps(1), Linux man-pages (%cpu is lifetime-averaged; column selection). https://man7.org/linux/man-pages/man1/ps.1.html
13. lsblk(8), util-linux via Linux man-pages (JSON output). https://man7.org/linux/man-pages/man8/lsblk.8.html
14. git-status(1) documentation (porcelain formats and their stability promise). https://git-scm.com/docs/git-status
15. systemctl(1), systemd documentation (show, is-active, is-failed, is-system-running, list-units, list-timers). https://www.freedesktop.org/software/systemd/man/latest/systemctl.html
16. journalctl(1), systemd documentation (bounds, output formats, cursors, access control). https://www.freedesktop.org/software/systemd/man/latest/journalctl.html
17. systemd.unit(5), systemd documentation (unit file load paths, drop-in directories). https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html
18. systemd-analyze(1), systemd documentation (boot decomposition, blame, calendar). https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html
19. os-release(5), systemd documentation (machine-readable distribution identity). https://www.freedesktop.org/software/systemd/man/latest/os-release.html
20. crontab(5), Linux man-pages (cron environment). https://man7.org/linux/man-pages/man5/crontab.5.html
21. sed(1), GNU sed via Linux man-pages (-i behavior). https://man7.org/linux/man-pages/man1/sed.1.html
22. rename(2), Linux man-pages (atomicity of rename within a filesystem). https://man7.org/linux/man-pages/man2/rename.2.html
23. install(1), GNU coreutils via Linux man-pages (copy with mode in one step). https://man7.org/linux/man-pages/man1/install.1.html
24. find(1), Linux man-pages (expression evaluation order, -delete, -printf). https://man7.org/linux/man-pages/man1/find.1.html
25. flock(1), util-linux via Linux man-pages (advisory locking from shell). https://man7.org/linux/man-pages/man1/flock.1.html
26. ShellCheck wiki, SC2086: double-quote to prevent word splitting and globbing. https://www.shellcheck.net/wiki/SC2086
27. rsync(1) manual (dry run, --delete, trailing-slash semantics). https://download.samba.org/pub/rsync/rsync.1
28. ip(8), iproute2 via Linux man-pages (-j JSON output, route get). https://man7.org/linux/man-pages/man8/ip.8.html
29. ss(8), iproute2 via Linux man-pages (socket statistics, filters). https://man7.org/linux/man-pages/man8/ss.8.html
30. curl manual page (--fail, --max-time, --retry, --write-out variables). https://curl.se/docs/manpage.html
31. Valve Steam for Linux, issue 3671: "Scary rm -rf steamroot bug" (the empty-variable deletion incident). https://github.com/ValveSoftware/steam-for-linux/issues/3671
32. Libes, D., "expect: Curing Those Uncontrollable Fits of Interaction" (USENIX Summer 1990; NIST publication record). https://www.nist.gov/publications/expect-curing-those-uncontrollable-fits-interaction

## A note on measured outputs

Outputs printed in this book's listings are real transcripts from the authoring
machine (Gentoo Linux, kernel 6.18.31-gentoo-dist, 64 CPUs, 125 GiB memory),
captured 2026-08-27/28 under the publisher gate's environment, and labeled
machine-specific where they are. Quantities that vary run to run (load, rates,
timestamps, temporary paths) will differ on re-execution; statuses and behaviors
are the reproducible claims.
