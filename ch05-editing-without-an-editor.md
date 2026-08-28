# Chapter 5 — Editing Without an Editor

*Draft status: author draft, gate-checked; human verification pending. Every worked
edit in this chapter runs in a scratch directory created by the listing itself; none
touches real configuration.*

## The costume and the change

Chapter 1 classed editors among the traps with no non-interactive flag, because
editing *is* the interactive act — a human, a cursor, and a buffer in conversation.
But step back from the mechanism and every editor invocation in administration is the
same underlying event: a file had one content before, and must have another content
after. The cursor was never essential; it was the human interface to a substitution.
The register performs the substitution directly, and it has a ladder of instruments
for doing so, ordered by how much of the file they touch: append a line, substitute
within lines, apply a structured diff, or replace the whole file. Each rung has its
tool, its characteristic accident, and its verification, and this chapter climbs them
in order. Two rules span all rungs and are the chapter's spine. First: **read before
you edit** — every mechanical edit is composed against an assumption about what the
file currently contains, and the assumption must be checked in the same session,
because in this register nobody is watching the file between your turns. Second:
**an edit is not done when the write succeeds; it is done when the read-back proves
the file now says what you intended** — chapter 2's ask-and-verify, applied to the
substrate everything else configures itself through.

Reading before editing has one subtlety worth its own paragraph: make sure the file
you read is the file you will edit. Configuration trees are full of symlinks —
`/etc/resolv.conf` is famously one on most modern systems — and an edit aimed at a
link can follow it or replace it depending on the tool, two very different outcomes.
The one-shot check is `readlink -f`, which resolves the entire chain to the physical
target:

```bash
cd "$(mktemp -d)"
mkdir real
printf "x=1\n" > real/app.conf
ln -s real/app.conf app.conf
readlink -f app.conf
```

```output
/tmp/tmp.kqCRsHy8rJ/real/app.conf
```

The name on the surface and the file on disk differ, and later rungs of the ladder
treat them differently: `sed -i` and the atomic-replace pattern both *replace the
link itself* with a regular file unless pointed at the resolved target — silently
severing an arrangement someone built on purpose. Resolve first; edit the target.

## Appending, and the accident of doing it twice

The lowest rung is adding lines to a file, and its tool is the shell's own `>>` — with
`printf` rather than `echo` supplying the bytes, since `printf` behaves identically
everywhere while `echo`'s treatment of flags and escapes varies by shell and mode.
The rung's characteristic accident is not the append that fails but the append that
*succeeds twice*. One-shot operators re-run commands: a turn times out with its work
half-known, a script is retried after a fix, an agent replays a step from an earlier
plan. An append inside that replay duplicates the line — and duplicated configuration
is not always harmless; a repeated `PATH` export is noise, but a repeated firewall
rule, kernel parameter, or cron entry can change behavior. The register's discipline
is to make every append *conditional on its own absence* — a guarded append,
idempotent by construction:

```bash
cd "$(mktemp -d)"
printf "PATH=/usr/bin\n" > env.conf
for attempt in 1 2; do
  grep -q "^EDITOR=" env.conf || printf "EDITOR=false\n" >> env.conf
done
cat env.conf
echo "lines: $(wc -l < env.conf)"
```

```output
PATH=/usr/bin
EDITOR=false
lines: 2
```

The loop simulates the retry: two attempts, one appended line, because the second
attempt's `grep -q` found the first's work and the `||` skipped the write. Run the
unguarded version and the file ends at three lines — a fact the listing's final
`wc -l` exists to make checkable, since an idempotence claim is exactly the kind of
claim chapter 2 says to verify rather than assert. For multi-line insertions the same
guard anchors on a marker comment (`grep -q "^# BEGIN myblock"`), which also gives a
future *removal* a handle to find the block by. Guarded appends are the smallest
instance of a theme this chapter returns to at the top of the ladder: in a register
where re-execution is routine, the well-formed edit is one whose second application
changes nothing.

The multi-line append's instrument is the here-document, and it carries a quoting trap
sharp enough to demonstrate rather than describe. The delimiter's quoting decides
whether the shell expands variables inside the block:

```bash
cd "$(mktemp -d)"
name=world
cat > expanded.txt <<EOF
hello $name
EOF
cat > literal.txt <<'EOF'
hello $name
EOF
cat expanded.txt literal.txt
```

```output
hello world
hello $name
```

