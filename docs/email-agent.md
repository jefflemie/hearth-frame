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

### 8 GB, and it cannot be expanded

**Measured constraint: 8 GB, no upgrade path.** This is the tightest
budget in the design and it decides more than any other number here. It
is not a reason to change machines; it is a reason to be explicit about
who is allowed to be resident at once.

| | resident |
|---|---|
| NixOS headless, sshd, mesh client | ~0.5 GB |
| Worker, SQLite, page cache | ~0.3 GB |
| Model2Vec + scikit-learn (layer 3) | ~0.2 GB |
| **Exactly one heavy stage at a time** | **up to ~3 GB** |

That last row is the whole discipline. The heavy stages are Claude Code
(Node, ~0.5-1.5 GB while running) and Docling (~2.3 GB standard,
~2.8 GB with Tesseract). Either fits. **Both at once does not.**

- **One heavy slot, enforced.** A single-slot semaphore in the worker,
  so extraction and a `claude -p` invocation can never overlap. This
  stops being hygiene and becomes architecture at 8 GB.
- **Short-lived subprocesses, never daemons.** Docling holds ~1.5 GB per
  converter instance for as long as it lives. Spawn per document, exit,
  return the memory. Same for Claude Code -- `-p` runs and exits, and
  nothing is left resident between batches.
- **`MemoryMax=` on every unit.** An extraction killed for exceeding its
  cap is a queue row to retry. An extraction that takes the whole
  machine down is a dead mailbox until someone notices. Cap it and
  choose which of those you get.
- **zram, not disk swap.** Compressed swap in RAM
  (`zramSwap.enable = true`) buys real headroom on 8 GB with no SSD
  wear. A small disk swapfile at low swappiness behind it, as a
  backstop for the rare spike.
- **earlyoom or systemd-oomd.** The kernel OOM killer picks badly and
  late. Choose the victim yourself, before the box goes unresponsive.
- **Never build on the box.** `nixos-rebuild` is memory-hungry.
  Build on your laptop and push the closure with
  `nixos-rebuild --target-host`. The T440 only ever receives a finished
  system, which suits both the RAM and the rollback story.

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

**Measured, 21 Sep 2026: ~201 threads in seven days -- about 30 a day,
~40 messages.** The earlier drafts assumed 200/day and were wrong by
roughly sevenfold. This is a well-kept mailbox, not a firehose: the
inbox holds 125 threads in total.

That changes the conclusion from *tight* to *comfortable*:

| | tokens/day |
|---|---|
| Naive: one invocation per message, harness loaded | ~170,000 |
| `--bare`, no tools, schema output | ~100,000 |
| ...batched | ~55,000 |
| ...plus archetypes and distillation | **~8,000** |

In steady state that is **two or three `claude -p` invocations a day** --
one or two batches of escalations plus the nightly sweep. Against a Pro
quota that is not a constraint worth fearing. The efficiency work is
still what buys the headroom; it just buys considerably more of it than
the first draft projected, and leaves the reserve for your own
interactive use genuinely large.

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

## The unit of learning is the archetype, not the sender

An earlier draft keyed extractors on `(sender, layout)`. **That is
wrong, and it is the kind of wrong that looks fine until it is in
production.**

"Citi" is not a thing. Citi is a per-transaction alert, a statement
notice, a payment confirmation, a credit card offer, a balance transfer
promotion, a rewards summary and a fraud alert -- seven pipelines
wearing one From address. Any design that says "the Citi extractor" has
already failed. The same is true of every bank, utility and merchant in
the mailbox.

This is a solved problem with a literature. B2C mail is generated by
filling user data into templates, so messages from the same mass-sender
*script* share an HTML skeleton even when every value differs.
Clustering on that skeleton is called **structural clustering** or
**template induction**, and the measured result is the point:

> Strict structural matching reaches **85-90% precision and recall**,
> converging after a short learning cycle -- **a 10-20% improvement over
> the sender-based method.**

So the unit of everything downstream is the **archetype**: one sender's
one message shape. Discovered, not enumerated.

### How an archetype is discovered

Strip the variable content -- amounts, dates, names, account digits,
tracking URLs, anything that changes between sends -- and hash what
remains: the HTML tag structure and the static boilerplate. Two Citi
transaction alerts produce an identical skeleton. A Citi statement
notice produces a different one. Nobody has to know in advance that Citi
has seven shapes; the clustering finds them, and finds the eighth when
it appears.

It is hashing and string work. No model, no weights, negligible memory
-- which matters at 8 GB.

### Why this is the opposite of brittle

