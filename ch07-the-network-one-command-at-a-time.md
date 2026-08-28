# Chapter 7 — The Network, One Command at a Time

*Draft status: author draft, gate-checked; human verification pending. The runnable
listings in this chapter touch only the loopback interface and the local machine's
own state, per the book's listing policy; commands that reach real networks are
labeled fragments.*

## Nothing here stands still

Every substrate this book has read so far — processes, services, files — belonged to
the machine under your hands. The network is different in three ways that bend the
register's technique. It is *shared*: the state you observe includes other parties'
behavior, and your observations are events on their side too — a connection attempt
is a log line somewhere, a repeated probe is a pattern someone's monitoring may act
on, and so even "reads" carry a faint write. It is *layered*: a working connection
rides link, address, route, name resolution, socket, and service, stacked such that
any layer's failure wears the costume of the layers above it — the browser era
taught everyone "DNS error" faces for what were route problems underneath. And it is
*in motion*: caches expire, addresses lease and re-lease, peers restart, so two
observations a minute apart legitimately differ — chapter 3's snapshot caveats,
raised to a power. The register's response to all three is the same discipline in
sharper form: bounded shots, one layer at a time, each shot's answer read before the
next is chosen, and every measurement stamped with when and from where it was taken,
because on a moving substrate an unstamped fact is already a stale one.

The chapter's frame for diagnosis is the oldest one in network operations, adapted
to one-shot economics: **walk the layers, cheapest first, and let each answer
eliminate half the remaining suspects.** A human at a terminal walks it by intuition
and fast iteration. A transcript operator walks it deliberately — the layer sweep is
five or six cheap reads, and dispatching them as one composed batch (their combined
output still small) frequently converts a would-be conversation into a single turn
whose transcript contains the whole differential diagnosis.

## The sweep, and the seam in the path

The canonical layer reads live in the `iproute2` suite: `ip link` for whether the
interface is up, `ip -j addr` for what addresses the machine actually holds, `ip
route get <dest>` for which way packets to a destination would leave, `ss` for what
sockets exist. They are presented here as fragments, and the reason is a seam this
book's own construction exposed. On the authoring machine, `ip` resolves at
`/usr/bin/ip`; on plenty of other systems — including the publisher's gate sandbox,
whose `PATH` is the minimal `/usr/bin:/bin` — the `iproute2` tools sit in `sbin`
directories that minimal paths do not include. Chapter 3 stated the general rule
(*which tools your shot can reach is part of your machine's state*); the network
tools are where it bites hardest in practice, because `sbin` placement reflects an
old assumption that network inspection is an administrator's act. When a network
tool answers 127, the fix is a fuller path or an absolute one — not a different
diagnosis.

```bash fragment
# The one-turn layer sweep (adjust names to the machine; ip may need /usr/sbin).
ip -brief link
ip -brief addr
ip route get 1.1.1.1
resolvectl status | head -n 12
ss -tlnp
```

`-brief` on the first two is the register's friend — one line per interface, fixed
columns, built for exactly this reading. The `route get` verb deserves special
notice: rather than dumping the route table for you to simulate in your head, it
asks the kernel to *run its actual decision* for one destination and report the
result — source address, egress interface, gateway. It is the difference between
reading the law and asking the judge, and it retires a whole class of "but the
table looks right" confusion.

## Sockets from the source

For the socket layer, the register has an option the fragments above do not cover:
go where `ss` itself goes. The kernel publishes socket tables under `/proc/net`,
and a short reader answers the most common question — *what is listening?* —
with no tool dependency at all:

```bash
python3 - <<'PYEOF'
listeners = []
for path in ("/proc/net/tcp", "/proc/net/tcp6"):
    try:
        rows = open(path).read().splitlines()[1:]
    except FileNotFoundError:
        continue
    for row in rows:
        f = row.split()
        if f[3] == "0A":                      # state 0A = LISTEN
            addr, port = f[1].rsplit(":", 1)
            listeners.append((int(port, 16), "v6" if path.endswith("6") else "v4"))
for port, fam in sorted(set(listeners))[:8]:
    print(f"listening {fam} port {port}")
PYEOF
```

```output
listening v4 port 22
listening v6 port 22
listening v4 port 53
listening v6 port 3000
listening v4 port 3001
listening v4 port 4141
listening v4 port 4142
listening v4 port 4180
```

The authoring machine's first eight listeners, decoded from the kernel's own table:
an ssh daemon on 22, a local DNS resolver on 53, and a family of high ports
belonging to the household's services. The format is documented (hex address:port
pairs, hex state codes, `0A` for LISTEN), and the exercise is not a recommendation
to abandon `ss` — which decodes more, faster, with process attribution — but a
demonstration that the socket table is *readable state*, reachable even from seats
too minimal to carry the tool. When `ss` is present, its register-ready spelling is
`ss -tlnp`: TCP listeners, numeric (no reverse-DNS stalls — the `n` matters in
one-shot mode, where a slow resolver turns a socket listing into a hang), with
owning processes where privilege allows. The follow-up that completes most
socket-layer diagnoses is the bind-address column: a service listening on
`127.0.0.1:8080` is alive and *correctly unreachable* from outside — the classic
"it works on the box but not from anywhere else" has this one-line explanation more
often than any firewall does.

