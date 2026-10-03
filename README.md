# Agent-First Tooling Tenets

**Version:** 1.2 · **Status:** public draft · **License:** CC-BY-4.0 (text), MIT (example snippets)

Design tenets for the tool surfaces AI agents use: CLIs, HTTP services, MCP and function tool definitions, SDK wrappers, and internal admin interfaces.

They were derived by studying how agents use our tooling — a corpus of archived session transcripts, incident retrospectives, and a live friction-report stream from the agents themselves — and by working backwards from the errors and wasted turns we found there. They are opinionated, not prescriptive. This is not a protocol, a certification, or an industry standard, and we are not asking anyone to conform to it. Our observation is that the more of these tenets a tool surface satisfies, the less friction agents have using it.

## 1. Where this came from

Two operating environments, 2026: a home lab plus small production environment (this repo's owner), and a second environment operated by [@jacobhausler](https://github.com/jacobhausler) and its agents.

- A founding five-rule version was co-designed in a single agent-human working session, from one concrete discovery failure. It was not derived from a broad corpus.
- Later analysis supplied measurements and corrections. A corpus of **8,743 archived agent sessions** (**1.4M records**, **228,508 tool results**) measured the output-cost problem described in §2.2.
- The rules were restated as seven tenets in August after validation against load-bearing tools in both environments. The maintenance policy (§6) and the untyped-fallback telemetry practice (§7) followed in September, informed by the second environment.
- One early rule was rewritten after our own data contradicted it: it mandated "complete answers, one round-trip," and transcript analysis showed tool authors following it correctly were building the corpus-dumping defect from §2.2. That is the clearest argument we have for the maintenance policy in §6.

Infrastructure operations were where this was worked out, but the failure classes are properties of agent cognition meeting human-shaped interfaces, not of sysadmin work. An agent parsing a `git` table, a build log, or a JSON API error page pays the same cost. Where examples are infrastructure-flavored, read them as instances of the general rule.

**Scope of the evidence.** The corpus is one environment's transcripts; the incident retrospectives cover both environments and are few. We make no claims about agent behavior in general. The measurements we report are ours, the mechanisms we infer are labeled as inferences, and where an outside study is the evidence it is linked.

## 2. Failure classes we observed

Agents use tooling at a volume humans do not: many calls per day, each one a round-trip through a model's context window, with the results persisting into memory and shaping later behavior. Most tooling was designed for a human typing by hand, once, with a terminal open and tribal knowledge loaded. We cataloged five recurring failure classes.

### 2.1 Discovery failure

The agent doesn't know a tool exists or what the canonical invocation is, so it improvises with `curl`, `ssh`, `sed`, or hand-rolled scripts. In our sessions, improvised calls against subsystems that already had tooling are common enough that we treat every unknown tool as a likely source of a workaround.

### 2.2 Parsing tax

Human-pretty output costs tokens and invites misreads. Measured in our corpus: file/output reads were the largest single token sink — **111,140,940 characters across 10,323 calls** (mean 10,766 characters, p95 35,238), larger in our environment than every administrative tool family combined.

One case from the same corpus: a queue tool answered "how deep is the queue" (an integer) by returning 3,584 tokens of titles.

We read the mechanism this way: when an agent has no cheap way to get a fact, it buys the expensive one, and repeated expensive reads teach it that expensive reads are the only option available. We are inferring the teaching effect from the pattern of calls; the token figures above are the measurement. The actionable part is that the missing cheap verb is ours to fix, not the agent's discipline problem.

### 2.3 Inconsistent contracts

Tool A exits 0/1/2, tool B prints errors to stdout with exit 0, tool C requires undocumented flag order, tool D authenticates somewhere else entirely. The agent learns each tool per call site, and what it learns transfers to no other tool.

### 2.4 Silent friction

When a tool is awkward, agents route around it without reporting it. There is no bug-filing reflex in the loop, so the tool stays awkward while the workaround becomes the routine. This is the reason for T3 — the fix is a feedback path cheap enough to use in the moment.

### 2.5 Friction propagates through shared memory

Agents with persistent memory do not just route around a bad tool, they record the workaround: "tool X is unreliable, use Y instead." That note then loads into every later session and every agent inside the same memory boundary. Three effects we have seen:

- **Abandonment outlives the fix.** The note spreads to agents that never hit the original friction and to sessions after the tool was repaired. Outside practitioners describe the same failure under "agent memory rot" (§11), including that a wrong memory is worse than no memory.
- **Avoidance generalizes.** One unreliable tool teaches avoidance of typed verbs as a class, pushing agents toward raw untyped fallback paths, which is where our worst incidents are.
- **Paving beats policing.** Making agents use tools that are not built for agents — instructions, gates, reminders — has been harder for us than giving them tools that behave as expected. Instruction text gets skimmed; an interface that is hard to misread does not need to be remembered. For this reason the tenets are written as pull (tools worth reaching for), not push (rules to be reminded of).

In the incident retrospectives we reviewed across both environments, every self-inflicted service outage we examined traced back to an improvised mutation on an untyped fallback path (raw SSH, break-glass credentials, direct API calls) — either because no typed verb existed for what the agent needed, or because the typed verb was not trusted. None came from a well-typed verb. That is a small number of retrospectives over two environments: enough that we treat fallback volume as a leading indicator (§7), not enough to rank causes of production breakage anywhere else.

## 3. Derivation and maintenance loop

These tenets came out of, and are maintained by, a loop over real usage:

```
        ┌──────────────────────────────────────────────────┐
        │            agent sessions using tooling           │
        └───────┬──────────────────────────────┬───────────┘
                │ live, in-context              │ retrospective
        <tool> feedback "..."          transcript corpus mining
        (instant, cheap, full context)  (reconstructs friction from
                │                       noisy logs, after the fact)
                ▼                                ▼
        friction intake ────────► semantic clustering of friction
                │                                │
                ▼                                ▼
     clusters ≥ threshold ──► work items ──► tool fixes / tenet amendments
                                                │
                                                ▼
                              amendment ships WITH its incident receipt
```

The two halves cover different gaps. Transcript mining reconstructs friction after the fact and has to infer it from noisy logs. A feedback verb captures it live, with the invocation context attached, at near-zero cost. The self-correction in §1 is the loop's most useful artifact: the loop caught the document's own bad rule.

## 4. Design goals

Two positions anchor the rest:

- **The agent interface is the default.** Interfaces are designed by and for agents. A human interface, if needed at all, is a separate mode built on concrete need, and it does not compromise the agent interface. The common reflex — pretty by default, `--json` for machines — is the human-shaped default; agent-first inverts it.
- **Uniformity is the payoff.** The win is not any one tool's cleverness. It is that every tool behaves the same way, so learning one teaches the rest. A new tool should cost an agent that already knows the pattern nothing to learn.

Agents are also encouraged to build their own tools: they are the ones that feel the friction, and an agent designing a tool for its own use gets defaults right that a human designer guesses at. The price of that latitude is that agent-built tools follow the same tenets.

## 5. The tenets

### T1 — Agent-first output

A tool's default output should be structured data sized for the question, with a top-level `answer` string that is sufficient on its own, read without parsing. Detail beyond the answer should sit behind an explicit verbosity flag. The `answer` field is the contract: an agent that reads only that string has the right information; the rest is for deeper reasoning.

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

Human-pretty rendering (tables, colors, progress spinners) belongs in a separate renderer behind a flag. Box-drawing and ANSI banners cost roughly 2–5× the tokens of plain text in our measurements and add no information an agent needs. One round-trip means the caller shouldn't need to re-invoke to disambiguate; it does not mean the tool should return the whole table (§2.2, §1).

### T2 — Uniform contract

We aim for every tool to match across five dimensions:

| Dimension | Recommendation |
|---|---|
| **Discovery** | One machine-readable registry; every tool self-describes (`--describe`) |
| **Interface** | Structured default output; flexible any-direction input; errors are documentation; stable exit codes; provenance and staleness in the payload |
| **Auth** | Uniform across tools and abstracted from the agent (below) |
| **Feedback** | `<tool> feedback "…"` wired to a friction intake |
| **Lifecycle** | Declaratively deployed; metrics and logs exposed; registry and doc entry per tool |

Interface details:

- **Flexible input, any direction.** A resolver should accept a name, an ID, or an address and answer regardless of which was given. The agent should not need the tool's internal keying.
- **Errors are documentation.** On misuse or refusal, return correct usage plus a working example, machine-readably, in the channel the agent is already reading. The correction lands in context at the moment of the mistake, which is what separates a one-step recovery from a three-step one. A refusal that only says no sends the agent looking elsewhere.
- **Stable exit codes.** `0` ok · `1` not-found (don't retry) · `2` usage · `3` upstream-down (served from cache). Agents branch on exit codes, so per-tool numbering breaks that. Where not-found / retryable / side-effect distinctions matter beyond these four, encode them in the response envelope (`"retryable": false, "side_effects": "none"`) — machine-readable somewhere. "Exit 0, stdout contains Error" is how silent-success incidents happen; it is in the community CLI spec's failure catalog for the same reason (§11).
- **Provenance and staleness.** Every fact should carry `source` and live/cached status so an agent can reason about trust — most of all during an incident, when stale data is the danger. Related, and the reason the payload matters more than prose: agents report completion on failed runs. Prefer a machine-checkable success signal over an agent's claim about its own work.
- **Read-only by default.** Mutating tools should declare it in the registry and gate on scoped auth. On higher-stakes surfaces, mutation verbs earn preview/confirm semantics and idempotency keys — agents retry, and a retried non-idempotent mutation is an incident class we have seen.
- **Explicit execution locus.** A tool *runs* on the host adjacent to its data source and is *called* from anywhere via a documented transport. The locus should be a declared class (`control-plane`, `any-node`, `any`), never a pinned hostname; pinning breaks when the host is rebuilt.
- **Graceful degradation.** A tool that live-queries an upstream should cache last-good and serve it with a staleness stamp and exit `3`. That is what makes live query safe to reach for during the incident where the upstream is degraded.

**Auth: uniform and abstracted.** The storage convention matters less than two principles. (a) Auth handling should be uniform across tools, so an agent learns it once or never learns it. (b) Auth should be abstracted from the agent as far as the security posture allows: the agent should be able to use the tool correctly without handling, seeing, or transmitting a credential. The context window is a leak surface — anything pasted into it is now in transcripts, memory systems, and possibly a third-party provider's logs. A shape that works for us: a scoped least-privilege token in a `0600` env file, preset by a wrapper, never on the command line. Any shape achieving uniformity plus abstraction is fine; better still is auth the agent never encounters because the transport is credentialed or the data isn't sensitive (T4). Where a credential genuinely requires a human, the tool should say so in one uniform place rather than each tool inventing its own ceremony. The failure being designed out is agents fumbling per-tool auth: inventing token-delivery one-liners, pasting secrets into commands, or giving up and taking an untyped path.

### T3 — Feedback path on every tool

Every tool should expose `<tool> feedback "…"`, filing friction about that tool, auto-tagged with the tool name and the triggering invocation. The feedback path should be instant, never block, never throw, and spool offline when the intake is unreachable — a feedback verb that can fail is one that won't get used.

In our environment, a tool without a feedback path doesn't get registered.

This is what makes the §3 loop immediate: agents report in context, at the moment of friction, with no extra round-trips, capturing first-hand the signal transcript mining would later have to infer. A periodic rollup that clusters reports semantically and promotes recurring clusters to work items turns the stream into a roadmap. The pattern works because the complaint is filed about the tool while the agent holds the full invocation context — information that is gone once the session ends.

### T4 — Prefer tokenless, credential-free local transports

Where state is reachable over a network-native, no-secret path, prefer it over SSH hops or bearer tokens. Worked example: if the service mesh exposes discovery over DNS, service→node:port is a plain DNS SRV query from anywhere on the LAN — no SSH, no token, no dependency on a loopback-only admin API. Fewer secrets in flight, fewer failure modes in the tool's dependency chain, and no auth ceremony for the agent to fumble (T2 auth).

### T5 — Terse, memorable verbs

Tool name = service name = the human phrase. `drop` is the file-upload service, the CLI, and the sentence "drop me a file." Simple verb names remove name-reasoning from each call and make discovery questions ("how do I upload a file?") trivially matchable. Corollary: prefer a few consolidated high-level verbs over a sprawling flag surface. If a tool needs paragraphs of explanation, simplify the tool rather than the prose.

### T6 — Everything discoverable through `--help`

All functionality should be discoverable through `--help` (for MCP and function tools, through the tool description). Treat an undocumented flag as a bug. Agents recover from usage errors by reading help text; what isn't there invites a guess. `--help` is also the only documentation surface guaranteed to be present at the moment of need: an agent that can't find a flag in help text doesn't go read the wiki. Note the integrity caveat in §10 — descriptions are also an injection surface, so help text is a contract to audit, not just to write.

### T7 — One authoritative catalog

A single machine-readable registry is the source of truth for every agent-facing tool: name, canonical invocation, execution locus, auth model, output format, feedback verb, self-describe command, doc link, status. Human- and agent-facing discovery surfaces (a `tools` command, generated docs, an agent-memory pointer) are generated from it, so there is one thing to keep honest.

In our environment, an unregistered tool is either registered or removed — which makes audits cheap: you audit the catalog, not the filesystem. The catalog pointer in agent memory or instructions should be one line ("for any tool: run `tools`"). Long hand-maintained tool prose in instruction files drifts: six weeks after a restatement of these tenets, dependent surfaces still carried the previous version. The catalog should be the record and everything else should point at it.

## 6. Maintenance policy (how we run this in our environment)

Documented rules drift without a maintenance loop. What we do:

- Each operating environment periodically audits **its catalog** against the tenets and posts the gap list: what doesn't meet the contract, with tracked fix-or-delete items.
- **Rules change with evidence.** A new or amended rule ships with the measurement or incident that forced it. The bar applies to deletions too — a rule removed without a posted reason is hidden debt. When a rule's premise is refuted by new data, the rule gets rewritten rather than quietly ignored (§1, §3).
- Every gap-list item closes as fixed or deleted. A tool that can't reach the contract cheaply gets deleted rather than exempted; exemptions accumulate, deletions don't.
- A drift check asserts every registered binary or service has a catalog entry (T7), wired into CI or a merge gate, so registration can't be forgotten structurally.

## 7. Practice: untyped-fallback telemetry

If T1–T7 reduce friction, this practice measures the dangerous improvisation in §2.1 and §2.5 directly.

Every untyped fallback invocation (freeform `ssh <prod-host>`, break-glass credential ops, raw API mutation calls) spools one telemetry line: scrubbed command, agent or session ID, task context. Fallback volume is a catalog-defect metric: rising fallback use against a subsystem means the catalog is missing verbs agents need. The industry has named the same signal ("tool selection drift" in agent-observability tooling: "a 20% increase in raw-query tool usage may indicate the agent is bypassing the intended parser", §11).

Treat it as a demand signal, not a compliance failure. Punish untyped fallback use and agents stop surfacing it.

- **Classify by verb, not transport.** A call uses the canonical interface if its payload matches a registered verb's invocation pattern. If the sanctioned way to query a remote control plane is `ssh node 'tool …'`, that is the canonical interface, not a fallback.
- **Refusals should teach** (T2): an error that only says no routes agents toward untyped workarounds.
- **Track two failure classes separately:** missing verb (catalog gap — build it) versus registered verb with an incomplete or untrustworthy surface. The second is worse in our experience, because it trains avoidance of all verbs, including the good ones (§2.5).
- **Attribute by scope.** A fallback against an out-of-scope or one-off system is noise; one against a subsystem with a catalog is signal. Unattributed bypass logs contaminate the metric. Count state changes and receipts, not agent self-reports.
- **Act on it.** Every untyped fallback invocation with no intended verb on file opens a catalog issue.
- **Make the meter answerable.** Each environment should be able to report its own fallback-use count to a peer on demand: a real count, or "ungraded, wrappers pending."

**Memory hygiene is part of the loop.** Because workaround notes propagate further than the workaround was ever true (§2.5), environments using shared memory should treat "tool X broken, use Y" notes as stale-able: dated, scoped, and invalidated when the fix lands (friction report resolved → dependent memory entries revised). Fallback telemetry and memory rot are two ends of the same pipe.

## 8. What this looks like in practice

Worked examples across surface types, infra and beyond.

**Any-direction lookup instead of N tools.** `resolver <query>` answers name↔IP↔port↔ID in any direction with one positional argument, so the agent never needs to know which key it holds. The alternative — three per-direction scripts — is three invocations to memorize and three dead ends.

**The friction logger that can't fail.** Reference implementation of T3: one positional arg, no flags, no auth, no config; returns instantly; never blocks, never throws; auto-captures cwd, invocation, and tag; spools to a local file when the intake is unreachable and flushes later. Agents use it because it can't fail. Every other tool's `feedback` verb is a thin wrapper over it with its own tag preset.

**Test runner / build tool (dev surface).** Default output is `{answer: "3 failed, 211 passed. First: auth_test.py:88", failures: [...]}`, not the 2,400-line log. The full log sits behind a verbosity flag and `--help` teaches both modes. The agent decides what to read; the tool doesn't decide for it by dumping.

**Registry-as-code.** Registration is a YAML entry validated by a script and checked in CI; the `tools` command and the docs are generated from it. A new agent-facing tool that skips registration fails the drift check at merge time (T7).

**Auth abstracted away.** `deploy "app"` works with no credential handling: the wrapper presets a scoped token from a `0600` file; the agent never sees, types, or references it. All tools take the same shape, so there is no per-tool auth archaeology. Contrast a real defect we hit: one wrapper swallowed upstream auth stderr and surfaced a raw Python traceback; the agent couldn't read the failure, so it re-ran the mutation three times.

**Cached degradation as an incident feature.** A fleet-query tool caches last-good results and serves them with `"live": false, "cache_age_s": 312` and exit `3` when the control plane is down. During the incident that takes the control plane with it, the tool keeps answering — stamped stale — instead of joining the outage.

**Errors that redirect instead of blocking.** A wrapper refuses a raw flag and prints the typed verb to use instead, as machine-readable output. The agent proceeds through the canonical interface in the same turn.

**Fallback telemetry driving the roadmap.** A weekly count showed a spike in freeform SSH against the database tier; investigation found no registered verb for connection draining; the verb shipped; fallback volume against that tier dropped to near zero. The metric found a gap nobody reported, because nobody was supposed to be doing it by hand. The other direction, same data: two production burns (a double-restart, a remote engine kill) traced to an untyped fallback path while a typed verb existed but wasn't trusted — which is what forced the trust-versus-existence distinction in §7.

**The self-correction, end to end.** Transcript mining found that the queue-depth question cost 3,584 tokens; a feedback-report cluster independently said the same; the work item added a `--count` verb to the queue tool; the fix was verified against the next week's corpus; the document gained the receipt in §2.2. The loop produces both better tools and a better document.

## 9. Adoption checklist

- [ ] Default structured output with a top-level `answer`, sufficient without parsing; detail behind a verbosity flag
- [ ] Flexible any-direction input; errors and refusals return usage plus a working example
- [ ] `--describe` emits a machine-readable contract line
- [ ] Provenance and staleness fields; stable exit codes (`0/1/2/3`) or envelope-encoded equivalents
- [ ] Read-only by default; mutations declared, scoped-auth-gated, ideally idempotent-safe
- [ ] Auth uniform across tools and abstracted from the agent as far as security allows; no credential in the agent's context
- [ ] Live-query tools cache last-good and degrade with exit `3`
- [ ] `<tool> feedback "…"` exists, is tool-tagged, and can't fail
- [ ] Tokenless local transport preferred where one exists
- [ ] Everything discoverable through `--help`; terse verb naming
- [ ] Registered in the authoritative catalog, with a doc entry
- [ ] Untyped fallback paths spool telemetry — classified by verb, attributed by scope, acted on via catalog issues

## 10. Open problems

- **Descriptions are an injection surface.** `--help` text, tool descriptions, and catalogs are instructions an agent is required to read, which makes them a tool-poisoning vector: documented attacks embed instructions in tool descriptions invisible to humans but obeyed by models. T6 and T7 assume a trusted environment. Crossing that boundary needs the extra machinery — signed or hash-pinned catalogs, help text reviewed like code, third-party tool descriptions treated as untrusted input.
- **Four exit codes is a floor, not a ceiling.** Real systems need not-found / retryable / partial-side-effect distinctions at the layer where agents make retry decisions. Encode the richness in the envelope or accept ambiguity where it costs most.
- **Catalog existence does not imply catalog consumption.** Publication standards for agent-readable indexes have died unread; the large majority of published `llms.txt` files are never fetched (§11). Audit consumption, not just registration, or the gap list is decorating a dead file.
- **Uniform multi-verb contracts add per-call ceremony.** Verb discipline (preview/verify phases) and token efficiency are in tension. Read-only tools need an exemption path so adoption doesn't mean a five-call handshake to read one number.
- **Some surfaces can't be redesigned.** For third-party and vendor tools, the harness side (wrappers, output filters) is the right way to apply this — the contract applies to the surface the agent touches, not necessarily the binary underneath.
- **Untyped-fallback telemetry can be gamed or contaminated.** Count receipts and state changes, not claims; attribute scope; expect bookkeeping artifacts in early measurement (one of our counters peaked at 124 events and was 68% bookkeeping noise).

## 11. Related work

We found these independently, and they are the reason to think the tenets generalize past our two environments — not proof, but consistent evidence from other people hitting the same failures from other directions.

- **SWE-agent / Agent-Computer Interfaces (Yang et al., ICLR 2025)** — treats agents as a new category of end users needing purpose-built interfaces; measured +64% relative task success from interface redesign alone, with ablations on output-window size and concise per-turn feedback. <https://arxiv.org/abs/2405.15793>
- **Anthropic, "Writing effective tools for agents"** — supports T1/T2: structured response formats cutting token cost ~3×, error messages as teaching moments, result-size caps, and a telemetry-driven tool-redesign loop like §3. <https://www.anthropic.com/engineering/writing-tools-for-agents>
- **cli-agent-spec** — community spec converging on the same failure catalog (exit-0-but-failed, pager hangs, flag-order traps) with a richer exit-code contract (`retryable`, `side_effects` per code) — see §10. <https://github.com/cli-agent-spec/cli-agent-spec>
- **Token-tax measurements** — ~89% of measured CLI output tokens classified as noise (banners, progress art, box-drawing); MCP schema front-loads measured at ~55K tokens against near-zero for CLI + `--help` discovery. <https://alies.dev/articles/cli-output-for-ai> · <https://simonwillison.net/2025/Aug/22/too-many-mcps/>
- **"MCP Tool Descriptions Are Smelly!"** — 97.1% of sampled production tool descriptions contain at least one quality smell: the external case for T6/T7 and gap-list audits. <https://arxiv.org/abs/2602.14878>
- **Tool selection drift** (agent-observability tooling, e.g. Fiddler/Datadog MCP monitoring) — the industry's name for §7's routing-around signal. <https://www.fiddler.ai/blog/mcp-agent-observability>
- **Agent memory rot / fail-closed shared memory** — practitioner evidence for §2.5 and §7's memory hygiene: propagated stale workaround notes, and the "you can always get past a gate, you can never get past one silently" bypass-logging pattern. <https://vuk.digital/fail-closed-shared-memory-for-ai-agents>
- **Tool poisoning attacks** — the integrity caveat in §10. <https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks>

---

*Questions, amendments, and adoption reports: open an issue. Friction with this document itself counts as feedback — see T3.*