Unquoted `EOF`: the block is a template, and `$name` became `world`. Quoted `'EOF'`:
the block is literal, and `$name` survived as text. Both behaviors are wanted — the
first for generating config from session facts, the second for writing files that
themselves contain shell syntax (a script, a crontab line, a systemd `ExecStart` with
specifiers). The accident is using the first mode while believing you are in the
second: every `$` in the payload silently expands — usually to empty, per chapter 2's
unset-variable economics — and the written file is a corrupted version of the
intended one that *looks* right at a glance because its shape survived. When a
here-doc's payload contains a single `$`, `` ` ``, or backslash you intend literally,
quote the delimiter; make the exceptions deliberate.

## Substitution in place, guarded

The middle rung changes existing lines, and its tool is `sed -i`. Used bare, it is
the most accident-prone instrument in this chapter, for a structural reason: `sed`
applies a pattern to *whatever matches*, and the register's operator is not watching
matches happen. A pattern that matches zero times edits nothing — silently, exit
status 0, the calm face again. A pattern that matches more lines than intended edits
all of them, equally silently. Both accidents are the same root cause — the edit's
precondition lived only in the operator's head — and both have the same cure: count
the matches first, in the same shot, and proceed only when the count is the expected
one:

```bash
cd "$(mktemp -d)"
printf "retries = 3\ntimeout = 30\n" > service.conf
n=$(grep -c "^retries = " service.conf)
[ "$n" -eq 1 ] \
  && sed -i "s/^retries = .*/retries = 5/" service.conf \
  && grep "^retries" service.conf
```

```output
retries = 5
```

Three moves in one transcript: the count established the precondition (exactly one
line will be touched), the substitution ran only inside that guarantee, and the
read-back printed the proof. Had the file held two `retries` lines, or none, the
chain would have stopped before the edit with the count as its explanation — a
failure that costs one turn and explains itself, against a silent mis-edit that
costs a debugging session weeks later. The pattern discipline inside the `sed`
expression matters equally: anchor to the line's start (`^retries = `), match the
whole value (`.*`), and prefer patterns that restate the line's full grammar over
minimal fragments that happen to work today. `sed -i` also accepts a backup suffix
(`-i.orig`), which drops a sibling copy before rewriting — cheap insurance, though
the chapter's top rung offers something better, and one caveat repeats from the
symlink section: `-i` writes a new file over the *name*, so pointed at a link it
replaces the link.

For edits beyond a line's internals — inserting a block after a marker, deleting a
stanza — resist the temptation to compose ever-cleverer `sed` programs. The register
has a better instrument one rung up.

## The diff is the native edit

An interactive human edits by manipulating a buffer; the register's structurally
best edit format is the unified diff — precisely because it is *both* the change and
its documentation, in a form `patch` can apply, `git` can ingest, and a supervising
reader can review in the transcript before anything happens. A diff states its
context lines, so it refuses to apply against a file that has drifted from the
version it was composed against — the read-before-edit rule, enforced by the file
format itself.

```bash
cd "$(mktemp -d)"
printf "alpha\nbeta\ngamma\n" > config.txt
printf "alpha\nBETA\ngamma\n" > intended.txt
diff -u config.txt intended.txt > change.diff
patch --dry-run -p0 config.txt < change.diff \
  && patch -p0 config.txt < change.diff \
  && cat config.txt
```

```output
checking file config.txt
patching file config.txt
alpha
BETA
gamma
```

The rehearsal is the point of the composition: `--dry-run` verifies the diff applies
cleanly — against the real file, changing nothing — and only its success unlocks the
real application. That two-step is chapter 6's dry-run principle arriving early, and
with diffs it is airtight in a way `sed` guards approximate: the dry run checks the
*entire* precondition (every context line), not just a count. Two status notes for
the transcript reader: `diff` itself answers like `grep` — 0 for identical, 1 for
different, 2 for trouble — so a `diff` "failing" with 1 mid-script is the expected
answer *the files differ*, not an error; and a real `patch` failure leaves `.rej`
files naming exactly the hunks that could not land, which are evidence to read, not
litter to delete. On any machine with git present, `git diff --no-index`, `git apply
--check`, and `git apply` make the same ladder with sharper diagnostics; and inside
an actual repository, the repository's own tooling — not this chapter's — is the
right instrument, with version control providing the undo channel the register
otherwise has to build by hand.

## Replace the whole file, atomically

The top rung retires editing altogether: generate the complete intended content,
validate it, and swap it into place. This is the register's preferred rung for any
file whose entire content the operator can own — because it is idempotent by
construction (generating the same content twice converges), reviewable (the new
content can be shown whole in the transcript), and, done correctly, atomic. The
correctness hinges on one syscall guarantee: `rename(2)` within a filesystem is
atomic — any process opening the path sees the old complete file or the new complete
file, never a half-written intermediate. `mv` onto an existing name, same
filesystem, is that syscall in shell clothing:

