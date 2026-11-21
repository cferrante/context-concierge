# Context Concierge (agentic RAG)

> **Status: paused (2026-11-16) while the portfolio site at `cferrante.dev` goes live; it will
> run there as a feature ([design §15](docs/design.md#15-running-as-a-feature-of-cferrantedev)).**
>
> **Built and evaluated locally; not deployed yet.** Phases 0–6.3 of the
> [plan](docs/plan.md) are done: AWS risk spikes, the C++ retrieval engine, the Go orchestrator,
> the split-pane UI, real models (Groq with a Bedrock fallback, Titan embeddings), a golden
> question set with a retrieval gate in CI, and an answer judge checked against hand labels.
> Next is the adversarial suite (6.4), then a private deploy on AWS (Phase 7). See
> [Progress](#progress) for what is measured and what is still a target.
>
> The design spec is in [docs/design.md](docs/design.md), the build order in
> [docs/plan.md](docs/plan.md), targets and measured results in
> [docs/benchmarks.md](docs/benchmarks.md), and every term is explained in
> [docs/concepts.md](docs/concepts.md).

A fast, intelligent, transparent AI assistant for my portfolio site: the **Context Concierge** (the
name is configurable, `ASSISTANT_NAME`). Visitors ask questions about me and my work; the
Context Concierge answers from my own documents ([RAG](docs/concepts.md#rag)), cites its sources, can take
actions through tools, and **shows its work**: you watch it think on the left and watch the backend
act on the right, with real measured timings.

**Goals:** prove *speed* and *intelligence* of responses, at nearly $0 a month.

## Progress

| Milestone | Proves | Status |
|---|---|---|
| **M1 Risks retired** | cgo on Lambda, Go SSE streaming, CloudFront passthrough, Groq speed | ✅ 2026-11-01 |
| **M2 Local end-to-end** | Full chat with thought stream and action cards on my laptop | ✅ 2026-11-02 |
| **M3 Safe and smart** | Real models, judge, golden + adversarial evals passing | In progress: 6.0–6.3 done, 6.4 (adversarial) next |
| **M4 Private deploy** | Running on AWS, reachable only by me | Not started (Phase 7) |
| **M5 Public launch** | Invite-only features, logging, privacy, measured benchmarks | Not started (Phases 8–10) |

**Measured so far** (method and every run in [docs/benchmarks.md](docs/benchmarks.md#results)):

| What | Result | Where |
|---|---|---|
| Hybrid retrieval, 2k chunks × 512 dims | p50 0.122 ms, p99 0.225 ms | Laptop; CI fails the build at p99 ≥ 1 ms |
| C++ vs pure Go, vector top-k | 0.097 ms vs 0.408 ms p50 | Laptop, same day, same contract |
| Groq gpt-oss-120b from Lambda | TTFT p50 198 ms, 483 tokens/s | Spike 1.5, n = 20 |
| Lambda cold start, Go + SSM key read | 205 ms | Spike 1.5, n = 1 |
| Retrieval hit@5, Titan embeddings | 58 / 61 public questions | CI gate at 0.95 |
| Answer evals, judged on Groq | 49 / 64 public questions pass (77%) | Baseline at temperature 0; report-only until run-to-run spread is measured |
| Suggested question, stored answer | p50 0.9 ms | `make dev` over loopback, n = 20 |
| CSS shipped | 1,894 bytes gzipped | Budget 15 KB, checked in CI |

End-to-end targets in the [latency budget](docs/benchmarks.md#latency-budget-one-live-chat-turn-warm-lambda)
(for example, first visible thought) are design targets until Phase 7 measures them on AWS.

## What you see

```
┌──────────────────────────────────┬──────────────────────────────────┐
│  THOUGHT STREAM (terminal view)  │  ACTION CARDS                    │
│                                  │                                  │
│  > rewriting follow-up question  │  ┌────────────────────────────┐  │
│  > retrieving: "go experience"   │  │ ● Retrieval (C++)   0.4 ms │  │
│  > 4 chunks, top score 0.82      │  └────────────────────────────┘  │
│  > plan: answer + offer meeting  │  ┌────────────────────────────┐  │
│  > calling check_availability    │  │ ● check_availability  8 ms │  │
│                                  │  └────────────────────────────┘  │
│  Answer: ... [1] [2]             │  ┌────────────────────────────┐  │
│                                  │  │ ● Inference (Groq)         │  │
│  [ suggested questions ]         │  │   TTFT 210 ms · 480 tok/s  │  │
│  [ chat input              ⏎ ]   │  │   cached prompt: 1.9k tok  │  │
│                                  │  │ ✓ Judge: grounded          │  │
│                                  │  └────────────────────────────┘  │
└──────────────────────────────────┴──────────────────────────────────┘
```

- **Left — Thought stream:** a terminal-style view of the model's reasoning and the agent's steps,
  streamed in real time over **[Server-Sent Events (SSE)](docs/concepts.md#sse)**, followed by the answer with numbered
  citations.
- **Right — Action cards:** one card per backend step (retrieval, each tool call, inference,
  judge). A card lights up the instant Go starts the step and shows its **measured** duration when
  it finishes. Numbers are always real, never mocked.
- **Suggested questions:** chips for the most common questions. Their answers are generated,
  judge-verified and stored at deploy time, so they appear instantly (`Inference: 0 ms
  (pre-verified)`), a direct contrast with live answers.

Both panes are driven by a single SSE stream with typed events:

| Event | Drives | Payload |
|---|---|---|
| `thought` | left pane | reasoning / step text |
| `token` | left pane (answer) | answer text chunk |
| `citation` | both | chunk id, source title, link |
| `step_start` | right pane | step id, kind, label |
| `step_end` | right pane | step id, duration, provider, token counts, status |
| `judge` | right pane | verdict, unsupported claims |
| `done` / `error` | both | totals / error code |

## Architecture

Stack, model routing, request lifecycle, performance, retrieval, invitations, evaluation,
privacy, testing, cost, repository layout, open decisions and known risks are in
[architecture in detail](docs/architecture.md).

## Local development

Requires Go, a C++17 compiler, CMake and [Ollama](https://ollama.com) running natively.
`make tools` fetches pinned, checksum-verified copies of the
[Tailwind CSS standalone CLI](https://tailwindcss.com/docs/installation/tailwind-cli) and Deno
into `build/tools` (Tailwind alone is about 100 MB; check disk space first).

```bash
cp .env.example .env         # local settings; never secrets
ollama serve                 # start Ollama (stop with Ctrl+C or `systemctl stop ollama`)
ollama pull qwen3:4b         # ~2.6 GB, plus nomic-embed-text (~0.3 GB) for embeddings
make test                    # C++, Go, JS and shell tests, CSS checks
make dev                     # serve the page and /chat on 127.0.0.1:8787 over Ollama
make dev DEV_FLAGS=-invitee  # the same, as an invitee (private chunks and action tools)
make help                    # every target, including the live smokes and evals
```

On a CPU, Ollama answers take 30–100 s. For realistic speed, `CHAT_PROVIDER=groq` uses Groq
with the key read from SSM through your AWS SSO session (`aws sso login` first).