## Names: two different truths

Name resolution earns its own section because there are two of it, and conflating
them wastes diagnostic turns. Applications do not, as folklore has it, "ask DNS";
they ask the system's Name Service Switch, which consults `/etc/hosts`, local
resolvers, caches, and only *then* the DNS protocol, per `/etc/nsswitch.conf`. The
one-shot read of that whole stack — the answer applications actually receive — is
`getent`:

```bash
getent hosts localhost
```

```output
::1             localhost
```

Measured, and already a lesson: on the authoring machine, `localhost` resolves
first to the IPv6 loopback — a service bound only to `127.0.0.1` and probed "at
localhost" can fail this machine's probe while being perfectly alive, which is a
resolution fact, not a service fact. The DNS-protocol truth, by contrast, comes
from `dig` (or `resolvectl query`), which speaks to nameservers directly and
bypasses `/etc/hosts` and NSS entirely:

```bash fragment
dig +short example.com          # the protocol's answer, resolver's cache included
dig +short @1.1.1.1 example.com # a specific server's answer, cache and all
resolvectl status               # which resolvers this machine would actually use
```

The diagnostic use of having both: when `getent` and `dig` disagree, the *gap
between them* is the finding — an `/etc/hosts` override someone forgot, a stale
local cache, an NSS module misbehaving — and no amount of further DNS querying
would have found it, because the DNS was never the layer that lied. Chapter 2's
determinism rule also lands here with special force: resolution answers are
cache-and-vantage-dependent, so a transcript that records a name lookup should
record *which* truth it asked and when, or the record will win arguments it should
lose.

## Reading a refusal

Failed connections are not one phenomenon, and the register's four-question routine
pays off richly here, because the three textures of failure implicate different
layers and a probe's error already contains the triage. **Connection refused**
arrives fast and means the packet *completed its journey*: a host answered, and
answered that nothing listens on that port. Route, addressing, and the host itself
are thereby acquitted in the same instant — the suspect list collapses to the
service (dead, or bound elsewhere, per the socket section) and port number. A
refusal is the most informative failure there is, and treating it as generic
"can't connect" discards its gift. **Timeout** is the opposite texture: silence.
Packets left and nothing returned — consistent with a dead host, a black-holing
route, or (most common in practice) a firewall that *drops* rather than rejects,
precisely because silence tells probers the least. A timeout therefore acquits
nobody, and its diagnostic value is only comparative: reachable on port 22 but
timing out on 443 outlines a filter's shape; timing out on everything outlines a
route or host problem. **Connection reset** is the strange third texture — the
conversation began and was slammed shut — and points at the application layer or
at middleboxes terminating what they dislike: a service crashing on this
particular input, a proxy rejecting a protocol, an idle connection reaped. The
one-shot operator reads which texture the probe reported *before* choosing the
next layer to inspect; three textures, three different next shots, and the error
line already chose between them.

