# NEXUS — Local AI Interface

> A single-file, zero-install AI chat interface with a full agent toolkit, running entirely in your browser.

---

## Table of Contents

1. [What is AI (LLM)?](#what-is-ai-llm)
2. [What is an AI Agent?](#what-is-an-ai-agent)
3. [The Tool-First Architecture](#the-tool-first-architecture)
4. [Project Overview](#project-overview)
5. [Features at a Glance](#features-at-a-glance)
6. [Supported Models & Providers](#supported-models--providers)
7. [Tool Registry](#tool-registry)
8. [Slash Commands](#slash-commands)
9. [Agent Mode](#agent-mode)
10. [RAG (Retrieval-Augmented Generation)](#rag-retrieval-augmented-generation)
11. [MCP — External Tool Integration](#mcp--external-tool-integration)
12. [Getting Started](#getting-started)
13. [Architecture Deep-Dive](#architecture-deep-dive)
14. [Performance Optimizations (v41)](#performance-optimizations-v41)
15. [Privacy & Security](#privacy--security)
16. [File Structure](#file-structure)
17. [Roadmap](#roadmap)

---

## What is AI (LLM)?

An **LLM (Large Language Model)** is a type of artificial intelligence trained on massive amounts of text data — books, websites, code, conversations — using a technique called *deep learning*. During training, the model learns statistical patterns in language: which words follow which, how ideas connect, and how to generate coherent, contextually appropriate text.

When you send a message to an LLM, it doesn't "look up" an answer. Instead, it predicts the most statistically plausible continuation of your input, token by token. A **token** is roughly 3–4 characters of text. Models like GPT-4, Claude, and Llama process your entire conversation as a flat sequence of tokens inside a fixed-size **context window** — everything outside that window is invisible to the model.

### Key LLM properties relevant to this project

| Property | What it means | How NEXUS handles it |
|---|---|---|
| **Context window** | Maximum tokens the model can see at once | Auto-trims to last 6 messages; bars show usage % |
| **Temperature** | Randomness/creativity (0 = deterministic, 2 = chaotic) | Adjustable slider in sidebar |
| **Max tokens** | Maximum length of the model's reply | Adjustable slider; capped per-provider |
| **System prompt** | Hidden instruction prepended to every conversation | Editable textarea, cached to avoid rebuild overhead |
| **Streaming** | Model outputs tokens as they're generated, not all at once | All streaming paths use SSE/chunked responses |

### Local vs. Cloud LLMs

| | Local (Ollama, LM Studio…) | Cloud (OpenAI, Anthropic, Groq…) |
|---|---|---|
| **Privacy** | 100% — data never leaves your machine | Data sent to provider's servers |
| **Cost** | Free after hardware | Pay per token |
| **Speed** | Depends on your GPU/CPU | Typically faster at scale |
| **Model size** | Limited by your VRAM | Frontier models available |

NEXUS supports both categories from one interface.

---

## What is an AI Agent?

A standard LLM call is a **single round-trip**: you send text, the model replies, done. An **AI agent** breaks this model by giving the LLM access to **tools** — functions it can invoke to interact with the world — and running it in a **loop** until it has enough information to produce a final answer.

```
User message
     │
     ▼
  LLM call ──► "I need to check the weather"
     │
     ▼
Tool execution ──► weather("London") ──► {temp: 18°C, condition: "Cloudy"}
     │
     ▼
  LLM call ──► "London is currently 18°C and cloudy..."
     │
     ▼
Final answer to user
```

The loop continues until the model produces plain text (no tool call). Each iteration is called a **step**. Agents can fail in several ways:

- **Infinite loops** — model keeps calling tools without concluding
- **Token explosion** — raw tool outputs injected into context balloon the prompt size
- **Hallucinated tool calls** — model invents tool names that don't exist
- **Slow responses** — every step is an API call with latency

NEXUS's architecture is designed to prevent all of these.

---

## The Tool-First Architecture

Most agent implementations are **LLM-first**: every request goes through the LLM, which decides whether to use a tool, which tool to use, what arguments to pass, and then interprets the result. This is expensive, slow, and wastes tokens on tasks that don't need reasoning.

NEXUS v41 implements a **Tool-First Hybrid Architecture**:

```
                    ┌──────────────────────────────────┐
  User Input ──────►│   detectIntent()  (pure JS)      │
                    │   Zero LLM cost, zero latency    |
                    └──────────────┬───────────────────┘
                                   │
               ┌───────────────────┼───────────────────┐
               │                   │                   │
               ▼                   ▼                   ▼
         intent=chat         intent=tool          intent=agent
               │            directExec=true       complex task
               │                   │                   │
               ▼                   ▼                   ▼
        Stream directly      Run tool directly    Agent loop
        (0 tool manifest)   ──► compress output  (max 2-3 steps)
        1 LLM call          ──► 0 or 1 LLM call  minimal manifest
```

### Fast Mode (default for most requests)

For simple queries — weather, URL fetch, YouTube, calculator, search — NEXUS:

1. Detects intent with a pure-JS regex classifier (`routeIntent`)
2. Executes the tool directly, no LLM needed for routing
3. Compresses the tool output to ≤1,200 chars (`compressToolOutput`)
4. Makes **exactly one** LLM call to synthesize a human-readable answer
5. For self-formatting tools (datetime, calculator, random, dictionary, unit converter) — **zero synthesis calls**. Raw result renders directly.

Total: **1 tool call + 0–1 LLM calls**.

### Agent Mode (complex / multi-step tasks)

For coding tasks, file editing, and multi-step reasoning:

1. Intent is classified as `agent`
2. Only the **relevant tool subset** is injected into the system prompt (not all 18 tools)
3. Tool descriptions use **1-line slim format** (~150 tokens vs. ~2,000 tokens in LLM-first designs)
4. Loop is capped at **2–3 LLM iterations max**
5. Tool outputs are compressed before re-injection into context
6. System prompt is **cached** — rebuilt only when the user changes it

---

## Project Overview

**NEXUS** is a self-contained, single HTML file (< 500 KB) that functions as a full-featured AI chat client and agent platform. No npm, no bundler, no server, no installation. Open the file in any modern browser and start chatting.

```
nexus-v41.html    ← the entire application
```

It connects to local AI servers (Ollama, LM Studio, Jan, llama.cpp) via direct HTTP calls, and to cloud providers (Anthropic, OpenAI, Groq, OpenRouter) via their APIs. API keys are stored locally in your browser with lightweight obfuscation — they never leave your machine except in the API call itself.

### Why a single file?

- **Zero attack surface** — no supply chain, no npm packages, no CDN dependencies at runtime (only fonts and a few JS libraries loaded once on first use)
- **Portable** — put it on a USB drive, share it over LAN, run it offline
- **Auditable** — the entire application is readable source code with no build step obfuscation

---

## Features at a Glance

| Category | Features |
|---|---|
| **Models** | Local (auto-scan + manual), Online (API key), multi-model switching |
| **Chat** | Streaming, markdown rendering, code highlighting, copy/regenerate/TTS |
| **Agent** | Tool-calling loop, intent routing, tool subset injection, output compression |
| **Tools** | 18 built-in tools (see Tool Registry below) |
| **Slash Commands** | 14 typed shortcuts for direct tool access |
| **Vision** | Camera capture, screen share, image file upload → vision models |
| **Code Interpreter** | JS (native), Python (Pyodide WASM), Bash (simulated) in-browser |
| **Web Search** | DuckDuckGo (free) or Serper/Google (API key), native Anthropic search |
| **RAG** | Upload docs → chunk → cosine-similarity retrieval → grounded answers |
| **arXiv** | Search academic papers, inject abstract + metadata into context |
| **MCP** | Add any HTTP API as a custom tool the agent can call |
| **Files** | In-browser virtual filesystem (OPFS + localStorage fallback) |
| **Charts** | Plotly.js interactive charts rendered inline in chat |
| **Diagrams** | Mermaid.js flowcharts, sequence diagrams, Gantt, ERD, etc. |
| **Images** | AI image generation via Pollinations API (no key needed) |
| **Memory** | Persistent per-session memory injected into every system prompt |
| **History** | Chat history saved to localStorage, rename/delete/reload |
| **Themes** | Dark (default) and Blue themes |
| **Mobile** | Responsive layout with slide-out sidebar |

---

## Supported Models & Providers

### Local (self-hosted)

NEXUS auto-scans 11 common ports on startup:

| Server | Default Port | Protocol |
|---|---|---|
| **Ollama** | 11434 | Ollama native API |
| **LM Studio** | 1234 | OpenAI-compatible |
| **Jan** | 1337 | OpenAI-compatible |
| **Msty** | 10000 | OpenAI-compatible |
| **llama.cpp** | 8080 | OpenAI-compatible |
| **LocalAI** | 8080, 8000 | OpenAI-compatible |
| **Open WebUI** | 3000 | OpenAI-compatible |
| **Oobabooga** | 5000, 5001 | OpenAI-compatible |

Any OpenAI-compatible server can also be added manually.

### Cloud (API key required)

| Provider | Models | Notes |
|---|---|---|
| **OpenAI** | GPT-4o, GPT-4.1, GPT-4, GPT-3.5, o1/o3/o4 | Streaming, vision |
| **Anthropic** | Claude 3/4 family | Native web search, vision |
| **Groq** | Llama 3, Mixtral, Gemma | Very fast inference |
| **OpenRouter** | 200+ models | Single API key, many providers |
| **Custom** | Any OpenAI-compatible endpoint | Self-hosted or third-party |

### Model capability auto-detection

NEXUS automatically detects and flags capabilities per model:

- `VIS` — vision (image input) supported
- `TOOLS` — function/tool calling via API
- `128K` / `200K` — long context window

---

## Tool Registry

NEXUS ships with 18 built-in tools available to the agent. Tools are pure-JS functions — no external runtime required except where noted.

| Tool | Trigger keywords | What it does |
|---|---|---|
| `youtube` | YouTube URL, "summarize video" | Fetches transcript + metadata via Invidious API |
| `calculator` | "calculate", "compute", "evaluate" | Scientific calculator: trig, log, powers, constants |
| `datetime` | "what time", "current date", "today" | Current date/time in user's local timezone |
| `weather` | "weather", "forecast", "temperature" | Real-time weather + 3-day forecast via Open-Meteo (no key) |
| `unit_converter` | "convert", "how many km", "in pounds" | Length, weight, temp, volume, speed, data size |
| `dictionary` | "define", "what does X mean" | Definition, pronunciation, etymology via Free Dictionary API |
| `random` | "random", "roll dice", "flip coin" | Dice, coin, integer range |
| `filesystem` | "create file", "write to", "save as" | In-browser virtual filesystem (OPFS + localStorage) |
| `code_executor` | "run this", "execute", `/code` | JS (native), Python (Pyodide WASM), Bash (simulated) |
| `web` | URLs, "search for", "fetch", "latest" | Web search (DDG/Serper) + full-page fetch with proxy |
| `chart_generator` | "chart", "graph", "plot", "visualize" | Plotly.js interactive charts from CSV data |
| `diagram_generator` | "flowchart", "diagram", "sequence", "Gantt" | Mermaid.js diagrams rendered inline |
| `image_generator` | "generate image", "draw", `/genimage` | AI images via Pollinations API (free, no key) |
| `view_file` | Agent file read | Read stored file with line numbers and range support |
| `str_replace_file` | Agent file edit | Precise find-and-replace in stored files |
| `create_file` | Agent file write | Create or overwrite files in the browser store |
| `present_file` | Agent file surface | Render download card for a completed file |
| *(MCP tools)* | User-defined | Any HTTP API added via the MCP panel |

---

## Slash Commands

Type `/` in the input box for autocomplete. All slash commands bypass the agent loop and execute the tool directly.

| Command | Description | Example |
|---|---|---|
| `/weather [city]` | Current weather + 3-day forecast | `/weather Tokyo` |
| `/calc [expression]` | Scientific calculator | `/calc Math.sqrt(144) * PI` |
| `/datetime` | Current date, time & timezone | `/datetime` |
| `/convert [amount] [from] [to]` | Unit converter | `/convert 100 km miles` |
| `/define [word]` | Dictionary lookup | `/define ephemeral` |
| `/random [type] [args]` | Random number / dice / coin / uuid | `/random dice 20` |
| `/search [query]` | Web search | `/search latest AI news` |
| `/read [url]` | Fetch & summarise a webpage | `/read https://example.com` |
| `/youtube [url]` | Transcript + summary | `/youtube https://youtu.be/...` |
| `/chart [type] [title] \| csv` | Render a chart inline | `/chart bar Sales \| Month,Val\nJan,12000` |
| `/genimage [prompt]` | Generate an AI image | `/genimage a sunset over mountains, oil painting` |
| `/diagram [prompt]` | Generate a Mermaid diagram | `/diagram login flow for a web app` |
| `/code [language] <code>` | Execute code | `/code python print("Hello")` |
| `/help` | Show all commands | `/help` |

---

## Agent Mode

Enable via the **Agent** toggle in the composer toolbar.

When agent mode is on, the model can autonomously decide to call tools, inspect results, and call more tools — up to a configurable cap. NEXUS enforces strict limits to prevent runaway loops:

- **Max agent steps**: 4 (hard cap)
- **Max tool calls per request**: 2
- **Duplicate call guard**: identical tool+args combination never called twice
- **Post-tool synthesis**: after a tool result, a dedicated synthesis call prevents local models from re-entering JSON/tool-call mode

### Intent routing

Before any LLM call, NEXUS classifies the user's message with a pure-JS intent detector:

```
"what's the weather in Paris"  →  intent: weather  →  directExec=true
"https://example.com"          →  intent: fetch    →  directExec=false
"create a Python snake game"   →  intent: file     →  agent loop
"explain recursion"            →  intent: chat     →  stream directly
```

This means **the majority of requests never go through an agent loop at all**.

---

## RAG (Retrieval-Augmented Generation)

RAG lets you upload your own documents and have the model answer questions grounded in their content — not just its training data.

### How it works

1. **Upload** — drag or paste one or more text documents into the RAG panel
2. **Chunking** — documents are split into overlapping chunks (configurable size: 256–1024 tokens)
3. **Indexing** — each chunk gets a TF-IDF-style embedding stored in memory
4. **Retrieval** — on each message, NEXUS computes cosine similarity between your query and all chunks, retrieves the top-K most relevant ones (configurable: 1–8)
5. **Grounding** — retrieved chunks are prepended to the system prompt as `RETRIEVED KNOWLEDGE`, with source document names cited
6. **Citation** — after the model responds, matching chunks are shown below the message as expandable citations

No external vector database needed. Everything runs in-browser.

---

## MCP — External Tool Integration

The **MCP (Model Context Protocol) panel** lets you register any HTTP API as a callable tool — no code changes required.

To add a tool:
1. Open the **Tools → MCP** tab
2. Click **Add External Tool**
3. Fill in: name, endpoint URL, HTTP method, JSON Schema parameters, optional auth header
4. Click **Test** to verify the endpoint works
5. Click **Add Tool** — the agent can now call it by name

MCP tools appear in the agent's tool manifest automatically. You can enable/disable individual tools without deleting them, and test them with arbitrary arguments from the panel.

---

## Getting Started

### Option 1: Local AI (no API key, 100% private)

1. Install [Ollama](https://ollama.com) and pull a model:
   ```bash
   ollama pull llama3.2
   ollama pull llava          # for vision support
   ```
2. Open `nexus-v41.html` in your browser
3. Click **Scan for local AI** — Ollama is detected automatically
4. Start chatting

### Option 2: Cloud AI

1. Open `nexus-v41.html` in your browser
2. Click **Add online AI**
3. Select your provider (OpenAI, Anthropic, Groq, OpenRouter, or Custom)
4. Paste your API key → Connect
5. Start chatting

### Option 3: Both at once

Add as many models as you like. Switch between them by clicking the model name in the sidebar. NEXUS remembers all connected models across sessions (API keys are stored with lightweight XOR obfuscation in localStorage).

### Web search setup (optional)

Web search works out of the box via DuckDuckGo (limited but no key needed). For full Google results:

1. Get a free API key from [serper.dev](https://serper.dev) (2,500 queries/month free)
2. Paste it into **Controls → Serper API Key**

---

## Architecture Deep-Dive

### Request lifecycle

```
sendMessage(text)
     │
     ├─ slash command? ──────────────────► direct tool call → render
     │
     ├─ agent mode OFF
     │       ├─ local model? ───────────► streamLocal() → SSE stream
     │       └─ online model? ──────────► streamOnline() → SSE stream
     │
     └─ agent mode ON
             │
             ▼
         agentLoop(text, files, bubble)
             │
             ├─ routeIntent(text)  ← pure JS, 0ms, 0 tokens
             │
             ├─ intent=chat ──────────────► streamLocal/streamOnline (no manifest)
             │
             ├─ intent=X, directExec=true
             │       ├─ runTool(toolName, args)
             │       ├─ compressToolOutput(result)  ← ≤1200 chars
             │       ├─ no-synthesis tool? ─────────► return raw result
             │       └─ synthesisLLM(compressed) ──► 1 LLM call
             │
             └─ intent=X, directExec=false (agent path)
                     ├─ buildAgentSystemPrompt(injectedTools)  ← slim manifest
                     ├─ while (step < MAX_AGENT_STEPS):
                     │       ├─ callLLM(workingMsgs, sysPrompt)
                     │       ├─ tryParseToolCall(response)
                     │       ├─ tool call? → runTool → compressToolOutput → append
                     │       └─ no tool call? → final answer → break
                     └─ return finalText
```

### Key modules

| Module | Lines (approx.) | Responsibility |
|---|---|---|
| State & model registry | ~100 | `state`, `MODEL_CAPS_DB`, `CTX_WINDOWS` |
| Model management | ~300 | Scan, add, activate, probe endpoints |
| Intent router | ~60 | `routeIntent`, `INTENT_TOOLS`, `getToolsForIntent` |
| Tool registry | ~2,000 | 18 tool definitions with `execute()` functions |
| Agent loop | ~200 | `agentLoop`, `synthesisLLM`, `callLLM` |
| Streaming | ~300 | `streamLocal`, `streamOnline` (Ollama + OpenAI + Anthropic SSE) |
| Web search | ~200 | DDG + Serper, caching, dedup |
| RAG | ~300 | Chunking, TF-IDF, cosine similarity, citations |
| MCP | ~200 | Tool registration, HTTP execution, auth |
| Filesystem | ~400 | OPFS + localStorage VFS, 4 agent file tools |
| UI / rendering | ~3,000 | Chat, markdown, sidebar, modals, toolbar |

---

## Performance Optimizations (v41)

v41 introduced the Tool-First Hybrid Architecture. Compared to v40:

| Metric | v40 (LLM-first) | v41 (Tool-first) |
|---|---|---|
| **System prompt size** | ~2,000 tokens (full manifest every call) | ~150 tokens (1-line slim, cached) |
| **Simple tool request** | 1 routing call + 1 tool call + 1 synthesis = 3 LLM calls | 0 routing + 1 tool + 0–1 synthesis = 0–1 LLM calls |
| **Tool output in context** | Raw (can be 5,000+ tokens) | Compressed to ≤1,200 chars (~300 tokens) |
| **Multi-tool planner** | Extra LLM call for detection + per-tool LLM arg extraction | Removed entirely |
| **Chat-only messages** | Full manifest injected | No manifest, stream directly |
| **Datetime/calc/random** | Tool + synthesis LLM call | Tool only — raw result rendered |
| **System prompt rebuilds** | Every request | Cached; rebuilt only on user change |

### `compressToolOutput` strategies

| Tool | Strategy | Max output |
|---|---|---|
| chart/diagram/image | Return short `[RENDERED] ✓` only | 300 chars |
| YouTube | Head 12 lines + tail 6 lines | 1,200 chars |
| Web search | First 4 result blocks | 1,200 chars |
| Filesystem/view_file | Hard truncate with char count | 1,200 chars |
| Everything else | Generic truncation | 1,200 chars |

---

## Privacy & Security

| Concern | How NEXUS handles it |
|---|---|
| **API keys at rest** | XOR-obfuscated in localStorage (not plaintext, not cryptographic — security through inconvenience) |
| **API keys in transit** | Sent only in the `Authorization` header of the provider API call, never logged elsewhere |
| **Chat history** | localStorage only, never uploaded anywhere |
| **Local model data** | Never leaves your machine (direct HTTP to localhost) |
| **Code execution** | Runs in browser sandbox. Python via Pyodide WASM (isolated). No native process access. |
| **File store** | OPFS (browser Origin Private File System) or localStorage. Never synced to any server. |
| **Web search** | Query sent to DuckDuckGo or Serper — no other data included |

> **Note**: For maximum privacy, use a local model. Cloud providers (OpenAI, Anthropic, etc.) receive your messages per their own privacy policies.

---

## File Structure

```
nexus-v41.html
├── <head>
│   ├── Google Fonts (Space Mono, Geist)
│   ├── marked.js    — markdown rendering
│   ├── Plotly.js    — chart rendering
│   └── Mermaid.js   — diagram rendering
│
├── <style>           — ~2,800 lines of CSS
│   ├── CSS tokens & reset
│   ├── Sidebar & model panel
│   ├── Chat messages & bubbles
│   ├── Tool panels (arXiv, Vision, Chart, Diagram, MCP, RAG, Files)
│   ├── Modals, toasts, generating bar
│   └── Dark & Blue themes + responsive mobile
│
├── <body>            — HTML structure (~400 lines)
│   ├── Sidebar (logo, history, memory, models, controls, tools)
│   ├── Main (header, chat, input area, composer toolbar)
│   └── Modals (add local AI, add online AI, add MCP tool)
│
└── <script>          — ~8,000 lines of JS
    ├── Configuration & state
    ├── Model capability registry (MODEL_CAPS_DB)
    ├── Token estimator & context bar
    ├── Stream renderers (buffered, 25fps DOM updates)
    ├── Chat history (localStorage)
    ├── Model management (scan, add, probe, activate)
    ├── Web search (DDG, Serper, caching)
    ├── Intent router (routeIntent, INTENT_TOOLS)
    ├── Tool registry (18 tools × execute())
    ├── Agent infrastructure
    │   ├── compressToolOutput()
    │   ├── buildMinimalSystemPrompt() + cache
    │   ├── buildAgentSystemPrompt()  slim manifest
    │   ├── synthesisLLM()
    │   ├── callLLM()
    │   ├── tryParseToolCall()
    │   └── agentLoop()
    ├── Streaming (streamLocal, streamOnline)
    ├── UI tools (arXiv, Vision, Chart, Diagram)
    ├── RAG (chunking, TF-IDF, cosine similarity, citations)
    ├── MCP (registration, execution, panel)
    ├── Filesystem VFS (OPFS + localStorage)
    ├── Memory store
    ├── Settings store
    └── Slash command autocomplete
```

---

## Roadmap

- [ ] Conversation summarization for infinite context
- [ ] Multi-modal output (model-generated images in response)
- [ ] Plugin system for community tool packs
- [ ] Export chat as PDF / Markdown
- [ ] Shared sessions via WebRTC
- [ ] Electron wrapper for desktop notifications + system tray
- [ ] Voice output with streaming TTS
- [ ] NEXUS API server mode (expose agent over HTTP)

---

## License

MIT — do whatever you want, attribution appreciated.

---

<div align="center">
  <sub>NEXUS v4.1 · Single-file · Zero-install · Tool-first</sub>
</div>