The objection to deterministic rules is that they mis-fire silently.
The fix is not to abandon determinism but to **earn it and then guard
it**, in three tiers of decreasing confidence:

| | match | then |
|---|---|---|
| **A** | exact skeleton hash | extract deterministically, validate |
| **B** | near match -- skeleton drifted | extract, validate *harder*; on pass, absorb the variant into the cluster |
| **C** | no match | **Claude**, always |

Determinism applies only *after* confident identification, and every
deterministic path is followed by the arithmetic validation gate. A rule
that fires on an unrecognised message is the failure mode worth fearing
-- and tier C is what makes that impossible. Nothing falls through to a
default.

When Citi redesigns its alert, the skeleton changes, tier A misses,
tier C escalates, Claude identifies it and repairs the extractor, and
the new cluster registers itself. **The system self-heals, and its
failure mode is an escalation rather than a wrong number.**

### The archetype decides the whole pipeline, not just the parser

This is why the unit matters so much. It is not that different
archetypes need different *extractors* -- they need entirely different
*handling*:

| Citi archetype | volume | extract | Hearth | Claude, once learned |
|---|---|---|---|---|
| Transaction alert | very high | body fields | ledger row | never |
| Statement notice | monthly | PDF attachment | reconcile | only on drift |
| Payment confirmation | monthly | body fields | ledger row | never |
| Card offer / balance transfer | medium | none | no | never |
| Rewards summary | monthly | none | optional | never |
| **Fraud / security alert** | **rare** | none | no | **always** |

Read the volume column against the Claude column. **The highest-volume
archetype is the one that should cost nothing** -- per-transaction
alerts are tiny, near-identical, and the main thing driving Hearth.
Once that one archetype is learned, the single largest source of mail in
the house becomes free, forever. That is where this design pays for
itself on a Pro quota.

### Two guards that fall out of this for free

**Rarity is a signal, not noise.** A fraud alert is the rarest Citi
archetype and the one that matters most. Any system that learns from
frequency will learn it worst. So: **a rare archetype from a financial
sender always escalates**, whatever its confidence. "We have only seen
this twice" must never become "low priority."

**A novel archetype from a known sender is a security event.** A message
claiming to be Citi whose skeleton matches none of Citi's known
archetypes is one of two things: a redesign, or a phish wearing the
brand. The architecture detects this for nothing, and the response is
the same either way -- check SPF, DKIM and DMARC alignment, escalate to
Claude, and **never auto-act**. For an agent with a path into the
household ledger, this is not a nice extra.

### The split this implies

> **Machine-generated mail is the volume. Human mail is the judgment.**
> Archetypes should drive the first to nearly zero cost, so the quota
> can go almost entirely to the second.

Structural clustering works because a script wrote the message. It does
nothing for a real person writing to you, who has no archetype and needs
no extractor -- that mail goes to layer 3's classifier and to Claude.
Getting the machine half genuinely free is what makes the human half
affordable.

---

## What the real mailbox says

Surveyed 21 Sep 2026 against the live account. Four findings, two of
which overturn assumptions above. **Specifics -- institutions, balances,
card digits, family detail -- are deliberately not recorded here; see
the note at the end of this section.**

### 1. The taxonomy already exists, and it is already applied

The mailbox carries a hand-built label tree -- a bulk bucket, a notices
bucket, a review bucket, a needs-reply bucket, and a money subtree split
into alerts, transactions, actions, checks and other -- applied across
roughly **240 messages**, alongside a dozen topical labels for people,
house, receipts, tax and therapy.