## The remote shot

Most real administration in this register eventually crosses to another machine,
and the carrier is `ssh host 'command'` — the venerable one-shot form that chapter
1 counted among the mode's ancestors. Operating it well requires knowing three of
its behaviors precisely. First, **its exit status is the remote command's**,
faithfully carried home — the whole chapter 2 discipline works across the wire
unchanged — with one reserved value: 255 is ssh's *own* failure (could not
connect, could not authenticate), so 255 means "the question never arrived",
the remote cousin of 127's "the question never reached a tool", and statuses
below it mean "the question arrived, and this was its answer". Second, **it must
be disarmed for transcript mode**: `-o BatchMode=yes` forbids every interactive
fallback (password prompts, passphrase dialogs — chapter 1's prompt trap in its
most common network costume) so that a shot which would have hung fails instantly
and legibly instead; `-o ConnectTimeout=5` bounds the attempt. Third, **the
command travels as text through two shells**, and every expansion rule from
chapter 6 applies *twice*: locally when the shot is composed, remotely when the
far shell parses what arrived. The single-quoted form `ssh host 'echo $HOME'`
expands remotely (the intended reading, usually); the double-quoted form expands
*locally* and ships the result — both are legitimate tools, and the accident is
not knowing which one was dispatched. For anything beyond a short command, skip
the quoting puzzle entirely: compose the remote work as a here-doc streamed into
the far shell, where it travels as data —

```bash fragment
ssh -o BatchMode=yes -o ConnectTimeout=5 host bash -s <<'EOF'
set -u
df -P -k | awk 'NR > 1 && $5 + 0 >= 80 {print $6, $5}'
EOF
```

— the quoted delimiter (chapter 5's rule) keeping every `$` remote, and the remote
`bash -s` reading the program from the wire. The pattern composes with everything
this book has built: the shot inside the here-doc is written to the same standards
as any local shot, and its transcript, statuses included, comes home over the same
channel. For moving files rather than commands, `rsync` over ssh inherits chapter
6's rehearsal discipline (`-n` first, always, doubly so with `--delete` pointed at
a machine you cannot see), and for repeated remote shots against one host, ssh's
connection multiplexing (`ControlMaster`/`ControlPersist` in the client config)
collapses the per-shot handshake cost — the round-trip economics of chapter 1,
purchasable with configuration.

## Downloading with proof

One network operation deserves its own doctrine because it ends in execution:
fetching software. The register's rule is the same one chapter 5 applied to
config — *validate, then swap* — with the validator being cryptographic. Download
to a file, verify the file against a published digest, and only then let it near
an interpreter:

```bash fragment
curl -sS --fail --max-time 60 -o installer.sh "https://example.com/installer.sh"
echo "<published-sha256>  installer.sh" | sha256sum -c -
# only on "installer.sh: OK" does the file graduate to execution
```

The popular one-liner this replaces — piping a fetched URL directly into a shell —
is tolerated by interactive humans partly on the theory that they could watch it
run. This register has no such theory available: nothing streams past your eyes,
so the pipe form is all of the risk with none of the (already thin) mitigation,
and the download-verify-inspect sequence costs exactly one extra turn. `sha256sum
-c` speaks chapter 2's dialect — per-file `OK` lines and a nonzero exit on any
mismatch — and where the publisher signs releases, the same station is where the
signature check runs. The habit's quiet second benefit is the artifact itself: the
downloaded file, retained, is evidence — of what was fetched, when, and with what
digest — which is more than any pipe ever leaves behind.

