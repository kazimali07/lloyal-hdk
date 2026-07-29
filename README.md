# lloyal HDK

[![CI](https://github.com/lloyal-ai/hdk/actions/workflows/ci.yml/badge.svg)](https://github.com/lloyal-ai/hdk/actions/workflows/ci.yml)
[![GPU Tests](https://github.com/lloyal-ai/lloyal.node/actions/workflows/gpu-test.yml/badge.svg)](https://github.com/lloyal-ai/lloyal.node/actions/workflows/gpu-test.yml)
[![License](https://img.shields.io/badge/license-FSL--1.1--Apache--2.0-blue.svg)](LICENSE)
[![Commercial Use](https://img.shields.io/badge/commercial%20use-unrestricted-brightgreen.svg)](#why-fsl-instead-of-mit)

**Vertical inference — put the model *inside* your application.**

*A full-stack agentic AI framework with the model in your process. Build intelligence as a program you own, not as calls to a remote API.*

> **Post-training produces a tendency. A harness produces a procedure.**

An inference endpoint makes a model **callable** — you send text, you get text, and the reasoning lives behind an HTTP boundary you can't reach. HDK makes it **programmable**: the model runs inside your Node process, and your application addresses its *live attention state* directly — forking lines of reasoning, admitting evidence mid-generation, deciding what becomes a durable result. Same process, same memory, same data structures as the rest of your code. No inference server, no vector DB, no embedding pipeline, no per-token bill.

You write the reasoning once as an ordinary TypeScript program — a **harness** — and run it as a terminal app, a desktop app, or a served browser app off that one program. Every agent scaffold starts with `API_KEY=`. **A harness starts with a model.** Nothing on the reasoning path touches a third-party API; `grep API_KEY` in a scaffolded project finds nothing.

Free to use, embed, ship, and sell — commercial, private, internal, all of it. The one carve-out — forking the runtime to compete — is the [trust boundary](#why-fsl-instead-of-mit) that keeps capability-bearing apps installable safely. Converts to Apache 2.0 on a rolling two-year schedule.

<p>
  <img src="assets/demo-readme.gif" alt="Deep Research: 5 agents researching concurrently inside a shared 32K-token context window, plan → research with tool calls → synthesize" width="100%">
  <br>
  <em>Qwen3.5 4B + Qwen3 0.6B reranker · 5 parallel agents · shared 32K context · fully offline on M2 MacBook Pro 16 GB</em>
</p>

> The demo above is [**reasoning.run**](https://www.npmjs.com/package/reasoning.run), a deep-research CLI built with HDK. Try it in 30 seconds: `npx reasoning.run`.

## Get started

Scaffold your own harness — no API key, the model runs in-process:

```bash
npx harness.dev new              # interactive: name → surfaces → model → template
cd my-harness && npm install
npm start
```

```text
scaffolded acme (blank) · targets: cli, desktop, web · model: qwen3.5-4b
  Model      qwen3.5-4b                    ● resident
  Inference  local · no provider endpoint  ● offline
  ready — type to begin, ctrl-c to stop
```

The default **blank** scaffold ships the `lloyal/wikipedia` App (no auth) so the first command works with no key and no setup; `--template research` wires the tuned research pipeline over `lloyal/web` + `lloyal/corpus`.

**Run any surface** off the same program — same events, different binding:

```bash
npm start              # cli    — the terminal app
npm run dev:desktop    # desktop — an Electron window
npm run serve          # web    — boot the local host, then `npm run dev:web`
```

**Then, without touching `harness.ts`:**

```bash
harness.dev install lloyal/web      # add a signed capability
harness.dev targets:add web         # add a surface
harness.dev app:new jira            # scaffold your own capability → publish
```

Full command surface → [`harness.dev`](./packages/harness-cli/README.md).

## The shift

The unit you build moves from *an inference endpoint your app calls* to *an application your app is*. When the model is remote, every agent is a fresh conversation you re-feed context to, and concurrency multiplies cost linearly. When the model is embedded, agents fork a shared line of reasoning for free, correct each other in real time, and synthesize — coordination patterns that simply aren't expressible over API calls.

That difference isn't faster inference. It's a different *kind* of capability, and it's a cross-product, not one API: your application composes **topology** (which lines of reasoning exist and how they relate) × **observation** (watching a generation as it runs) × **evidence** (what gets admitted into context, and when) × **lifecycle** (when output becomes an accepted result) × **authority** (which capabilities a step may use) × **continuity** (the same program, preserved across turns, models, surfaces, and deployments). The model runs inside the harness — governed, forkable, and yours.

## What you get

- **A programming model, not an SDK.** A harness is *a tree of owned lifetimes over a tree of live inference state.* Agents bind to parent scopes via [Effection](https://frontside.com/effection) structured concurrency — cancellation propagates, teardown runs in reverse, cleanup is inseparable from ownership. Loops, conditions, and lexical scope **are** your orchestration; there's no graph DSL to learn.
- **Continuous-context agents.** Sub-agents fork the parent's full attention at zero tensor copy — one shared model context, N branches, **one GPU dispatch per tick regardless of branch count.** A fork is a bitset flip, not a recompute; cost tracks KV *fullness*, not agent count, so two vs. ten concurrent agents decode at the same per-tick speed. *(Code-confirmed against the vendored llama.cpp build — see [Why in-process](#why-in-process-is-a-different-capability).)* The result: **4.4× fewer tokens processed** than a prompt-rebuilding pipeline.
- **Retrieval-interleaved generation.** Agents assemble context _during_ generation — searching, reading, and reranking across your app's own data. One `Source` shape for files, SQL, the web, or user records. A cross-encoder focal lens admits only verbatim top-K chunks — never summarized.
- **A signed App platform.** Capabilities — web search, browser automation, payment connectors, your company's data — install as **Apps** from a curated channel at [`apps.lloyal.ai`](https://apps.lloyal.ai). Every bundle is Ed25519-signed and verified against an embedded trust root *before it runs*; the CLI shows an App's *attention surface* — protocol, tools, config, skill lines — from the verified bytes first. What you install is what was reviewed.
- **One harness, every surface, every tier.** Write the program once; each surface — terminal, desktop, browser — is a binding over the same events, all folding one `reduce`. The same contract runs it on a laptop, a shared GPU box, or a served fleet. *Where* it runs is a deployment decision, not an application one.

Mechanics, receipts, and the case for the architecture at [hdk.lloyal.ai](https://hdk.lloyal.ai).

## The programming model

The application contract is deliberately small — a harness is a scope that stays alive for a Session:

```typescript
export function* harness(
  ctx: SessionContext,               // the resident model + native session
  events: EventBus<WorkflowEvent>,   // application events → whichever surface is mounted
  commands: Signal<Command, void>,   // typed commands ← that surface
): Operation<void> {
  const { session } = yield* initAgents(ctx);

  for (const command of yield* each(commands)) {
    // Borrow a shared line of live attention; fork a cohort of agents over it.
    const notes = yield* withSpine({ parent: session.trunk, systemPrompt, tools }, function* (spine) {
      const pool = yield* agentPool({ parent: spine, terminal: reportTool, orchestrate: parallel(tasks) });
      return pool.agents.flatMap((a) => (a.result ? [a.result] : []));  // findings leave as data
    });

    const synth = yield* useAgent({ parent: session.trunk, task: renderSynthesis(notes) });
    yield* call(() => session.commitTurn(command.query, synth.result));  // durable, deliberately

    yield* each.next();
  }
}
```

Three trees describe one run, and they don't have to line up: the **lifetime tree** (Effection — what ends together), the **inference-state tree** (BranchStore — what attention is inherited), and the **orchestration graph** (your code — what depends on what). When the Session is released, the harness scope ends and every child — pools, tool calls, temporary branches — unwinds with it. You never enumerate what to cancel; the ownership tree already knows.

Reshape execution — breadth, depth, or a graph — by wrapping the pool in a `parallel` / `chain` / `fanout` / `dag` orchestrator, without changing the call. Full model at [docs.lloyal.ai](https://docs.lloyal.ai/start-here/what-is-the-hdk).

### Embed in an existing project

Skip the scaffold and wire the runtime into code you already have:

```bash
npm i @lloyal-labs/lloyal-agents @lloyal-labs/lloyal.node @lloyal-labs/rig
npx harness.dev install lloyal/wikipedia   # or lloyal/web, lloyal/corpus, acme/...
```

```typescript
import { main, call } from "effection";
import { createContext } from "@lloyal-labs/lloyal.node";
import { initAgents, useAgent } from "@lloyal-labs/lloyal-agents";
import { createAppRegistry, createInMemoryConfigStore, reportTool } from "@lloyal-labs/rig";
import { createWikipediaApp } from "@lloyal-labs/wikipedia-app";

main(function* () {
  const ctx = yield* call(() =>
    createContext({ modelPath: "model.gguf", nCtx: 32768, nSeqMax: 8, typeK: "q4_0", typeV: "q4_0" }),
  );
  yield* initAgents(ctx);

  const registry = yield* createAppRegistry({ configStore: createInMemoryConfigStore() });
  const wikipedia = yield* registry.enable(createWikipediaApp);

  const a = yield* useAgent({
    systemPrompt: "You are a research assistant.",
    task: "Who founded the city of Brasília, and when?",
    tools: [...wikipedia.tools],
    terminal: reportTool,
  });

  console.log(a.result);
});
```

## Apps — the signed capability channel

An **App** wraps a Source + Tools + a per-spawn skill template + a manifest, validated by `defineApp`. Three reference Apps ship first-party: `lloyal/web` (web search + page fetch), `lloyal/corpus` (local-doc grep + read + semantic search), and `lloyal/wikipedia` (the auth-free demo backend the **blank** scaffold defaults to).

```bash
npx harness.dev install lloyal/web         # install a reviewed capability
harness.dev targets:add web                # add a surface — never touches harness.ts
harness.dev models:use <id>                # swap the resident model
```

Shipping a capability of your own — a vertical API, your company's internal data, a browser-automation runtime — means publishing an App through the channel for other harnesses to install. First- and third-party ride the same Ed25519-verified path:

```bash
npx harness.dev app:new jira --publisher acme  # scaffold an App
npx harness.dev publish                        # ship through the signed channel
```

## Why in-process is a different capability

Most "AI for TypeScript" tools are a **client to an inference endpoint** — the Vercel AI SDK calls a provider's API; LangGraph orchestrates a graph of calls over one; even "local," through Ollama, the model is a separate daemon you POST to. The model is a service behind a boundary, so every agent, every turn, re-ships its context and pays per token.

HDK isn't a client — it's a **runtime that embeds the model**, the way an app embeds SQLite instead of reaching a database over the network. The weights are resident in your process, and your harness governs the model's *live* reasoning state as it runs. That's the difference between renting behaviour request-by-request and owning it as code — and it's why concurrent agents are cheap. Endpoint tools coordinate agents like **VMs**: each a full, isolated context you stand up and re-feed. HDK runs them like **containers on one kernel**: every agent is a zero-copy *branch* of one resident model state.

The mechanism is verifiable, not marketing — the numbers below are **code-confirmed against the vendored llama.cpp build**, read from source, not reconstructed:

- **N branches, one dispatch.** N branches that fit the micro-batch decode in **one `llama_decode`** — the batch splitter cuts on token rows and never reads `seq_id`. GPU dispatches per tick are **O(1) in branch count**.
- **Forking is free.** A fork (`seq_cp`) allocates no cells and copies no buffer — it's a single `std::bitset<LLAMA_MAX_SEQ>` write, one cell now owned by two branches. Zero decode, zero attention compute. *This is* prefix sharing, by construction.
- **Cost is KV fullness, not agent count.** Per-tick wall-time is `O(n_kv × token_rows)` — there is no `× n_seqs` multiplier. **Two vs. ten concurrent agents decode at the same per-tick speed;** concurrency is free on compute and priced only in KV *space*.

If you know the serving stack: vLLM and SGLang already reuse prefixes (RadixAttention) — but as a **server**, where the shared prefix is a token-keyed KV *cache* (radix-matched, LRU-evicted; a miss recomputes) reached over an API. HDK puts that tree *inside your application*: a branch is a structural **back-reference** into its parent's live cells (nothing to match, cache, or evict), pruned **semantically** when the reasoning is done — governed by policy, not evicted when a cache runs cold. A cloud per-token API structurally cannot replicate this economics.

## Stack vs. imports

The honest comparison is full stack against full stack. Each row of the right column is a service to install, configure, version, secure, and orchestrate. Each row of the left column is an import.

| Typical agent stack                                             | HDK                                   |
| --------------------------------------------------------------- | ------------------------------------- |
| Inference server (vLLM / Ollama / llama-server)                 | `@lloyal-labs/lloyal.node`            |
| Agent runtime (LangChain / LangGraph / AutoGen / CrewAI)        | `@lloyal-labs/lloyal-agents`          |
| Vector DB (Pinecone / Weaviate / pgvector) + embedding pipeline | Apps (`@lloyal-labs/web-app`, `@lloyal-labs/corpus-app`, your own) |
| Retrieval orchestration (Haystack / LlamaIndex)                 | `@lloyal-labs/rig`                    |
| Process orchestrator (Docker compose / Kubernetes / Airflow)    | TypeScript scopes (Effection)         |
| Frontend transport + served fanout                              | `@lloyal-labs/binding` + `@lloyal-labs/host` / `@lloyal-labs/relay` |
| Glue code                                                       | `npm i`                               |

## Public API

```typescript
// Agent runtime
import {
  initAgents, useAgent, agent, agentPool, useAgentPool, diverge,
  parallel, chain, fanout, dag, reduce, withSpine,
  Tool, Source, DefaultAgentPolicy,
  Ctx, Store, Events, AppRegistryCtx, AppConfigStoreCtx, GrantStoreCtx, RerankerCtx,
} from "@lloyal-labs/lloyal-agents";

// App protocol + framework tools
import {
  defineApp, createAppRegistry, createInMemoryConfigStore, createGrantStore,
  renderSpine, renderAgentPreamble,
  reportTool, PlanTool, DelegateTool, TavilyProvider, createKeylessSearchProvider,
} from "@lloyal-labs/rig";
```

That is essentially the framework.

## Repo layout

```
packages/
  agents/        @lloyal-labs/lloyal-agents — agent runtime — structured concurrency over shared KV state
  sdk/           @lloyal-labs/sdk           — backend-agnostic inference primitives (Branch, Session, Rerank)
  rig/           @lloyal-labs/rig           — App protocol helpers + retrieval providers + framework tools
  binding/       @lloyal-labs/binding       — the harness's headless interface: the event/command binding + its transports
  host/          @lloyal-labs/host          — the box model-runtime host: one resident model, N native harness sessions
  relay/         @lloyal-labs/relay         — the self-hostable relay: serves a headless harness to remote frontends over wss
  apps/
    web/         @lloyal-labs/web-app       — first-party web research App
    corpus/      @lloyal-labs/corpus-app    — first-party local-corpus research App
    wikipedia/   @lloyal-labs/wikipedia-app — first-party Wikipedia demo App
  harness-cli/   harness.dev                — scaffold · install · publish · review CLI (Apache 2.0)

examples/
  compare/       DAG primer (App-protocol-shaped): parallel research → compare → synthesize
  react-agent/   Pre-App-protocol `useAgent` baseline (mechanism demo, not a 3.0 reference)
  reflection/    Pre-App-protocol `diverge` primer (research → draft → critique → revise)
```

`reasoning.run` is the production-grade reference harness — `npx reasoning.run` and read its source. The native binding [`@lloyal-labs/lloyal.node`](https://github.com/lloyal-ai/lloyal.node) lives in a separate repo and is pulled in as a dependency.

## Requirements

- **Node 22+**
- **A GGUF model file on disk** — any model the native backend supports (the scaffold fetches one, digest-verified, on first run)
- macOS / Linux / Windows on x64 or arm64. CPU works; CUDA / Metal / Vulkan supported via prebuilt native binaries.
- **Native backend:** [llama.cpp](https://github.com/ggml-org/llama.cpp) today, via `@lloyal-labs/lloyal.node`. The SDK and harness contracts sit above the engine — intelligence is written against the runtime, not the backend.

## Compatibility

GPU integration tests run against six architectures and chat-template families on every PR:

| Model                 | Params | Quant  | Template |
| --------------------- | ------ | ------ | -------- |
| SmolLM2-1.7B-Instruct | 1.7B   | Q4_K_M | ChatML   |
| Llama-3.2-1B-Instruct | 1B     | Q4_K_M | Llama 3  |
| Phi-3.5-mini-instruct | 3.8B   | Q4_K_M | Phi 3    |
| Qwen3-4B-Thinking     | 4B     | Q4_K_M | ChatML   |
| gemma-3-1b-it         | 1B     | Q4_K_M | Gemma    |
| GLM-Edge              | —      | Q4_K_M | GLM-Edge |

The native backend ships prebuilt binaries across 13 platform/GPU combinations:

| Platform    | arm64             | x64               |
| ----------- | ----------------- | ----------------- |
| **macOS**   | Metal             | CPU               |
| **Linux**   | CPU, CUDA, Vulkan | CPU, CUDA, Vulkan |
| **Windows** | CPU, Vulkan       | CPU, CUDA, Vulkan |

## Development

```bash
git clone https://github.com/lloyal-ai/hdk
cd hdk
npm install
npm run build       # tsc -b across workspace
npm test            # unit tests
```

Every PR runs build, typecheck, and unit tests on CI, plus a cross-repo GPU integration job: HDK PRs trigger [`lloyal-node`](https://github.com/lloyal-ai/lloyal.node)'s GPU workflow, which builds the PR's packages against the native runtime on NVIDIA L4 hardware and runs the full agent integration suite before merge.

## Docs

- **What HDK is and why** → [hdk.lloyal.ai](https://hdk.lloyal.ai)
- **Learn, reference, guides** → [docs.lloyal.ai](https://docs.lloyal.ai)
- **API reference** — TypeDoc-generated from source

## Why FSL instead of MIT?

HDK apps are **capability-bearing** — arbitrary code (browser automation, file access, payment connectors) bundled with skill instructions, running in shared inference context. OS sandboxing protects the machine; it does nothing about what an app's content reaches the model's attention. Cloud agent platforms can yank misbehaving extensions with a kill switch; HDK runs on user machines and can't.

Safety has to be **upstream and structural**: the canonical channel at [apps.lloyal.ai](https://apps.lloyal.ai) reviews and Ed25519-signs every App; the runtime verifies that signature against an embedded trust root at install. MIT doesn't preserve that — a fork could strip the trust root and ship to an unreviewed channel. FSL restricts one thing — that fork — to keep the trust root enforceable. It can't stop a determined bad actor; it keeps channel-switching from being the easy path.

## License

**Commercial use is unrestricted** — build and sell products with HDK, embed it in proprietary software, run it in production. The FSL restriction is narrow: you cannot ship a competing HDK runtime, managed HDK service, or alternative HDK App distribution channel.

HDK runtime packages (`@lloyal-labs/lloyal-agents`, `@lloyal-labs/sdk`, `@lloyal-labs/rig`, `@lloyal-labs/binding`, `@lloyal-labs/host`, `@lloyal-labs/relay`, `@lloyal-labs/web-app`, `@lloyal-labs/corpus-app`, `@lloyal-labs/wikipedia-app`) are Fair Source under FSL-1.1-Apache-2.0 and convert to Apache 2.0 two years after each release. `packages/harness-cli` (the `harness.dev` CLI) is Apache 2.0 from day one — see its own `LICENSE` file.

See [`LICENSE-FAQ.md`](./LICENSE-FAQ.md) for concrete examples of what's permitted and what's restricted, [`LICENSE`](./LICENSE) for the legal text, and [`NOTICE`](./NOTICE) for attribution.
