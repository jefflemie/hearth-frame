# The email agent: availability foundation

Layer 0 (the machine), layer 1 (making a fixed Claude quota last),
layer 2 (turning documents into as few tokens as possible) and layer 3
(teaching a local model to do the routine reading). The Hearth write
path and the reply policy are deliberately not here yet -- they are
worthless on a box that goes dark on a Tuesday because something
auto-updated.

**Two constraints are settled and everything here obeys them: the
Claude Pro subscription is the only paid component, and every other
tool is free, open source and runs locally.**

Written in the same spirit as `index.html`: what was measured, what was
assumed, and what is still unverified, kept apart from each other.

---

## The reframe this whole design turns on

**You cannot make Claude highly available. You make the system
indifferent to Claude being unavailable.**

Claude is a network service someone else operates. It will rate-limit,
it will 529, it will have a bad twenty minutes. No amount of care on
the T440 changes that. And on a **fixed subscription quota** it is
stronger than that: the model will be unavailable *because you used it*,
predictably, every week, and the only control you have is how fast you
spend. So availability cannot mean "the model always answers." It has to
mean:

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

**Tailscale, on a Headscale control server.** Not port-forwarding, not
dynamic DNS. It survives IP changes, ISP swaps, and NAT, and it gets you
in when the thing is sick. The clients are BSD-3 but Tailscale's
coordination server is proprietary and hosted -- see the stack section;
**Headscale** is the self-hosted replacement that keeps remote access
inside the FOSS constraint.
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

## Layer 1 -- Pro only: tokens are uptime

**Decided: the Claude Pro subscription is the engine. No API.** Every
other tool in the stack is free and open source, and runs locally.

That settles the question and changes what efficiency *is*. On the API,
a wasted token costs a fraction of a cent. On a subscription it costs
**availability** -- the quota is fixed, pooled across claude.ai, Desktop
and Claude Code, refilled on rolling five-hour windows under a weekly
cap, with no `retry-after` to respect and nothing to buy in the moment.

So the arithmetic inverts:

> Every token spent classifying a newsletter on Monday is a token not
> available for a mortgage letter on Thursday.

Efficiency is no longer frugality. It is the availability mechanism.
Everything in this layer exists to make the quota last.

**And the quota is not the agent's alone.** It is the same pool you draw
on when you sit down to work. An agent that reads mail enthusiastically
all morning can lock *you* out of your own Claude by lunchtime. A
reserve is not a nicety here; it is the difference between a useful
assistant and a hostile roommate.

### Three mechanisms, in order of leverage

**1. Distillation -- Claude teaches, a local model does the routine work.**
The whole of layer 3 below. Largest by a wide margin.

**2. Batching -- amortise the per-invocation overhead.**
Every `claude -p` invocation pays a fixed cost before it sees a single
word of email: system prompt, tool definitions, whatever context the
harness loads. Paying that 200 times a day is the single most
Pro-specific waste available to you. Batch twenty emails into one
invocation and the overhead is divided by twenty.

**3. Invocation hygiene -- strip the harness to the bone.**
Free, immediate, and the first thing to do.

- **`--bare` on every call.** It skips auto-discovery of hooks, skills,
  custom commands, subagents, plugins, MCP servers, auto memory and
  `CLAUDE.md`. Each of those is tokens on every invocation, and a
  classification call needs none of them. It is documented as the
  recommended mode for scripted calls and is slated to become the
  default for `-p`.
- **No tools.** Tool definitions are tokens. Pass no `--allowedTools`
  for a call that only has to return a judgement. The worker does the
  I/O; Claude does the thinking.
- **A JSON schema, not prose.** `--output-format json` with a schema
  puts the answer in `structured_output` and constrains generation to
  it. No preamble, no explanation, no "Here's my analysis:" -- every
  output token is a field you asked for.
- **A fresh session per batch.** Never `--continue`. Accumulated context
  is the silent quota killer: it grows monotonically and you pay for all
  of it on every turn.

### What that compounds to

Illustrative arithmetic, not measurement -- the overhead figure is the
one to replace with your own first:

| | tokens/day |
|---|---|
| Naive: one invocation per email, harness loaded | ~840,000 |
| `--bare`, no tools, schema output | ~500,000 |
| ...batched 20 per invocation | ~270,000 |
| ...plus distillation, ~15% escalation rate | **~42,000** |

Assumes 200 mail/day, ~1.2k tokens of extracted text each, ~3k fixed
overhead per `--bare` invocation. **Roughly twenty-fold**, and the last
line is the one that decides whether this is comfortable or constantly
against the ceiling.

### The budget governor

A fixed quota with no API to query needs a component that no
API-based design would have: something that knows what has been spent
and decides who gets the rest.