## curl as a measuring instrument

At the top of the stack, the register's universal probe is `curl`, and the
craft is to promote it from "fetch tool" to *instrument* — a probe with a pass/fail
contract and calibrated dials. The demonstration runs against a real HTTP server
the listing itself starts on the loopback, so the whole exchange is local,
deterministic, and disposable:

```bash
w=$(mktemp -d); cd "$w"
printf "pong\n" > ping.txt
python3 -c '
import http.server, threading, pathlib
srv = http.server.HTTPServer(("127.0.0.1", 0), http.server.SimpleHTTPRequestHandler)
pathlib.Path("port").write_text(str(srv.server_address[1]))
threading.Thread(target=srv.serve_forever, daemon=True).start()
import time; time.sleep(4)
' &
sleep 1; port=$(cat port)
curl -sS --fail --max-time 5 \
  -w "http %{http_code}, %{time_total}s total, %{size_download} bytes\n" \
  -o fetched.txt "http://127.0.0.1:$port/ping.txt"
cat fetched.txt
wait
```

```output
http 200, 0.004980s total, 5 bytes
pong
127.0.0.1 - - [27/Aug/2026 22:12:50] "GET /ping.txt HTTP/1.1" 200 -
```

(The last line is the toy server's own access log, arriving on stderr — left in
the transcript deliberately as a two-streams reminder: the probe's *answer* is the
`-w` line and the fetched body, and a parser should have been aimed only at those.)
Each flag on that `curl` is a policy decision worth making consciously. `--fail`
converts HTTP-level failure into exit-status failure — without it, a 500 page
downloads "successfully" and chapter 2's number-first reading is defeated at the
protocol boundary. `--max-time` is the chapter 1 hang-proofing, non-negotiable on
any probe whose far side you do not control. `-sS` silences the progress bar (a
repainter!) while preserving real errors. And `-w` turns the exchange into
*measurements* — beyond the basics shown, curl exposes the full timing anatomy
(`time_namelookup`, `time_connect`, `time_appconnect`, `time_starttransfer`), which
decomposes a slow request into *which layer was slow* in a single shot: resolution
time implicates the previous section, connect time the path, start-transfer time
the far service. One probe, dials read, differential diagnosis included — against
real services, the same spelling with `-o /dev/null` measures without keeping.

Two instrument-handling notes complete the toolkit. A probe that writes nothing and
checks nothing but reachability should say so honestly — `curl -sS --fail -o
/dev/null` plus `-w` is a *health check*; the habit of HEAD requests (`-I`) probes
cheaper but measures a different thing, since servers legitimately treat HEAD
differently. And bound your retries: `--retry 3 --retry-max-time 30` gives
transient-failure tolerance with a ceiling, which in one-shot economics beats both
the brittle single attempt and the unbounded loop an interactive human would have
interrupted by feel.

## The invisible middleman

One network fact lives in the environment rather than on the wire, and it belongs
in this chapter because it falsifies probes: the proxy. The convention — honored
by `curl`, package managers, language runtimes, and most HTTP clients —
is a set of environment variables (`http_proxy`, `https_proxy`, `no_proxy`,
plus uppercase variants some tools prefer) naming an intermediary through which
requests should travel. A machine configured this way makes every environment-
respecting probe measure *the path through the proxy*, while tools that ignore
the convention (lower-level probes, raw socket checks) measure the direct path —
and the two can disagree completely: curl succeeding while the direct route is
firewalled, or curl failing against a dead proxy while the network itself is
fine. The diagnostic consequences compound in exactly the environments this book's
operators inhabit. Chapter 4 showed `sudo` and stripped sessions losing
variables; proxies are the network edition — the interactive human's shell has
the proxy set, the agent harness or cron job does not, and the "network is down
for my script but fine for me" ticket writes itself. The register's discipline:
the proxy variables are part of any connectivity finding — one `env | grep -i
proxy` (bounded, labeled) belongs in the layer sweep on any machine you did not
configure yourself, `no_proxy`'s exemption list read as carefully as the proxies
themselves (a loopback probe that unexpectedly transits a proxy because
`no_proxy` omitted `localhost` is a classic self-inflicted mystery), and a probe
report states which path it measured. From this seat, through this proxy — or
explicitly not through one: vantage, again, with one more clause in it.