This was listed as an open question ("what categories is the student
learning?"). It is answered. Layer 3's Phase 1 no longer needs a weekend
of quota to bootstrap labels from nothing.

**But do not simply ingest them.** Spot-checking shows real
inconsistency: two near-identical card alerts from the same issuer
carry different labels, one carries none at all, and a rewards offer is
filed under transactions. The existing tree is a **strong prior and a
statement of intent, not ground truth.** Phase 1 becomes *Claude audits
and repairs the existing labels* -- far cheaper than labelling from
scratch, and it produces a cleaner training set than either the human or
the model would alone.

### 2. Statements are often not PDFs at all

Layer 2 assumed statement data lives in an attachment. For the
highest-volume card issuer here it does not: the "statement available"
email carries **account, due date, minimum payment and statement
balance as text in the body**. No attachment, no pdfplumber, no Docling,
no OCR.

This is the cheapest possible path and it was hiding in plain sight.
**Check the body before reaching for the attachment** -- for some
archetypes the PDF is redundant, and the tier-0 extractor is a handful
of regexes over plain text. The document pipeline is for fewer messages
than the previous draft assumed.

### 3. The archetype argument, confirmed in the wild

One issuer's alerts sit under two sender addresses spanning at least
five archetypes -- transaction alert, payment received, statement ready,
credit-limit change, annual privacy notice -- plus a separate marketing
address. Another issuer splits statements and marketing across
*different subdomains*, with six distinct archetypes sharing the single
marketing address.

And the two largest issuers demand **opposite extraction strategies**:

- One puts **merchant and amount in the subject line** -- the body is
  nearly empty. Extraction is a subject-line regex.
- The other sends an **identical subject for every alert**, with
  merchant, amount and card in the body. The subject is worthless as a
  key; only the body carries signal.

Two banks, two archetypes with the same *meaning* and no shared
extraction logic whatsoever. Sender-keyed rules would have produced one
extractor for each and got both wrong.

### 4. It is a shared household mailbox

Mail arrives for two people -- medical, marketing and event mail
addressed to one, card and retirement mail to the other -- in one
account. **Every archetype needs a `who`**, which is the same dimension
Hearth already models in its `?who=` parameter. Routing, priority and
whether something even reaches a digest all depend on it.

### 5. Hearth already emails itself a daily digest

A dated summary arrives each morning carrying the pay period, overdue
bills with ages, amounts and accounts. **The output channel already
exists**, and Hearth already knows what it is owed. This reframes the
open question about the write path: the agent may not need to write to
Hearth so much as to feed the thing that already produces that digest.

### On what is not written down here

The survey saw account balances, card digits, a home address and phone
number, employer, medical providers and family names. **None of it is in
this repository**, in keeping with the precedent `index.html` already
sets: the design notes record shapes and never values. The concrete
inventory -- which institutions, which archetypes, which fields -- lives
outside git until you say otherwise. If this repository is private, or
you would rather it were in, say so and it goes in.

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
is the one failure mode the whole document path exists to prevent.

**Keep that diagnosis; the prescription below is not the obvious one.**
An earlier draft concluded "so make Docling the default," and 8 GB
rules that out. The tiering that follows reaches the same accuracy by a
different route -- a cheap deterministic parser per vendor, with an
arithmetic check that catches it when it is wrong -- and costs a
fraction of the memory and the quota.

**Treat those benchmark numbers as a reason to measure, not as truth.**
They come from specific corpora that are not your post, and the tier-0
path below is not in any of them. Keep ~20 real
bills from your actual recurring senders as a golden set with the
correct figures written out by hand, and run any parser change against
it. Twenty documents is an afternoon and it is the only evidence that
transfers to this mailbox.

### The tiering, rebuilt for 8 GB

The previous draft made Docling the default for anything financial. **At
8 GB that is wrong**, and the fix turns out to be better on every axis,
not merely cheaper.

**Tier 0 -- a per-archetype extractor that Claude writes once.**
Per the section above, the key is the archetype, never the sender.
Published comparisons land exactly on this case -- maintaining
*per-template configurations over a stable set of senders* gets the
**highest accuracy of any open-source coordinate tool** out of
pdfplumber, which is pure Python over pdfminer with **no ML and no model
weights at all**.

The usual objection is that per-template rules are laborious to write
and break on drift. That objection dissolves here, because you have
Claude on the box:

> **Claude's job is not to read the document. It is to write the
> extractor for that archetype, once.**

Fifty lines of pdfplumber, authored in a single invocation, then run for
free on every future instance of that archetype. Claude returns only when the
validation gate below trips. This is the same distillation idea as layer
3 -- scarce judgement spent on teaching, not on labour -- applied to
documents instead of to classification, and it is near-zero on both RAM
and tokens.

**Tier 1 -- MarkItDown.** DOCX, PPTX, XLSX, CSV, HTML, plain text-layer
PDFs. ~100 MB, fast, fine.

**Tier 2 -- Docling, as the exception.** New vendors, unknown layouts,
and tables the cheap path cannot validate. Standard pipeline only
(~2.3 GB), in the single heavy slot, as a subprocess that exits.
**If OCR is needed, Tesseract (~2.8 GB) -- never EasyOCR (~4 GB), and
never the code/formula model (~15.5 GB), which does not fit and never
will.** Its output is not just an answer; it is the worked example
Claude uses to write that vendor's tier-0 extractor, so each Docling run
should be the last one for that archetype.

**Tier 3 -- Claude reads the pages.** Scans and layouts that defeat
everything else. On a Pro quota this is genuinely expensive now, which
is exactly why it sits at the bottom.

### The validation gate, which is what makes the cheap path safe

Trying the cheap parser first is only safe if you can tell when it was
wrong. For financial documents you can, for free:

> **A bill carries its own checksum.** Line items sum to a stated total.
> There is a date. The amounts parse as currency. The account number
> matches the sender on file.

Run that arithmetic after every tier-0 and tier-1 extraction. It costs
nothing, it needs no model, and it catches the failure that actually
matters -- a mangled table producing a confidently wrong number. On a
failed check, escalate a tier and, if Docling resolves it, have Claude
repair the vendor's extractor.

This is the single most valuable check in the document path. It is also
the answer to layout drift: drift shows up as a sum that no longer
balances, not as a silent wrong entry in the ledger.

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

**Phase 0 -- structural.** Dedupe, `List-Unsubscribe`, and above all
**archetype matching**. For machine-generated mail with a matched
archetype there is nothing for the classifier to decide: *the archetype
is the classification*, and it already names the extractor, the Hearth
action and the priority. Free, and it should account for the large
majority of arriving volume.

This narrows the student's job considerably, and usefully. It is not
learning to recognise Citi transaction alerts -- structural clustering
does that exactly, for nothing. It is learning to route **what is
left**: human mail, unmatched machine mail, and the first few instances
of an archetype before its cluster is established. A smaller, harder,
more valuable training set.

**Phase 1 -- bootstrap.** Claude labels ~300-500 real emails, batched
twenty at a time behind `--bare`. A deliberate, one-time, budgeted spend
-- a weekend of quota to buy months of autonomy.

**Phase 2 -- train the student.** Embed each email and fit a classifier.
On this hardware that is close to free:

- **Model2Vec** (`potion-base-32M`) distills a sentence transformer into
  static token embeddings: **~50x smaller and up to ~500x faster**, at
  roughly **93%** of `all-MiniLM-L6-v2`'s quality (~52.1 vs 56.3 MTEB).
  Distillation itself runs in about **30 seconds on CPU**.
- **8 GB settles this choice.** Model2Vec infers from static
  embeddings with numpy alone. `all-MiniLM-L6-v2` via
  sentence-transformers pulls in PyTorch, which costs the better part of
  a gigabyte resident before it embeds anything -- a gigabyte that has
  to come out of the single heavy slot. Take the ~7% quality difference
  and keep the RAM.
- Then plain **logistic regression** over the embeddings. Not a
  neural net. Boring, fast, interpretable, and it tells you its
  confidence, which is the part that matters.

**Note what this avoids.** No local *generative* LLM. Nothing to
quantise, no tokens/sec to agonise over on a Haswell ULV, and no weights
competing with Docling for an 8 GB ceiling that has no upgrade path.
Embeddings plus a linear model is microseconds per email and **tens of
megabytes** resident -- small enough to stay loaded permanently rather
than fighting for the heavy slot. A local 7B was never viable here; this
is why it does not need to be.

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
| **Per-vendor extraction (tier 0)** | **pdfplumber / pdfminer.six** | **MIT** |
| Hard layouts only (tier 2, ~2.3 GB) | Docling | MIT |
| OCR, when unavoidable (~2.8 GB) | Tesseract | Apache-2.0 |
| Embeddings | Model2Vec / sentence-transformers | MIT / Apache-2.0 |
| Classifier | scikit-learn | BSD-3 |

**Note the order.** pdfplumber does the routine work because it has no
model weights and no PyTorch; Docling is the escalation, not the
default. On 8 GB that ordering is forced, and per the comparisons it is
also the more accurate one for a stable supplier base.

**Three traps worth naming now.**

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

**Docling's optional models do not all fit.** The standard pipeline is
~2.3 GB and Tesseract OCR ~2.8 GB, both workable in the single heavy
slot. EasyOCR at ~4 GB is not worth the risk, and the code/formula model
at ~15.5 GB is roughly twice the machine. Pin the pipeline
configuration explicitly rather than accepting whatever a future default
enables.

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
4. **The archetype inventory.** Begun -- see the survey above -- and
   still the critical path. The heavy senders are identified and their
   archetypes sketched; what remains is skeleton-hashing a few months of
   history to measure how stable each one is and how often it drifts.
5. **The action vocabulary.** Given an archetype, what may the agent
   *do*? The table in the archetype section sketches it for Citi --
   ledger row, reconcile, file, escalate -- but the real list, and which
   entries are irreversible, is a decision rather than a discovery.
6. **Scanned vs native PDFs.** If recurring senders email real PDFs,
   templates plus Docling cover nearly everything and OCR never runs. If
   receipts get photographed, Tesseract gets real traffic on a Haswell
   ULV and the extraction budget needs redoing.