- **Meter every call.** `--output-format json` returns `total_cost_usd`
  and a per-model breakdown. On Pro that dollar figure is notional --
  nothing is billed -- but it is a faithful *proportional* signal, which
  is exactly what a governor needs. Log it on every invocation. This log
  is your only instrument; Pro offers no usage endpoint to ask.
- **Track both windows.** Rolling five-hour and weekly. The weekly cap
  is the one that bites, and it bites on a Thursday.
- **Hold a reserve for the human.** A fixed slice of the week that the
  agent may not touch, so your own interactive use is never starved.
- **Admit by priority, not arrival.** When quota is scarce, a first-time
  sender with an invoice outranks a newsletter. Arrival order is the
  wrong order.
- **Degrade, don't stop.** Near exhaustion, fall back to layer 3's local
  classifier alone and defer every escalation to the next window. The
  queue from the top of this document means deferral costs latency and
  nothing else.

One practical note: sustained automated use of a personal subscription
is worth a glance at your plan's terms before you scale it up. Claude
Code on Pro and headless `-p` are both documented and supported; the
volume is the part only you can size.

---

## Layer 2 -- extraction, or: where the tokens actually go

Attachments, not bodies, are the token bomb. A four-page utility bill
sent to the model as a PDF is **~9,000 tokens**; the same bill as
extracted Markdown is under a thousand. For a mailbox that exists to
drive a household ledger, almost every attachment that matters is a
bill, an invoice, a receipt or a statement -- so this layer decides the
bill more than the model choice does.

### Why sending PDFs to the model directly is the expensive path

The Claude API accepts a PDF as a document block, and it is tempting
because it is one line of code. What it does underneath is extract the
text **and rasterise every page to an image**, then charge you for
both. Published guidance is **1,500-3,000 tokens per page** depending
on density. That is a per-page floor you cannot optimise, paid on every
page whether or not it carried information.

Extracting locally to Markdown first is typically an **8-10x
reduction** on a text-and-table document, because you stop paying for
pictures of whitespace.

| 4-page utility bill | tokens | on Haiku 4.5 | on Opus 5 |
|---|---|---|---|
| As a PDF document block | ~9,000 | $0.009 | $0.045 |
| Extracted to Markdown | ~900 | $0.0009 | $0.0045 |
| Matched by a learned template | **0** | **$0** | **$0** |

At 30 attachment-bearing mails a day averaging three pages, that is the
difference between attachments costing ~$6-30/month and costing under a
dollar. It is the difference between the layer-1 estimate holding and
tripling.

### The correction: MarkItDown is the wrong default for *these* documents

MarkItDown is the obvious pick -- Microsoft's, ~15 formats, and genuinely
fast: roughly **12 seconds per 100 pages** against Docling's ~2 minutes
on CPU. For speed and breadth it wins outright.

But it is weak exactly where this mailbox is strong. In published
comparisons Docling reaches ~88% F1 against MarkItDown's ~82% overall,
and on **tables** the gap stops being a gap and becomes a cliff --
MarkItDown's table-structured extraction has been measured as low as
**F1 0.07**, against Docling's TableFormer handling merged and nested
cells essentially perfectly.

Tables are not an edge case here. Tables are the bill. A parser that
scrambles a table does not produce a vaguer ledger entry, it produces a
**confidently wrong number**, silently, in the household accounts. That
is the one failure mode worth spending both tokens and seconds to
avoid.

**Treat those benchmark numbers as a reason to measure, not as truth.**
They come from specific corpora that are not your post. Keep ~20 real
bills from your actual recurring senders as a golden set with the
correct figures written out by hand, and run any parser change against
it. Twenty documents is an afternoon and it is the only evidence that
transfers to this mailbox.

### The tiering

Route by document class. One parser for everything is the mistake.

**Tier 0 -- learned templates. The win that dwarfs the parser choice.**
A household receives the same bills from the same ~20 senders every
month, in the same layout. Parse a given sender's bill correctly once,
store a template keyed on `(sender, layout fingerprint)`, and every
subsequent month extracts by deterministic code: **zero tokens, zero
latency, exact fidelity, and no model in the loop to have an opinion.**
Only a template *miss* -- new sender, changed layout, failed
validation -- escalates. After a few months this should absorb the large
majority of recurring financial mail, and the model's job narrows to
what it is actually good at: the new and the strange.

This is the highest-leverage idea in the whole design and it is easy to
skip, because on day one it does nothing at all.

**Tier 1 -- MarkItDown.** DOCX, PPTX, XLSX, CSV, HTML, and PDFs with a
clean text layer and no tables that matter. Fast, cheap, plenty good.

**Tier 2 -- Docling.** Anything financial, anything table-heavy,
anything where a wrong number reaches Hearth. Slow, and worth it.