## Waiting, without watching

Network work constantly requires waiting — for a restarted service to accept
connections, a DNS change to propagate, a peer to finish coming up — and waiting
is the thing the register cannot natively do: chapter 1 retired `watch` along with
the rest of the repainters. The honest substitute is the *bounded poll*: a loop
with a check, an interval, a maximum, and — the part naive versions omit — a
distinct final answer for exhaustion:

```bash
cd "$(mktemp -d)"
( sleep 2; printf "ready\n" > up.txt ) &
for i in 1 2 3 4 5 6; do
  [ -e up.txt ] && { echo "up after $i checks"; break; }
  sleep 1
done
[ -e up.txt ] || echo "gave up after 6 checks"
wait
```

```output
up after 3 checks
```

The background subshell plays a service that takes two seconds to come up; the
loop finds it on the third check and says so. Substitute the real readiness probe
for the file test — `curl --fail` against a health endpoint, `ss` finding the
listener, `getent` returning the new record — and this is the register's entire
waiting toolkit. The composition rules are the same three costs as ever. The
*budget* (six checks, one second apart) is chosen from knowledge of the thing
awaited, and stated in the transcript — an unbounded poll is a chapter 1 hang
built by hand. The *success line names the elapsed cost* (`after 3 checks`),
which turns the wait itself into a measurement: a service that used to arrive on
check one and now arrives on check five has said something worth recording. And
the *exhaustion line is affirmative* — "gave up after 6 checks" is a finding, the
good-shots-say-none rule from chapter 2, because the poll that times out silently
forces the next operator to distinguish "never came up" from "never checked".
When the wait stretches beyond a turn's patience, the pattern moves up a level:
dispatch the poll as a background job or a transient unit whose *record* you read
next turn — the machine does the waiting, and the transcript does the watching.

## What one shot cannot see

The chapter closes, as chapter 3 did, on honest limits — sharper here, because the
network's failure modes are so often *intermittent*. A one-shot probe samples an
instant; packet loss of one percent, a flapping route, a peer that stalls under
concurrent load — none of these reliably appear in any single sample, and a passing
probe does not acquit them. The register's recourses parallel chapter 3's. Sample
deliberately: `ping -c 20` (bounded count — never bare `ping`, the canonical
never-ending repainter) reports loss and latency spread over twenty samples in one
shot, and its summary line is built for transcripts. Use the accumulators:
`/proc/net/dev` deltas (chapter 3's technique, unchanged) integrate every packet
between two reads, including the ones your probes missed; the interface error and
drop counters in the same file convert "feels flaky" into a number that either
grows or does not. And when the question is genuinely about traffic *content*,
bound the capture the way you bound everything: `tcpdump -c 100` (privileged, a
fragment) takes a hundred packets and stops — a capture with no `-c` on a busy
interface is the volume cost of chapter 1 arriving all at once, and has filled
more than one disk in the folklore.

The deepest limit is vantage. Every read in this chapter observes from one host on
the graph; "the network is down" and "this machine cannot reach it" are
indistinguishable from a single vantage, and distinguishing them requires a second
one — another machine, an external probe service, the peer's own logs. A
transcript-mode operator states its vantage as part of its findings (*from this
host, at this time, the service did not answer within five seconds*) and resists
the sweeping conclusion its evidence cannot carry — the discipline chapter 1
promised, of claims sized exactly to what the shot could see. What that discipline
looks like when a whole piece of work is being handed back — evidence, ledger, and
all — is the business of the final chapter.
