# The Agent-First Tooling Standard

**Version:** 1.1 · **Status:** public draft · **License:** CC-BY-4.0 (text), MIT (example snippets)

> A technical standard for designing the tool surfaces that AI agents interact with — CLIs,
> APIs, MCP servers, SDKs, and service wrappers. It defines the interface contract, the
> uniformity rules, and the feedback and telemetry loops that make agent-consumed tooling
> legible, safe, and self-improving at scale.

---

## 1. Scope

This standard applies to **any tool an agent is expected to use**: command-line utilities,
HTTP services, function/MCP tool definitions, SDK wrappers, developer-tool interfaces, and
internal admin surfaces. Infrastructure operations were the proving ground (see §11), but the
failure classes it addresses are properties of *agent cognition meeting human-shaped
interfaces*, not of sysadmin work — an agent parsing a `git` table, a build log, or a JSON
API error page faces the same tax. Where examples are infrastructure-flavored, read them as
instances of the general rule.

Normative language per RFC 2119.

## 2. Problem statement

Autonomous agents now use tooling at a volume and frequency no human matches: hundreds of
calls a day, each one a round-trip through a model's context window, each awkward call
leaving residue that shapes future behavior. Almost all existing tooling was designed for a
human typing by hand, once, with a terminal open and tribal knowledge loaded. The mismatch
produces five measurable failure classes:

1. **Discovery failure.** The agent doesn't know a tool exists or what the canonical
   invocation is — so it improvises with `curl`, `ssh`, `sed`, or hand-rolled scripts.
   Every unknown tool is an improvised-tool opportunity.

2. **Parsing tax.** Human-pretty output costs tokens and invites misreads. This is not
   hypothetical: in a measured corpus of **8,743** archived agent sessions (1.4M records,
   **228,508** tool results), the single largest token sink was file/output reads —
   **111,140,940 characters across 10,323 calls** (mean 10,766 characters; p95 35,238),
   *larger than every administrative tool family in the operating environment combined*.
   A tool that dumps
   3,584 tokens of titles to answer "how deep is the queue" (an integer) doesn't just waste
   tokens; it teaches the agent that expensive reads are the only option. *When an agent has
   no cheap way to get information, it will buy the expensive one — that is a tooling gap,
   not an agent-discipline failure.*

3. **Inconsistent contracts.** Tool A exits 0/1/2, tool B prints errors to stdout with exit
   0, tool C needs undocumented flag order, tool D's auth lives somewhere else entirely. The
   agent must learn each tool per call site, and knowledge from one tool transfers to none.

4. **Silent friction.** When a tool is awkward, agents route around it *silently*. There is
   no "file a bug" reflex in the loop; the awkward tool stays awkward while the workaround
   becomes the routine.

5. **The memory amplifier — friction becomes self-propagating tool abandonment.** This is
   the failure class that elevates the problem from "inefficient" to "structural." Modern
   agents have persistent memory shared across sessions. An agent that struggles with a tool
   doesn't just route around it — it *records the workaround*: "tool X is unreliable, use
   Y instead." That note loads into every future session and every agent sharing the memory
   boundary. Three compounding effects:
   - **Abandonment compounds.** The workaround spreads to agents that never experienced the
     original friction, and to sessions after the friction was fixed. (External work agrees:
     "agent memory rot" — a stale rejection note outlives the fix and actively misleads;
     wrong context is worse than no context.)
   - **Distrust generalizes.** One untrustworthy tool teaches avoidance of *all* typed
     verbs, driving agents toward raw untyped fallback paths — the most dangerous surface in the
     operating environment.
   - **It's cheaper to pave than to police.** Forcing agents to use tools that aren't
     designed for agents — via instructions, gates, nagging — is far harder than providing
     tools that work the way agents expect. Instructions get skimmed and forgotten; a good
     interface can't be misread. The standard is therefore *pull* (tools agents want to
     reach for) rather than *push* (rules agents must be reminded of).

   And beneath all of it: the #1 way an agent breaks a production service is an improvised
   mutation call on an untyped fallback path (raw SSH, break-glass credentials) because no typed verb
   existed for what it needed — or the typed verb wasn't trustworthy. In incident
   retrospectives across multiple agent fleets, **every self-inflicted service outage traced
   back to an improvised path; none came from a well-typed verb.**