**Tier 3 -- the model.** Scans, photographs of receipts, layouts that
defeat both parsers. Send the pages as a document block and pay the
1,500-3,000. Rare by construction, which is what makes it affordable.

### Docling on a Haswell ULV, and why it does not matter

Two minutes per hundred pages is a figure from a modern CPU. On the
T440 -- dual-core ULV, no GPU -- expect meaningfully worse, and check
the RAM headroom before committing, since the layout and table models
are not free. OCR on scanned documents will be slower again.

**None of which matters, because of layer 1.** Nothing is waiting. The
queue already decoupled arrival from processing, so a ninety-second
Docling run on a bank statement blocks precisely nothing -- it is one
row that reaches the model a minute and a half later than it might
have. This is the second dividend of building the queue first: it does
not only absorb model outages, it makes extraction latency free, which
is what lets the accurate-but-slow parser be the default for the
documents that matter.

Keeping extraction local is also the privacy answer. Bank statements
and medical bills get turned into text on your own machine; only the
extracted text crosses the network.

### The rules that keep it cheap

- **Prefer `text/plain` when the email offers it.** Most mail is
  `multipart/alternative` and already carries a plain-text part.
  Reading it costs nothing and needs no parser. MarkItDown on the HTML
  part is the *fallback*, not the path.
- **Extract once, keyed on SHA-256 of the attachment bytes.** Store the
  Markdown in the queue database forever. The agent will revisit
  threads; it must never pay twice for the same PDF, and identical
  attachments forwarded around the family collapse to one extraction.
- **Classify before extracting.** Size, MIME type and sender decide
  whether a PDF is a bill or a 40-page catalogue. Do not run Docling on
  marketing.
- **Hard per-email token ceiling.** No single message may spend more
  than a set budget. A 200-page attachment hits the cap, gets a
  first-pages-only summary, and is flagged for a human rather than
  quietly eating the day. This is an availability control as
  much as a cost one.
- **Strip the body properly before it goes anywhere near the model** --
  quoted reply chains, signatures, legal boilerplate, tracking pixels.
  Routinely 10-50x on its own, and it is pure code.

---

## Layer 3 -- distillation: Claude as teacher, not labourer

The largest efficiency win is not making Claude's calls cheaper. It is
**not making most of them.**

Claude is a superb classifier and a ruinously expensive one to run two
hundred times a day against a fixed quota. But classification is a
learnable task, and you have a perfect teacher already on the box.

### The loop

**Phase 0 -- deterministic.** Dedupe, known-sender rules,
`List-Unsubscribe`, and layer 2's learned templates. Free, and it should
handle the dull majority.

**Phase 1 -- bootstrap.** Claude labels ~300-500 real emails, batched
twenty at a time behind `--bare`. A deliberate, one-time, budgeted spend
-- a weekend of quota to buy months of autonomy.

**Phase 2 -- train the student.** Embed each email and fit a classifier.
On this hardware that is close to free:

- **Model2Vec** (`potion-base-32M`) distills a sentence transformer into
  static token embeddings: **~50x smaller and up to ~500x faster**, at
  roughly **93%** of `all-MiniLM-L6-v2`'s quality (~52.1 vs 56.3 MTEB).
  Distillation itself runs in about **30 seconds on CPU**.
- Or **`all-MiniLM-L6-v2`** directly -- under 10ms per email on CPU,
  MTEB 56.3 -- if you want the accuracy and can spare the milliseconds.
- Then plain **logistic regression** over the embeddings. Not a
  neural net. Boring, fast, interpretable, and it tells you its
  confidence, which is the part that matters.

**Note what this avoids.** No local *generative* LLM. Nothing to quantise,
no tokens/sec to agonise over on a Haswell ULV, no 4GB of weights
competing with Docling for the 12GB ceiling. Embeddings plus a linear
model is microseconds per email and tens of megabytes resident. The
T440 is comfortably the right machine for this, which it would not be
for a local 7B.

**Phase 3 -- steady state.** The student classifies everything. Only
low-confidence cases escalate, batched, to Claude.

**Phase 4 -- active learning.** Every escalated answer becomes a new
training label. Retrain nightly. The escalation rate decays as the
student learns your mail, so **the system gets cheaper the longer it
runs** -- the right shape for a fixed quota.

### The confidence threshold is the throttle

This is the control loop the whole design has been building toward. One
knob connects the quota to the behaviour:

- Quota tight -> **raise** the threshold. Fewer escalations, more local
  autonomy, slightly more error.
- Quota plentiful -> **lower** it. More escalations, faster learning,
  better labels.

The budget governor turns that knob. Nothing else needs to change for
the system to ride out a heavy week.

### The rule that stops efficiency eating correctness

