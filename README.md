<div align="center">

# Aielia

**An everyday AI assistant that thinks before it acts — and stops before it sends.**

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Built with](https://img.shields.io/badge/built%20with-Build%20A%20Harness-4c1.svg)](https://github.com/3IVIS/buildaharness)
[![Try it](https://img.shields.io/badge/try%20it-buildaharness.com%2Ftry-orange.svg)](https://buildaharness.com/try)

</div>

---

Aielia is the personal assistant built with **[Build A Harness](https://github.com/3IVIS/buildaharness)**.
It is light enough for "what time zone is Tokyo in?" and careful enough to
**stop for your approval** before it sends an email, pays an invoice, runs a
shell command, or deletes a file.

The difference is the *harness*. A harness is a governance and reliability
control plane wrapped around an autonomous agent: the agent has the
intelligence, the harness has the authority. The agent proposes what to do;
the harness decides what is actually allowed to happen next; evidence decides
whether the result is accepted. Aielia routes **every turn** through an
11-layer implementation of that control plane — governing what it believes,
what it is allowed to do, how it catches its own mistakes, and what it learns.

---

## Try it

**In your browser → [buildaharness.com/try](https://buildaharness.com/try)**

Bring your own model key (Anthropic / OpenAI / OpenRouter — stored only in
your browser's `localStorage`, never proxied server-side), or watch the
approval gate fire before you add one.

**In your terminal:**

```bash
npx @buildaharness/personal-assistant
```

First run walks you through picking a model — reuse an existing `claude` CLI
login (no API key needed), or paste an Anthropic / OpenAI / OpenRouter key.
Then just talk to it.

**As a library:**

```ts
import { LLMClient } from '@buildaharness/runtime'
import { PersonalAssistant } from '@buildaharness/personal-assistant'

const aielia = new PersonalAssistant({
  llmClient: new LLMClient({ proxyUrl, authToken }),
})

await aielia.turn('What time zone is Tokyo in?')
// { status: 'ok', reply: '…', riskLevel: 'LOW', stepsUsed: 1 }

await aielia.turn('Send an email to my boss saying I quit.')
// { status: 'needs_approval', reason: '…', riskLevel: 'HIGH' }  — no LLM call made

await aielia.turn('Send an email to my boss saying I quit.', { approved: true })
// approved — proceeds and runs the harness normally
```

One core, three front ends — terminal CLI, browser (`@buildaharness/chat-ui`),
and native desktop (`@buildaharness/desktop`) — all running the identical
harness underneath.

---

## What makes it different

Most open agents ship either a permission model *or* an output-quality gate.
Aielia ships both, plus the layers that connect them:

- **Live per-tool-call ControlState gate** — every read-only tool call is
  checked against a per-turn `ControlState` *before* it runs (deterministic
  ALLOW / DENY / REQUIRE_APPROVAL, not advisory), so a developing failure
  pattern can trip a real deny mid-turn.
- **Staged effects** — `write_file`, `run_shell_command`, and `send_email`
  never execute inline. They stage the *exact* proposed action for approval;
  once you approve, that staged action runs verbatim — no second model call
  can improvise a different one.
- **Fail-safe risk classification** — a classifier error or unparseable
  response returns `UNKNOWN → requires approval`, never a silent default to
  low-risk.
- **Reviewer Pass** — a 3-lens review (consistency, adversarial, abstraction
  fit) and output-contract validation run before a reply goes out.
- **Typed fact provenance** — only facts you actually *state* promote to
  durable memory by default; model-inferred facts stay session-scoped until
  confirmed.
- **AnswerClaim** — replies distinguish "verified against evidence" from
  "found this but couldn't independently confirm it," surfaced in the chat's
  "Why?" panel.
- **Crash-safe mid-turn resume** — a turn that dies mid-flight resumes from
  its last checkpoint instead of silently starting over; a checkpoint that
  keeps crashing on replay is discarded automatically after two attempts.
- **Untrusted-content boundary** — web results and shell output are wrapped
  as data the model is instructed never to follow as commands, with a warning
  prefix when the text looks instruction-shaped.
- **Private-network guard** — `fetch_url` resolves the target host and
  refuses loopback / RFC1918-private / link-local / cloud-metadata addresses,
  re-checked on every redirect hop.

---

## How it's implemented

Aielia is the `@buildaharness/personal-assistant` package. Where a heavy
autonomous agent decomposes an objective into a multi-task plan, Aielia
treats **every chat message as one objective, one task**. That keeps the
harness cheap per turn — the harness loop itself makes *no* LLM calls, it is
synchronous state-machine bookkeeping — while still walking every layer.

```
Caller State  →  World Model  →  Evidence & Reasoning  →  Control State
     →  Planning (trivial one-task graph)  →  Execution  →  Verification (9 layers)
     →  Recovery  →  Memory (context compression)  →  Learning (ExperienceStore)
     →  Output & Reviewer Pass
```

**One real LLM call per ordinary turn. Zero for a blocked one.** Every layer
of the harness is touched on the turns that do run.

### The 11 layers

| Layer | Role |
|---|---|
| **Caller State** | Constraints and clarifications carried into the turn |
| **World Model** | Typed beliefs, contradiction detection, generation IDs |
| **Evidence & Reasoning** | Observations from tools, tool-reliability tracking, value-of-information |
| **Hypothesis** | Candidate explanations from multiple sources |
| **Contradiction** | Belief-graph conflict detection over knowledge-tier facts only |
| **Control State** | 5-tier resolver → NORMAL / CAUTIOUS / BLOCKED; ALLOW / DENY / REQUIRE_APPROVAL per action |
| **Planning** | Task graph (one task per chat message here) |
| **Execution** | Tool dispatch behind the VOI gate and the per-call policy check |
| **Verification** | 9-layer output verification before a result is accepted |
| **Recovery** | 6 named recovery strategies over a typed failure library |
| **Memory** | Six memory tiers (episodic, semantic, procedural, preference, commitment, identity) with structurally-enforced provenance |
| **Learning** | `ExperienceStore` — strategy weights, learned decompositions, recovery sequences reused across turns |
| **Reviewer Pass** | 3-lens adversarial review + output-contract validation |

### What lives outside a single harness run — deliberately

- **Conversation history** — kept in a `MemoryAdapter` (in-memory by default;
  IndexedDB in the browser, filesystem in the CLI / desktop app) and fed to
  the model directly. Each message is also individually indexed so `/search`
  can resolve a hit to the one exchange that matched.
- **Risk classification** — a cheap keyword heuristic flags consequential
  requests *before* the harness and before the one real network call ever
  runs. A separate LLM-backed classifier produces non-binding *hints*; the
  actual approval requirement is always recomputed from structural signals.
- **Learning across turns** — the `ExperienceStore` persists strategy weights
  and learned decompositions so they survive a reload.

### Storage per front end

| Front end | Where | Storage |
|---|---|---|
| `PersonalAssistant` class | Any Node / browser code | In-memory; bring your own adapter |
| CLI (`npx @buildaharness/personal-assistant`) | Terminal | Files under `~/.buildaharness/personal-assistant/` |
| `@buildaharness/chat-ui` | Browser | IndexedDB / Dexie |
| `@buildaharness/desktop` | Native window (Tauri) | Files under the OS app-data dir |

### LLM backends

`proxy` (a self-hosted `@buildaharness/proxy`), `claude-cli` (shells out to an
authenticated `claude` CLI — no API key), or a provider directly:
`anthropic`, `openai`, `openrouter` with your own key. On the `claude-cli`
backend, tool calls resolve inside the `claude` subprocess, so the same
deterministic policy gate is enforced through an ephemeral loopback socket the
MCP server blocks on before executing a read-only call.

Full write-up:
[`packages/personal-assistant/README.md`](https://github.com/3IVIS/buildaharness/blob/main/packages/personal-assistant/README.md)
in the Build A Harness repo.

---

## Build your own harness

Aielia is the front door. Underneath it is a full visual **harness builder** —
draw the same 11 layers on a canvas, compile to LangGraph / CrewAI / Mastra /
MS Agent Framework, and trace every decision in Langfuse.

```
Canvas  →  flow.json  →  LangGraph · CrewAI · Mastra · MS Agent Framework  →  Langfuse
```

See **[github.com/3IVIS/buildaharness](https://github.com/3IVIS/buildaharness)**.

---

## Links

| | |
|---|---|
| Try Aielia in the browser | https://buildaharness.com/try |
| Build A Harness project | https://buildaharness.com |
| Source (monorepo) | https://github.com/3IVIS/buildaharness |
| npm package | `@buildaharness/personal-assistant` |
| Assistant deep-dive | [`packages/personal-assistant/README.md`](https://github.com/3IVIS/buildaharness/blob/main/packages/personal-assistant/README.md) |
| Architecture | [`docs/architecture.md`](https://github.com/3IVIS/buildaharness/blob/main/docs/architecture.md) |
| Threat model | [`docs/threat-model.md`](https://github.com/3IVIS/buildaharness/blob/main/docs/threat-model.md) |

---

<div align="center">

Apache 2.0 — see [LICENSE](LICENSE). Part of [Build A Harness](https://github.com/3IVIS/buildaharness).

</div>
