# The email agent: availability foundation

Layer 0 (the machine) and layer 1 (keeping Claude answering). The
classifier's taxonomy, the Hearth write path, and the reply policy are
deliberately not here yet -- they are worthless on a box that goes dark
on a Tuesday because something auto-updated.

Written in the same spirit as `index.html`: what was measured, what was
assumed, and what is still unverified, kept apart from each other.

---

## The reframe this whole design turns on

**You cannot make Claude highly available. You make the system
indifferent to Claude being unavailable.**

Claude is a network service someone else operates. It will rate-limit,
it will 529, it will have a bad twenty minutes. No amount of care on
the T440 changes that. So availability cannot mean "the model always
answers." It has to mean:

> No email is ever lost, no email is ever acted on twice, and every
> email is eventually read -- however long the model was away.

That is a property of the *queue*, not of the model. Get the queue
right and a three-hour outage costs you three hours of latency and
nothing else. Get it wrong and a thirty-second blip loses mail.

Everything below serves that sentence.

---

## Layer 0 -- the machine

### The T440 is a better choice than it looks

It has Lenovo's Power Bridge: an internal battery *plus* the removable
one. A laptop server is the only server that ships with its own UPS,
already wired, already monitored, with a battery the OS can read the
charge of. For a box whose job is to never miss an email, riding out a
brownout without a filesystem-corrupting hard stop is most of the
battle.

Haswell ULV is ample. The agent is I/O-bound on network calls to Gmail
and to Anthropic; it does not think locally.

### OS: NixOS, with the agent in a container

The requirement is "extremely resilient to updates, reboots". The
instinct is to reach for auto-updating. That instinct is wrong. **For
this machine, auto-update is the threat, not the feature.** High
availability here means *never surprised*, not *always latest*.

NixOS gives exactly that:

- Nothing changes unless `flake.lock` changes. The machine cannot drift
  under you overnight. There is no unattended-upgrades, no
  `dnf-automatic`, nothing to fire at 03:00 and take the mailbox with
  it.
- Every `nixos-rebuild switch` writes a **new generation** and leaves
  the old one in the bootloader. A bad change is one reboot from being
  undone -- by you over SSH, or by you holding the power button if SSH
  is what you broke.
- `nixos-rebuild build-vm` boots the entire machine config in a VM on
  your laptop. You test the real config, not an approximation, before
  it touches the T440.
- The whole machine -- units, timers, firewall, secrets wiring -- lives
  in git next to this repo. If the T440 dies, a new box is one
  `nixos-install` from being the same machine.

The cost is honest and worth naming: NixOS is not FHS, and Claude Code
plus Node native modules assume FHS. Fighting that with `nix-ld`
wrappers is a recurring tax and a recurring source of 3am breakage.

**So don't fight it.** Host is NixOS. The agent and Claude Code run in
a Podman container -- ordinary Debian inside, declared from the NixOS
config via `virtualisation.oci-containers`. Immutable, rollback-able
host; boring, FHS-normal payload. Each layer does what it is good at.

*Alternative, lower learning curve:* Fedora IoT or CoreOS -- rpm-ostree
atomic updates with automatic rollback on failed boot, everything in
Podman quadlets. Genuinely fine. You give up the "whole machine in one
git repo" property, which is the thing most worth having here.

### The settings that actually decide whether it comes back

These are the ones that matter. Most "my headless laptop died" stories
are one of these being unset.

**Never sleep, ever.** Closing the lid must be a no-op:

```nix
services.logind.lidSwitch = "ignore";
services.logind.lidSwitchExternalPower = "ignore";
services.logind.powerKey = "ignore";
systemd.targets = {
  sleep.enable = false; suspend.enable = false;
  hibernate.enable = false; hybrid-sleep.enable = false;
};
```

**Hardware watchdog.** The failure that SSH cannot rescue is a wedged
kernel. Haswell chipsets carry the Intel TCO watchdog (`iTCO_wdt`);
systemd will pet it and the chipset will reset the box if systemd stops
petting:

```nix
systemd.watchdog.runtimeTime = "30s";
systemd.watchdog.rebootTime  = "10m";
```