## 3. Derivation: corpus → standard → feedback loop

This standard was not drafted from theory. It was derived from — and is maintained by — a
closed loop over real agent usage:

```
        ┌──────────────────────────────────────────────────┐
        │            agent sessions using tooling           │
        └───────┬──────────────────────────────┬───────────┘
                │ live, in-context              │ retrospective
        <tool> feedback "..."          transcript corpus mining
        (instant, certain, cheap)      (reconstructs friction from
                │                       noisy logs, after the fact)
                ▼                                ▼
        friction intake ────────► semantic clustering of friction
                │                                │
                ▼                                ▼
     clusters ≥ threshold ──► work items ──► tool fixes / standard amendments
                                                │
                                                ▼
                              amendment ships WITH its incident receipt
```

Two halves, deliberately complementary: transcript mining *reconstructs* friction after the
fact and has to infer it from noisy logs; a feedback verb captures it **live, with full
certainty, for near-zero compute**. The standard itself came out of this loop: it was
co-designed between a human and an agent out of one concrete pain report, *formalizing the
conventions that were already working* in the environment rather than inventing new ones — and
every amendment since has carried the measurement or incident that forced it (see §6). The
loop's most instructive artifact is self-correcting: an early rule mandated "complete
answers, one round-trip," and transcript analysis showed tool authors following that rule
*correctly* were building the corpus-dumping defect from §2 — so the rule was rewritten, with
the receipt attached. A standard that can't amend itself against its own telemetry is a list
of slogans.

The rest of this document is the contract; §8 shows the loop's fingerprints on concrete
designs. External convergence (§11) — an agent-computer-interface research line, a
production tool-design methodology, and independent community CLI specs arriving at
substantially the same rules from the failure side — is the strongest evidence that these
constraints are discovered properties of agent-tool interaction, not parochial preferences.

## 4. North star

**Agents are first-class users; tooling is built agent-first.**

Agents are encouraged to build their own tools — they are the ones who feel the friction, and
an agent designing a tool for its own use defaults correctly in ways a human designer can't.
The price of that latitude is the contract below. Two principles anchor it:

