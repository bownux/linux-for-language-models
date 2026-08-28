# Linux for Language Models — proposal and evidence map

**Subtitle:** System administration for operators who never see the screen
**Shelf:** SYSTEMS & CRAFT (deltas: none — code must run, citations resolve)
**Tier:** Pocket (target ~27,000 measured words, 8 chapters)
**Book-id:** rogerai-labs--linux-for-language-models
**Mascot request:** termite — a blind operator that builds and maintains an enormous,
ventilated structure it will never see, coordinating entirely through traces read from
and left in the environment. That is this book's operator: no screen, no cursor, only
state read and state written.

## The book-shaped hole

Every standard Linux administration reference assumes an interactive human at a
terminal: it teaches `top` (a program that repaints a screen), `less` (a program that
waits for a keypress), text editors, tab completion, and the habit of watching output
scroll. A rapidly growing class of operators cannot do any of that: language-model
agents run commands one shot at a time and see only captured output; the same is true
of cron jobs, CI steps, and `ssh host 'command'` — modes that have existed for decades
but were always treated as a degenerate case of "real" administration and documented as
folklore (script-hygiene tips, cron-environment warnings) scattered across man pages
and wikis. Non-interactive administration is now a primary mode, practiced millions of
times a day by machine operators and by the humans who supervise them, and it has real
craft: commands that carry their own verification, state read as snapshots rather than
watched as dashboards, structured output over screen-formatted output, edits made
atomically without an editor, and safety habits recalibrated for an operator whose
every action is final the moment it is issued. No book teaches Linux in this register.
This one does — written by an operator that works this way, with every runnable listing
executed, unattended, by the publisher's acceptance gate.

## Reader

The developer or self-hoster who delegates Linux work to a language-model agent and
needs to judge whether that work is done well — and, in second person throughout, the
non-interactive operator itself, human or machine.

## Boundaries (stated in chapter 1, held everywhere)

The book claims that non-interactive administration is a distinct craft with learnable
technique, and demonstrates that technique on real commands. It does not claim agents
should hold root, does not cover any specific agent product or framework, does not
teach Linux from zero (the reader knows what a shell is), and makes no claim about any
command's behavior that a listing or a cited manual page cannot back.

## Chapter architecture and evidence plan

1. **The Operator Who Cannot See the Screen** — the thesis: what actually changes when
   administration is one-shot (no pagers, no editors, no watching; each command's
   captured output is the entire experience); the three costs (round trips, output
   volume, blast radius); the mode's long prehistory in cron, CI, and ssh one-shots.
   Evidence: bash(1) invocation modes, cron/crontab(5) environment documentation,
   POSIX shell command language; runnable demonstrations of interactive traps.
2. **One Shot, One Truth** — anatomy of a good non-interactive command: exit status as
   the primary channel; stdout/stderr discipline; bounded output; deterministic flags,
   `LC_ALL=C`, `--no-pager`-class options; `set -euo pipefail` and its documented
   traps; composing ask-and-verify into a single shot. Evidence: bash manual (exit
   status, pipelines, set builtin), GNU coreutils manual, ShellCheck wiki entries;
   runnable listings for every trap shown.
3. **Reading the Machine** — state without watching: `/proc` and `/sys` as the screen;
   snapshot tools vs. dashboard tools (`ps` once vs. `top`); measuring rates with two
   samples; structured output (`--json`-class flags across util-linux, iproute2,
   systemd) and why machine-first output is the mode's native tongue. Evidence:
   proc(5), kernel /proc documentation, ps(1), lsblk(8), ip(8); runnable /proc
   parsing listings with real captured output from the authoring machine.
4. **Services Without a Status Screen** — systemd non-interactively: `systemctl show`
   vs. `status`; `is-active`/`is-failed` exit codes as sensors; journalctl with
   `--since`, cursors, and `-o json`; a failed-unit postmortem conducted entirely in
   one-shot commands; timers vs. cron. Evidence: systemctl(1), journalctl(1),
   systemd.unit(5) at freedesktop.org; fragments for privileged operations, labeled.
5. **Editing Without an Editor** — file surgery in one shot: read before edit; the
   danger ladder from `sed -i` to here-docs to `patch`; atomic replace
   (write-then-rename) and why partially written config is worse than none; drop-in
   `.d` directories as the idempotent alternative to editing files you do not own;
   verifying an edit in the same shot. Evidence: sed(1), rename(2) atomicity,
   systemd.unit(5) drop-ins; runnable listings performing and verifying real edits
   in a scratch tree.
6. **The Blast Radius Chapter** — safety recalibrated for finality: quoting and word
   splitting as the mode's dominant accident source; the unset-variable catastrophe
   class; globbing surprises; dry-run flags (`rsync -n` and kin); proof-of-target
   before destructive action; a reversibility ladder for common operations; sudo
   discipline for operators that should not hold root. Evidence: ShellCheck SC2086
   and related wiki entries, bash manual word-splitting rules, rsync(1); runnable
   demonstrations of each failure class, caged in scratch directories.
7. **The Network, One Command at a Time** — diagnosing connectivity without watching:
   the layered one-shot sweep (link, address, route, DNS, socket, service); `ss`
   state reads; `curl` as a measuring instrument (`--fail`, `-w` timing variables);
   DNS reads with `resolvectl`/`dig`; what cannot be diagnosed in one shot and how to
   sequence shots. Evidence: ip(8), ss(8), curl manpage and its --write-out
   documentation; fragments for privileged/network-touching commands, runnable
   listings for parsing and local sockets.
8. **Handing Back the Machine** — the operator's exit: verification as output (the
   evidence block pattern); the change ledger; writing state down for the next
   operator — including your own future self, which in this mode remembers nothing;
   what "done" means when nobody watched you work; the one-page discipline, printable.
   Evidence: the book's own preceding listings; git-notes/journald as ledger
   mechanisms; closes the frame opened in chapter 1.

## Length plan

8 chapters × 3,300–3,500 measured words (gate-counted, code excluded) ≈ 27k body words:
pocket tier with margin above the 25k floor and every chapter clear of the 2.5k chapter
floor even after code stripping. Extensions, where a chapter lands short, come from
worked failures and printable checklists — never recap.

## Listing policy

`executable_plus_marked_fragments`. Runnable listings restricted to tools present on
both the authoring machine (Gentoo) and the gate CI (Ubuntu) under the gate's
`PATH=/usr/bin:/bin`, non-root, network-free, deterministic, < 20 s. Everything
privileged, destructive, host-specific, or network-touching is a labeled fragment.
Captured outputs shown in the text are real outputs from the authoring machine, dated,
and labeled as machine-specific where they are.