A logistic regression can be confident and wrong. It has no idea what a
mortgage is. So confidence governs **triage only**:

> **A hard allowlist of high-stakes categories always escalates to
> Claude, whatever the student's confidence.** Anything touching money,
> anything proposing a Hearth write, anything from a first-time sender,
> anything legal or medical.

The student decides where the boring mail goes. It never decides
anything that would be expensive to get wrong.

### On "it must actually read every email"

This deserves a straight answer rather than a reassuring one, because
distillation is in genuine tension with it.

Under this design **every email is read by something** -- embedded,
classified, never filed unexamined -- but the thing reading most of them
is a linear model, not Claude. If the requirement means *Claude reads
every email*, distillation breaks it, and on a Pro quota the requirement
is simply unaffordable at 200/day.

The version worth committing to:

> Every email is read by something competent, **everything consequential
> is read by Claude**, and nothing is ever acted on unexamined.

Two things keep that honest. The high-stakes allowlist above, and a
**nightly sweep**: one batched, cheap invocation in which Claude reviews
the day's local decisions in summary -- every category assignment, the
near-threshold calls in full -- and flags what the student got wrong.
That catches drift, feeds the training set, and costs one invocation a
day rather than two hundred.

---

## The stack, and what "closed environment" really buys

Everything below is free and open source and runs on the box. Licences
noted because a couple of them matter.

| Job | Tool | Licence |
|---|---|---|
| Host, declarative config | NixOS | MIT |
| Container runtime | Podman | Apache-2.0 |
| Queue and all state | SQLite | public domain |
| Mail parsing | Python `email` stdlib | PSF |
| Quoted-chain / signature stripping | talon, email-reply-parser | Apache-2.0, MIT |
| HTML and Office to Markdown | MarkItDown | MIT |
| Table-faithful documents | Docling | MIT |
| PDF text layer | pdfplumber / pdfminer.six | MIT |
| OCR | Tesseract or RapidOCR | Apache-2.0 |
| Embeddings | Model2Vec / sentence-transformers | MIT / Apache-2.0 |
| Classifier | scikit-learn | BSD-3 |

**Two traps worth naming now.**

**PyMuPDF is AGPL-3.0.** It is the fastest PDF library and the obvious
reach, and its licence is viral in a way the rest of this list is not.
pdfplumber over pdfminer.six is the permissive path. Decide deliberately
rather than by `pip install`.

**Tailscale's client is BSD-3; its coordination server is not.** Layer 0
recommends Tailscale, and under the old assumptions that was fine. Under
"works in a closed environment" it is a hosted proprietary dependency in
the middle of your remote access. **Headscale** (BSD-3) is the
self-hosted control server that closes that gap and speaks to the stock
clients. If remote access must survive a vendor, run Headscale.

**And the honest boundary:** this environment is not closed. Claude is a
network service and so is Gmail. What the FOSS constraint actually buys
is that *everything else* is -- documents are parsed on your own
machine, classification runs locally, the queue and all state are files
you own, and no third service ever sees a bank statement. Nothing in the
stack can be discontinued, repriced, or rate-limited out from under you
except the two you chose deliberately.

Two consequences follow. Model weights (Docling's layout models,
the embedding model, Tesseract's language data) need one download, after
which they should be **pinned and vendored** like any other dependency
-- a closed box cannot fetch them later. And the **Gmail API is the one
service-shaped dependency in the ingest path**: if provider independence
ever matters more than convenience, IMAP via `imapclient` (BSD-3) is the
same design against any mailbox.

---

## Still open

1. **Hearth's write path.** The app is deployed *execute as me, access
   only myself*, so the agent cannot simply POST to `/exec`. The
   cleanest shape is probably an **intake queue** -- the agent appends
   proposed rows, Hearth's own script drains them under its own rules --
   so the ledger keeps a single writer and the agent stays outside the
   trust boundary. Needs Hearth's internals to decide.
2. **Autonomy.** Assumed for now: **drafts, never sends.** Labels, filing
   and Hearth *proposals* are autonomous; anything leaving the house or
   changing the ledger waits for a human. Easy to loosen later, very
   hard to walk back.
3. **Volume and accounts.** Every number in layer 1 is arithmetic over an
   assumed 200/day across one mailbox. Real volume, and whether this is
   one Gmail account or several, changes the escalation budget directly.
4. **The taxonomy.** What categories the student is learning, and which
   of them land on the always-escalate allowlist, is the next thing to
   design -- and it is the thing that decides whether layer 3 works.
5. **Scanned vs native PDFs.** If recurring senders email real PDFs,
   templates plus Docling cover nearly everything and OCR never runs. If
   receipts get photographed, Tesseract gets real traffic on a Haswell
   ULV and the extraction budget needs redoing.