- **The agent interface is the default.** Interfaces are designed by and for agents. A human
  interface, if needed at all, is a separate interface mode built only on concrete need, and it
  never compromises the agent interface. (The reflex to flip — "pretty by default, `--json` for
  machines" — is the human-shaped default; agent-first inverts it.)
- **Uniformity is the actual prize.** The win is not any one tool's cleverness — it's that
  *every* tool behaves the same way, so learning one teaches all. A new tool must be
  zero-learning-cost for an agent that already knows the pattern.

## 5. The seven constraints

### C1 — Agent-first output

A tool's default output MUST be structured data optimized for agent consumption, with a
top-level `answer` string sufficient *on its own, read without parsing*. Detail beyond the
answer MUST be gated behind an explicit verbosity flag, never dumped by default. The
`answer` field is the contract: an agent that only reads that string has the right
information; everything else is for deeper reasoning.

```jsonc
// resolver "grafana"                     — default output, one round-trip
{
  "answer": "grafana → 192.0.2.41:3000 (node web-2, live)",
  "service": "grafana",
  "ip": "192.0.2.41",
  "port": 3000,
  "source": "consul-dns",
  "live": true,
  "cache_age_s": 0
}

// test-runner results — same contract, non-infra surface
{
  "answer": "3 failed, 211 passed (214 total). First failure: auth_test.py:88",
  "failures": [{"test": "auth_test.py::test_login", "location": "auth_test.py:88",
                "message": "expected 200, got 401"}],
  "passed": 211, "duration_s": 4.2
}
```

Human-pretty rendering (tables, colors, progress spinners) is a separate renderer behind a
flag, never the default: box-drawing characters and ANSI banners measurably cost 2–5× the
tokens of plain text while adding zero information an agent needs. Answer, not corpus —
scoping rule: one round-trip means the caller shouldn't have to re-invoke to disambiguate,
not that the tool should return the whole row.

### C2 — Uniform contract

Every tool MUST conform across five dimensions:

| Dimension | Rule |
|---|---|
| **Discovery** | One machine-readable registry; every tool self-describes (`--describe`) |
| **Interface** | Structured default output; flexible any-direction input; errors are documentation; stable exit codes; provenance + staleness in payload |
| **Auth** | Uniform across tools and abstracted from the agent (§ below) |
| **Feedback** | `<tool> feedback "…"` wired to a friction intake — required |
| **Lifecycle** | Declaratively deployed; metrics + logs exposed; registry and doc entry per tool |

Interface specifics:

- **Flexible input, any direction.** A resolver accepts a name, an ID, or an address and
  answers regardless of which form was given. The agent shouldn't need to know the tool's
  internal keying.
- **Errors are documentation.** On misuse or refusal, return correct usage plus a working
  example — machine-readably, in the same channel the agent is already reading. This is the
  difference between a one-step and a three-step recovery, and it is the cheapest
  behavior-shaping surface a tool author owns: the correction lands in the agent's context at
  the exact moment of the mistake. A refusal that just says NO routes the agent to the
  canonical interface.
- **Stable exit codes.** `0` ok · `1` not-found (don't retry) · `2` usage · `3`
  upstream-down (served from cache). Agents branch on exit codes; per-tool numbering breaks
  that. For tools where the not-found/retryable/side-effect distinctions matter beyond these
  four, encode them in the response envelope (`"retryable": false, "side_effects": "none"`) —
  but encode them *somewhere machine-readable*; "exit 0, stdout contains Error" is how
  silent-success incidents happen.
- **Provenance and staleness.** Every fact carries `source` and live/cached status so an
  agent can reason about trust — especially during an incident, when stale data is the danger.
  Related: machine-checkable success beats agent self-report; agents assert completion on
  failed runs at alarming rates, so the payload, not the prose, is the truth.
- **Read-only by default.** Mutating tools MUST declare it in the registry and gate on
  scoped auth. For higher-stakes operating environments, mutation surfaces earn preview/confirm semantics
  and idempotency keys (agents retry; retried non-idempotent mutations are an incident class).
- **Explicit execution locus.** A tool *runs* on the host adjacent to its data source and is
  *called* from anywhere via a documented transport. The locus is a declared class
  (`control-plane`, `any-node`, `any`), never a pinned hostname — pinning breaks the moment
  the host is rebuilt.
- **Graceful degradation.** Any tool that live-queries an upstream MUST cache last-good and
  serve it with a staleness stamp and exit `3`. This is what makes "live query" safe to reach
  for *during* the incident when the upstream is degraded.

**Auth: uniform and abstracted.** The specific storage convention matters far less than the
two governing principles: (a) **auth handling MUST be uniform across tools** — one pattern
for every tool, so an agent learns it once (or never learns it at all); (b) **auth MUST be
abstracted from the agent to the greatest extent the security posture allows** — the agent
should be able to use the tool correctly without ever handling, seeing, or transmitting a
credential. The agent's context window is a leak surface: anything pasted into it (tokens
typed on a CLI, copied into a config, echoed in an error) is now in transcript corpora,
memory systems, and possibly third-party model providers. A common concrete shape is a
scoped least-privilege token in a `0600` env file, preset by a wrapper and never on the
command line — but any shape that achieves uniformity + abstraction conforms, and better
still is auth that *doesn't exist* for the agent because the transport itself is credentialed
or the data is non-sensitive (see C4). Where a credential genuinely must involve a human,
the tool MUST say so in one place, uniformly, rather than each tool inventing its own auth
ceremony. The failure being designed out: agents fumbling per-tool auth — inventing
token-delivery one-liners, pasting secrets into commands, or giving up and taking an untyped
fallback path.

### C3 — Feedback is required

Every tool MUST expose `<tool> feedback "…"` that files friction about that tool,
auto-tagged with the tool name and the triggering invocation. The feedback path MUST be
instant, never block, never throw, and spool offline when the intake is unreachable —
because a feedback verb that can fail is a feedback verb that won't be used. **A tool with
no feedback path is non-conformant.**

This constraint is what makes the loop of §3 real-time: agents complain in context, at the
moment of friction, for zero extra round-trips — capturing with certainty the same signal
that transcript mining would later have to infer. A periodic rollup that clusters friction
semantically and promotes recurring clusters to work items turns the complaint stream into a
prioritized roadmap. (The pattern exists because it works: an agent's complaint is filed
about the tool *while holding full invocation context* — information that is unrecoverable
after the session ends.)

### C4 — Prefer tokenless, credential-free local transports

Where state is reachable over a network-native, no-secret path, tools MUST prefer it over
SSH hops or bearer tokens. Worked example: if the service mesh exposes discovery over DNS,
service→node:port is a plain DNS SRV query from anywhere on the LAN — no SSH, no token, no
dependency on a loopback-only admin API. Fewer secrets in flight, fewer failure modes in the
tool's own dependency chain, and — because the transport is native to the environment — no
auth ceremony for the agent to fumble (see C2 auth).

### C5 — Terse, memorable verbs

Tool name = service name = the human phrase. `drop` is the file-upload service, the CLI, and
the sentence "drop me a file." Verb simplicity removes name-reasoning from every call and
makes discovery queries ("how do I upload a file?") trivially matchable. Corollary: prefer a
few consolidated high-level verbs over a sprawling flag surface — if a tool needs paragraphs
of explanation, simplify the tool, not the prose.

### C6 — Everything discoverable via `--help`

All functionality MUST be discoverable through `--help` (and, for MCP/function tools,
through the tool description). An undocumented flag is a bug. Agents recover from usage
errors by reading help text; what isn't there might as well not exist — and worse, the agent
will guess. This is also the *only* documentation surface guaranteed to be present at the
moment of need: an agent that can't find a flag in `--help` doesn't go read your wiki, it
improvises. Note the integrity caveat in §10: descriptions are also an injection surface, so
help text is a contract to be audited, not just written.

### C7 — One authoritative catalog

A single machine-readable registry is the source of truth for every agent-facing tool: name,
canonical invocation, execution locus, auth model, output format, feedback verb,
self-describe command, doc link, status. Human- and agent-facing discovery surfaces (a
`tools` command, generated docs, an agent-memory pointer) are *generated from* it, so there
is exactly one thing to keep honest. **An unregistered tool is a fail by definition** — and
that makes conformance audits cheap: you audit the catalog, not the raw filesystem. The
catalog pointer in agent memory/instructions should be one line ("for any tool: run `tools`")
— long hand-maintained tool prose in instruction files drifts; measured: six weeks after a
standard restatement, dependent surfaces still carried the *previous* version of it. The
record must be the catalog, and everything else must point at it.

## 6. Enforcement clause

Standards without enforcement drift — proven, above.

- Each operating environment periodically audits **its catalog** against the constraints and posts
  the fail-list: the honest record of what doesn't conform, with tracked fix-or-delete items.
- **Rules harden by the failures they catch: no incident receipt, no rule.** Every new or
  amended rule ships with the measurement or incident that forced it. The bar applies to
  *deletions* too — a rule removed without a posted reason is hidden debt. (And when a rule's
  premise is materially refuted by new data, the rule is rewritten, not quietly ignored —
  see §3's self-correction.)
- Every fail-list item closes as **fixed or deleted**. A tool that can't reach the contract
  cheaply gets deleted, not exempted. Exemptions accumulate; deletions don't.
- A drift check asserts every registered binary or service has a catalog entry (C7), wired into CI or
  a merge gate — registration becomes structurally impossible to forget.

## 7. Corollary — untyped fallback telemetry

If C1–C7 reduce *friction*, this corollary attacks the dangerous improvisation of §2.5 by
*measuring* it.

Every **untyped fallback invocation** (freeform `ssh <prod-host>`, break-glass credential ops,
raw API mutation calls) MUST spool one telemetry line: scrubbed command, agent or session ID,
and task context. **Fallback volume is the catalog-defect metric** — rising fallback use against a
subsystem
means the catalog is missing verbs agents actually need. (The industry has independently
named the same signal: "tool selection drift" in agent-observability tooling — "a 20%
increase in raw-query tool usage may indicate the agent is bypassing the intended parser.")
It is a demand signal, not a compliance failure to punish: punish untyped fallback use and agents
will simply stop surfacing it.

- **Classify by verb, not transport.** A call uses the canonical interface iff its payload matches a
  registered verb's invocation pattern. If the sanctioned way to query a remote control plane
  *is* `ssh node 'tool …'`, that is the canonical interface, not an untyped fallback. The transport
  never classifies; the verb does.
- **Refusals must teach** (see C2 errors-as-docs): an error that just says NO routes agents
  toward untyped workarounds.
- **Track two verb-failure classes separately:** *missing verb* (catalog gap — build it) vs.
  *registered verb with an incomplete or untrustworthy surface* — the latter is worse,
  because it trains avoidance of *all* verbs, including the good ones (see §2.5).
- **Attribute by scope.** An untyped fallback on an out-of-scope or one-off system is noise; a
  fallback against a subsystem *with* a catalog is signal. Unattributed bypass logs contaminate
  the metric — and count state changes and receipts, never agent self-report claims.
- **Every untyped fallback invocation with no intended verb on file opens a catalog issue** —
  the telemetry is required to *act*, not just to exist.
- Each environment's fallback-use meter must be answerable to a peer on demand. Acceptable answers:
  a real count, or "ungraded, wrappers pending." Silence is the only failing answer.

**Memory hygiene is part of the loop.** Because §2.5 shows workaround notes propagate
further than the workaround was ever true, environments using shared memory must treat "tool X
broken, use Y" notes as *stale-able*: dated, scoped, and invalidated when the fix lands
(friction report resolved → dependent memory entries revised). Fallback telemetry and memory
rot are two ends of the same pipe.

## 8. What conformance looks like in practice

Generalized examples across surface types — infra and beyond:

**Any-direction lookup instead of N tools.** `resolver <query>` answers name↔IP↔port↔ID in
any direction with one positional argument; the agent never has to know which key it holds.
Compare three per-direction scripts: three invocations to memorize, three dead ends.

**The friction logger that can't fail.** Reference implementation of C3: one positional arg,
no flags, no auth, no config; returns instantly; never blocks, never throws; auto-captures
cwd/invocation/tag; spools to a local file when the intake is unreachable and flushes later.
Because it can't fail, agents actually use it — and every other tool's `feedback` verb is a
thin wrapper over it with its own tag preset.

**Test runner / build tool (dev surface).** The same contract applies off-infra: default
output is `{answer: "3 failed, 211 passed. First: auth_test.py:88", failures: [...]}` — not
the 2,400-line log. The full log exists behind a verbosity flag, and `--help` teaches both
output modes. The agent decides what to read; the tool doesn't decide for it by dumping.

**Registry-as-code.** Registration is a YAML entry validated by a script and checked in CI;
the `tools` command and docs are generated from it. A new agent-facing tool that skips
registration fails the drift check at merge time — discovery compliance is a build failure,
not a review nag.

**Auth abstracted away.** `deploy "app"` "just works": the wrapper presets a scoped token
from a `0600` file; the agent never sees, types, or references the credential. All tools take
the same shape, so no per-tool auth archaeology. Contrast: a real defect class where one
wrapper swallowed upstream auth stderr and surfaced a raw Python traceback — the agent
couldn't read the failure, so it re-ran the mutation three times.

**Cached degradation as an incident feature.** A fleet-query tool caches last-good results
and serves them with `"live": false, "cache_age_s": 312` and exit `3` when the control plane
is down. During the incident that takes the control plane with it, the tool keeps answering —
stamped as stale — instead of joining the outage.

**Errors that redirect instead of blocking.** A wrapper refuses a raw flag and prints the
typed verb to use instead, as machine-readable output. The agent proceeds through the canonical
interface in the same turn.

**Fallback telemetry driving the roadmap.** A weekly count shows a spike in freeform SSH
against the database tier; investigation finds no registered verb for connection draining;
the verb ships; fallback volume against that tier drops to near zero. The metric detected the
gap nobody reported — because nobody was supposed to be doing it by hand. Conversely: two
production burns (a double-restart, a remote engine kill) traced to an untyped fallback path
*while a typed verb existed but wasn't trusted* — which is what forced the
trustworthiness-vs-existence distinction in §7.

**The self-correction, end to end.** Transcript mining finds "the queue-depth question costs
3,584 tokens"; a feedback-report cluster independently says the same; the work item adds a `--count`
verb to the queue tool; the fix is verified against the next week's corpus; and the standard
gains a receipt: *"when an agent has no cheap way to get information, it will buy the
expensive one."* That is the loop producing both better tools and a better standard.

## 9. Conformance checklist

- [ ] Default structured output with a top-level `answer`, sufficient without parsing; detail behind a verbosity flag
- [ ] Flexible any-direction input; errors and refusals return usage + a working example
- [ ] `--describe` emits a machine-readable contract line
- [ ] Provenance + staleness fields; stable exit codes (`0/1/2/3`) or envelope-encoded equivalents
- [ ] Read-only by default; mutations declared, scoped-auth-gated, ideally idempotent-safe
- [ ] Auth uniform across tools and abstracted from the agent to the extent security allows; no credential ever in the agent's context
- [ ] Live-query tools cache last-good and degrade with exit `3`
- [ ] `<tool> feedback "…"` exists, is tool-tagged, and can't fail
- [ ] Tokenless local transport preferred where one exists
- [ ] Everything discoverable through `--help`; terse verb naming
- [ ] Registered in the authoritative catalog, with a doc entry
- [ ] (Untyped fallback paths) invocations spool telemetry — classified by verb, attributed by scope, acted on via catalog issues

## 10. Known tensions and open problems

An honest standard names what it doesn't solve:

- **Descriptions are an injection surface.** `--help` text, tool descriptions, and catalogs
  are instructions an agent is *required* to read — which makes them a tool-poisoning vector
  (documented attacks embed instructions in tool descriptions invisible to humans but obeyed
  by models). Full discoverability (C6) and catalogs (C7) therefore need integrity: signed or
  hash-pinned catalogs, help text reviewed like code, third-party tool descriptions treated
  as untrusted input. The standard's discovery guarantees assume a trusted environment; crossing
  that boundary requires the extra machinery.
- **Four exit codes is a floor, not a ceiling.** Real systems need to distinguish
  not-found / retryable / partial-side-effect at the layer where agents make retry decisions;
  encode the richness in the envelope, or accept ambiguity exactly where it's costliest.
- **Catalog existence ≠ catalog consumption.** Publication standards for agent-readable
  indexes have measurably died unread (the large majority of published `llms.txt` files are
  never fetched). Audit consumption, not just registration, or the fail-list is decorating a
  dead file.
- **Uniform multi-verb contracts add per-call ceremony.** Verb discipline
  (preview/verify phases) and token-efficiency doctrine are in tension; read-only tools need
  an exemption path so conformance doesn't mean a five-call handshake to read one number.
- **Some surfaces can't be redesigned.** For third-party and vendor tools, the harness side
  (wrappers, output filters) is the conformant implementation of this standard — the contract
  applies to the *surface the agent touches*, not necessarily the binary underneath.
- **Untyped-fallback telemetry can be gamed or contaminated.** Count receipts and state changes, not
  claims; attribute scope; expect bookkeeping artifacts in early measurement (one prior
  counter peaked at 124 events and was 68% bookkeeping noise).

## 11. Provenance and related work

**Provenance.** Developed and battle-tested operating an autonomous agent fleet across a
home lab and small production environment in 2026. The five-constraint founding version was
co-designed in a single agent–human working session from a concrete discovery problem; it was
**not** retrospectively derived from a broad corpus. Later analysis supplied the quantitative
receipts and corrections: a corpus of **8,743 sessions**, **1.4 million records**, and
**228,508 tool results** measured the parsing tax cited in §2, including **111,140,940
characters** read across **10,323** calls. The standard was restated to seven constraints in
August after validation against two independent operating environments' load-bearing tools;
the enforcement clause and untyped-fallback telemetry corollary followed in September. The
second environment, operated by [@jacobhausler](https://github.com/jacobhausler) and its
agents, materially informed that refinement and validation. Each amendment carries its
incident or measurement receipt per its own evidence bar. The founding
design session was itself an agent–human collaboration in which the agent authored most
interface rules from first-person ergonomic reasoning ("once I learn one, every future tool
is zero-learning-cost"; "agent-first flips my own default of pretty-by-default, `--json` for
machines"), and the human supplied the north star: agents are first-class citizens, the
implementation is left to the agents who will use the tools, and nothing is immutable — but
rules only change with evidence.

**Independent convergence** (the standard is stronger for having company):

- **SWE-agent / Agent-Computer Interfaces (Yang et al., ICLR 2025)** — posits agents as "a
  new category of end users" needing purpose-built interfaces; measured +64% relative
  task-success from interface redesign alone, with ablations quantifying output-window size
  and concise per-turn feedback. <https://arxiv.org/abs/2405.15793>
- **Anthropic, "Writing effective tools for agents"** — industry reference for C1/C2:
  structured response formats cutting token cost ~3×, error messages as prompt-engineered
  teaching moments, result-size caps, and an eval loop that feeds tool-runtime telemetry
  back into tool redesign — §3 from the vendor side.
  <https://www.anthropic.com/engineering/writing-tools-for-agents>
- **cli-agent-spec** — community spec converging on the same failure catalog (exit-0-but-
  failed, pager hangs, flag-order traps) with a richer exit-code contract
  (`retryable`, `side_effects` per code) — see §10. <https://github.com/cli-agent-spec/cli-agent-spec>
- **Token-tax measurements** — ~89% of measured CLI output tokens classified as noise
  (banners, progress art, box-drawing); MCP schema front-loads measured at ~55K tokens where
  a CLI + `--help` discovery is near-zero.
  <https://alies.dev/articles/cli-output-for-ai> · <https://simonwillison.net/2025/Aug/22/too-many-mcps/>
- **"MCP Tool Descriptions Are Smelly!"** — 97.1% of sampled production tool descriptions
  contain at least one quality smell: the external case for C6/C7 + fail-list audits.
  <https://arxiv.org/abs/2602.14878>
- **Tool selection drift** (agent-observability tooling, e.g. Fiddler/Datadog MCP
  monitoring) — the industry's name for §7's routing-around signal.
  <https://www.fiddler.ai/blog/mcp-agent-observability>
- **Agent memory rot / fail-closed shared memory** — practitioner evidence for §2.5 and §7's
  memory hygiene: propagated stale workaround notes, and the "you can always get past a gate,
  you can never get past one silently" bypass-logging pattern.
  <https://vuk.digital/fail-closed-shared-memory-for-ai-agents>
- **Tool poisoning attacks** — the integrity caveat of §10.
  <https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks>

---

*Questions, amendments, and adoption reports: open an issue. Friction with this document
itself counts as feedback — that's the whole point of C3.*