This is the single highest-value line in the config. It converts "dead
until Jeff gets home" into "back in ninety seconds."

**Power-on after AC returns.** *Unverified for this model -- check the
BIOS yourself.* On ThinkPads the setting is usually called **Power On
with AC Attach** under Config -> Power. Desktop-class wording ("After
Power Loss", "AC Power Recovery") does not always appear on the T-series
and I could not confirm it for the T440 specifically. If it is absent,
the internal battery covers short outages and a long one needs a human
-- know which of those you are living with rather than assuming.

**Note the interaction, because it is counter-intuitive:** the internal
battery *defeats* the usual remote-reboot trick. A smart plug on the
mains will not power-cycle this machine; it will just run on battery and
smirk. The watchdog is not a nice-to-have here, it is your only
out-of-band reset.

**Filesystem.** The real reboot risk is corruption on an unclean stop.
Btrfs on a decent SSD, with snapshots of the agent's state directory.
NixOS generations roll back *code*; they do nothing for the queue
database, and the queue is the part you cannot recreate.

### Getting in, from anywhere, forever

**Tailscale.** Not port-forwarding, not dynamic DNS. It survives IP
changes, ISP swaps, and NAT, and it gets you in when the thing is sick.
Enable Tailscale SSH so there is no public sshd at all, and set the node
as an exit-node-less always-on service unit.

Keep a local account with a password and a working console anyway. The
day Tailscale itself is the broken thing, "headless" becomes "carry a
monitor to it," and that should be possible rather than merely
theoretical.

**Network failover.** Ethernet primary, WiFi as an automatic secondary
via NetworkManager connection priorities. If the mailbox genuinely must
survive the house losing internet, a **USB LTE dongle** is the third
leg -- specifically USB, not a mini-PCIe WWAN card, because ThinkPads
of this vintage enforce a WWAN whitelist in BIOS and an unlisted card
is a boot-time error rather than a device.

---

## Layer 1 -- keeping the reading going

### The uncomfortable part: a Pro subscription is the wrong engine

The plan as stated is a Claude Pro account living on the box, doing the
reading. Two properties of Pro make it the *least* available option
available to you:

1. **Usage is pooled across every surface.** claude.ai, Claude Desktop
   and Claude Code all draw on the same limit. Your own interactive
   Claude use and the mailbox compete for one budget. A heavy afternoon
   in a chat window is silently deducted from the agent's evening.
2. **There is a weekly cap, on top of the rolling five-hour windows.**
   A heavy Monday and Tuesday can leave the mailbox unread on Thursday.
   There is no retry-after to respect, no burst credit to buy, nothing
   to do but wait for the reset.

Subscription limits are designed around a human at a keyboard who stops
for lunch. A process that wakes for every inbound message is not that
shape. You would be building a high-availability machine around a
component with a weekly lockout.

**The API does not have this problem.** Rate limits are per-minute,
per-org, they return `429` with `retry-after`, and a backoff loop
absorbs them in seconds rather than days.

So, the split I'd actually build:

> **The API is the engine. The Pro subscription is your console.**

The always-on reader runs on the Anthropic API with a key of its own and
a budget of its own. The Pro account still lives on the T440 -- it is
what *you* SSH into to interrogate the agent, dig through the queue,
and change its rules by talking to it. That is exactly the shape Pro is
good at, and it can no longer take the mailbox down with it.

### What it costs, so the tradeoff is a number

At **200 inbound/day**, with the pipeline below (deterministic prefilter,
Haiku 4.5 triage, escalation to Opus 5 only where judgment is needed,
stable prompt prefix cached):

| | per email | per day | per month |
|---|---|---|---|
| Haiku 4.5 triage (~120/day reach it) | ~$0.0025 | ~$0.30 | **~$9** |
| Opus 5 escalation (~12/day) | ~$0.06 | ~$0.72 | **~$22** |
| | | | **~$30/mo** |

Published rates: Haiku 4.5 $1/$5 per MTok, Opus 5 $5/$25 per MTok.
Assumes a cached prompt prefix reading at roughly a tenth of input
price, ~1.2k tokens of extracted text per mail, ~200 tokens of JSON out.
Re-baseline it against your real volume before trusting it.

Roughly a Pro subscription's worth of money, for something that cannot
be locked out. And the Batch API halves the cost of anything not
latency-sensitive -- the nightly digest and any re-read pass should go
through it.

### Efficiency: where the wins actually are

"Super efficient" and "actually reads every email" are not in tension,
but only if you are precise about what reading means.

**The prefilter routes; it never discards.** Nothing is filed, archived
or dropped without a model having looked at it. What the deterministic
layer decides is *which* model at *what* effort -- not whether. That
distinction is the whole promise, and it is the first thing that will
erode under cost pressure if it isn't written down.

The four levers, in order of size:

1. **Extract before you send.** A marketing HTML email is 50k tokens on
   the wire and 400 tokens of content. Strip to text, drop quoted reply
   chains, drop signatures and legal boilerplate, cap attachments.
   Routinely a 10-50x reduction, and it costs nothing but code.
2. **Cache the stable prefix.** Taxonomy, household rules, Hearth's
   schema, the action vocabulary -- all identical on every call. Put
   them first and mark the breakpoint; only the email varies after it.
   This is the largest single lever and it is purely a matter of
   ordering the prompt correctly from day one.
3. **Micro-batch for cache locality.** The default cache TTL is short.
   Mail arrives in bursts; processing on a ~2-minute tick instead of
   per-message means a burst shares one warm cache instead of paying
   the write each time. A two-minute delay on email is not a delay.
4. **Two tiers.** Haiku 4.5 decides and handles the obvious. It
   escalates to Opus 5 when the mail needs real judgment, touches money,
   or proposes a Hearth write. Most mail never needs the expensive
   model; the few that do are exactly the ones worth paying for.

### The queue is the availability story

```
Gmail  ->  ingest  ->  [ SQLite, WAL ]  ->  worker  ->  effects
           (cursor)      durable queue      (model)     (idempotent)
```

- **Ingest** polls `users.history.list` against a stored `historyId`
  cursor every 30-60s. Incremental, cheap, and it cannot skip: the
  cursor only advances after the rows are committed. Start with polling;
  Pub/Sub **pull** subscriptions are the upgrade path when a minute of
  latency starts to matter, and pull works behind NAT with no inbound
  port.
- **The queue is the contract.** Ingest never calls the model. It writes
  rows and returns. If the model is unavailable for three hours, rows
  accumulate and drain afterwards. Nothing is lost because nothing
  depended on the model being up at the moment mail arrived.
- **Effects are idempotent**, keyed on `(message-id, action-hash)`,
  written *before* the call and confirmed after. A crash mid-action
  replays safely; it cannot double-label, double-file, or double-write
  to Hearth.
- **Circuit breaker.** On sustained model failure, trip to a degraded
  mode that still does the safe deterministic things -- label by known
  sender, file known vendors -- and defers everything requiring
  judgment. Degraded mode **never sends email and never writes to
  Hearth.** It only ever makes the pile smaller and better-sorted.
- **Backlog is the health metric.** Not uptime. "Oldest unprocessed
  message age" is the number that tells you whether the thing is doing
  its job, and the only one worth paging on.

---

## Still open

Not blocking layer 0, but these shape everything after it:

1. **Engine: API or Pro?** The recommendation above is the API for the
   reader, Pro as your console. It is a real cost decision and it is
   yours.
2. **Hearth's write path.** The app is deployed *execute as me, access
   only myself*, so the agent cannot simply POST to `/exec`. The
   cleanest shape is probably an **intake queue** -- the agent appends
   proposed rows, Hearth's own script drains them under its own rules --
   so the ledger keeps a single writer and the agent stays outside the
   trust boundary. Needs Hearth's internals to decide.
3. **Autonomy.** Assumed for now: **drafts, never sends.** Labels, filing
   and Hearth *proposals* are autonomous; anything leaving the house or
   changing the ledger waits for a human. Easy to loosen later, very
   hard to walk back.
4. **Volume and accounts.** The cost model needs a real number, and
   whether this is one Gmail account or several changes the ingest
   design.