```bash
cd "$(mktemp -d)"
printf '{"port": 8080}\n' > app.json
printf '{"port": 9090}\n' > app.json.new
python3 -c 'import json; json.load(open("app.json.new"))' \
  && mv app.json.new app.json \
  && cat app.json
```

```output
{"port": 9090}
```

The sequence is validate-then-swap, and the order carries the safety: the JSON parse
ran against the *staged* file, so a generation bug — truncated output, an unclosed
brace, chapter 2's substitution silently emptying a variable — is caught while the
live file is still intact, and the broken candidate never existed at the live path
for even a millisecond. Any consumer that opened `app.json` mid-operation got a
complete document. The pattern's fine print earns respect: the staging file must be
*in the same directory* as the target (cross-filesystem `mv` degrades to
copy-then-delete, which is not atomic — and `/tmp` is routinely a different
filesystem, so staging there forfeits the guarantee); the swap replaces metadata
along with content, so files with deliberate modes or owners want `chmod`/`chown` on
the staged copy before the `mv`; and the validator should be the *consumer's* grammar
— `python3 -c json.load` for JSON, `sshd -t` or `visudo -c` (privileged, fragments
by this book's rules) for the system files that ship their own checkers, a service's
own config-test flag where one exists. A validator that could have run and did not is
the difference between an edit and a gamble.

## When the unit of edit is a directory

The atomic swap has a limit the honest version of this chapter must state: it
covers one file. A change spanning several files — a config directory, an
application release, a static site — cannot be made atomic by renaming them one at
a time; between the first rename and the last, every reader sees a mixture of
versions, and the mixture is exactly the corruption atomicity exists to prevent.
The filesystem has no multi-file transaction. What it has is one more atomic
rename, applied a level up — the symlink flip, the pattern every deployment tool
reinvents:

```bash
cd "$(mktemp -d)"
mkdir -p releases/v1 releases/v2
printf "old\n" > releases/v1/app.txt
printf "new\n" > releases/v2/app.txt
ln -s releases/v1 current
cat current/app.txt
ln -sfn releases/v2 current.new && mv -T current.new current
cat current/app.txt
readlink current
```

```output
old
new
releases/v2
```

The live name, `current`, is a symlink; versions are complete, immutable sibling
trees; and the "edit" is a rename of a freshly built link onto the live name —
one `rename(2)`, so every reader holds either wholly-v1 or wholly-v2, never a
blend. The awkward spelling of the flip is load-bearing and worth reading
closely: `ln -sfn` *onto the live name directly* would not be atomic (with a
directory target it can pass through a deleted-then-recreated state, and some
implementations descend *into* the target instead), so the new link is created
beside the live one and `mv -T` — the `-T` forbidding the same descend-into
misreading — performs the actual instantaneous cutover. Rollback is the same
gesture pointed backward, which places this pattern on the top rung of chapter
6's reversibility ladder: the entire previous version still exists, untouched,
one flip away. The pattern's tax is discipline about state: nothing writes
*into* a released tree (releases are built complete, then flipped), and anything
the application mutates at runtime lives outside the versioned trees entirely.
Paid, the tax buys the multi-file edit this chapter otherwise could not offer.

## Structured formats want structured editors

Every instrument so far treats files as lines of text, and for the classic
`key = value` formats that is the truth of them. But a growing share of what
administration edits is *structured* — JSON, YAML, TOML — and line tools are the
wrong instrument for tree-shaped data, in a way the register's operators are
specially positioned to get wrong: a `sed` substitution against a JSON file often
works, today, against this file, and that success teaches a habit that fails the
first time the target key appears twice at different depths, or gains a string
value containing the pattern, or arrives reserialized with different whitespace.
Line tools match *rendering*; the file's meaning lives in its *parse*. The correct
instrument edits the parse:

```bash
cd "$(mktemp -d)"
printf '{"port": 8080, "workers": 4, "debug": false}\n' > app.json
python3 - <<'PYEOF'
import json, pathlib
p = pathlib.Path("app.json")
cfg = json.loads(p.read_text())
cfg["workers"] = 8
tmp = p.with_suffix(".json.new")
tmp.write_text(json.dumps(cfg, indent=2) + "\n")
tmp.replace(p)
print("workers now:", json.loads(p.read_text())["workers"])
PYEOF
```

```output
workers now: 8
```

Twelve lines that assemble the whole chapter in miniature: parse (which *is* the
read-before-edit — a malformed file dies here, before harm), modify by addressing
the key in the tree rather than a pattern in the text, stage, atomically replace
(`Path.replace` is the same `rename(2)` under the `mv` of the previous section),
and re-parse as the read-back proof. For operators with `jq` installed, `jq
'.workers = 8' app.json > app.json.new` reaches the same place for JSON one-liners;
`python3` earns the listing because it is *already there* on effectively every
machine and speaks YAML and TOML through the same pattern (the standard library
reads TOML natively; YAML needs the common third-party module). One structural
honesty note: parse-and-reserialize normalizes formatting and drops comments where
the format allows them (YAML, TOML), which is a real cost in human-maintained
files — one more argument for the drop-in answer below, where your generated file
is wholly yours and the human's stays untouched.

## Mode, owner, and the moment of creation

An edit's content can be right while the file itself is wrong: readable by the
world when it holds a secret, owned by root when a service user must write it —
failures invisible in a `cat` and fatal in operation. The register's discipline is
to set metadata *at creation, in the same shot*, never as a remembered follow-up.
The shell's default is governed by `umask` (measured `0022` on the authoring
machine: new files arrive world-readable), which is the wrong default for
credentials, and the fix-it-later `chmod` leaves a window in which the secret was
exposed — a window an operator with no continuous presence cannot even measure.
The single-shot instrument is `install`, which combines copy, mode, and (with
privilege) ownership in one atomic gesture:

```bash
cd "$(mktemp -d)"
printf "secret=1\n" > cred.new
install -m 600 cred.new cred.conf
stat -c "%n mode %a" cred.conf
```

```output
cred.conf mode 600
```

The `stat` read-back is the metadata edition of the chapter's standing rule, and
belongs after any operation whose *point* was a mode or owner. Fold this into the
atomic-swap pattern (stage, `chmod`/`chown` the staged copy, then `mv`) and the
replacement arrives with content and metadata correct in the same instant —
no window, nothing to remember, nothing for the next operator to discover the
hard way.

## Do not edit what you do not own

The ladder's final lesson is about choosing not to climb it. Much of what
administration edits — package-installed configuration, another tool's managed files
— has an owner that will edit it again: the package manager on upgrade, the
provisioning system on its next run, the tool regenerating its own state. Editing
such files puts two writers on one file, and the second writer always wins
eventually. Modern configuration design offers the way out this chapter's systemd
threads have already pointed at: the drop-in directory. `<unit>.d/*.conf`,
`sudoers.d/`, `sshd_config.d/` — the pattern is general: the owned file stays owned,
and local intent lives in a *separate file* the owner promises to include. A drop-in
converts every rung of this chapter into its safest form at once: creating a file is
naturally guarded (it exists or it does not), naturally atomic (stage and rename),
naturally reviewable (the whole local intent in one small file), and removable by
deleting one path — an undo channel requiring no memory of what the file looked like
before, which for an operator with no memory is not a convenience but the whole
point. When a drop-in mechanism exists, the register's edit of choice is not an edit
at all; it is a new file with your name on it, placed where the owner agreed to
look.

## The tools that insist

A last practicality: some tools open an editor as their *interface*, and the ladder
must be threaded through them rather than around them. Each has its non-interactive
door, usually less advertised than the editing form. `crontab -e` has `crontab
<file>` — generate the full table (top rung: whole-file replacement), validate by
listing it back with `crontab -l`, and install it as data; the editor was never
required. `git commit` takes `-m`; the interactive rebase's editor can be replaced
wholesale by setting `GIT_SEQUENCE_EDITOR` to a script that rewrites the todo file
— an editor implemented as a one-shot edit, the chapter's thesis made literal.
`visudo`, whose editor session exists to guarantee syntax checking, splits into its
two halves: `visudo -c -f candidate` runs the *checker* alone against a staged
file, which slots exactly into the validate-then-swap pattern — and better still,
the file being staged belongs in `sudoers.d`, converting the whole operation into
the drop-in form below. The general method when meeting a new insistent tool: read
its manual for the non-interactive door first (`-c` flags, `--file` forms, `EDITOR`
overrides — the environment variable is honored by most, and an `EDITOR` that is
itself a script receives the temp file's path as its argument, making any
scripted transformation into a legal "editor"). The door almost always exists,
because scripts needed it decades before this register's operators did — the same
inheritance chapter 1 traced, paying out one tool at a time.

The chapter's ladder, bottom to top: guard your appends, count before you
substitute, rehearse your diffs, validate then swap, and prefer the drop-in that
makes the whole question moot. Every rung ended in a read-back, because in this
register the file's final state is the only witness to the edit that matters.
What editing has not yet faced is the operation that cannot be read back —
the one that removes, overwrites, or reaches beyond the machine. That is chapter 6,
and it is the chapter this whole book exists to make safe.
