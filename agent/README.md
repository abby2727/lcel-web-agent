# Agent (backend API)

This folder is the **Express + TypeScript API** for [Search V1 (LCEL Web Agent)](../README.md). It exposes `POST /search`, runs the LangChain LCEL pipeline, and returns `{ answer, sources }` as JSON.

## What this part of the project is for

- **Serve the search agent** — Accept `{ q: string }` (min 5 characters), run heuristic routing, then either a **direct LLM answer** or a **web-grounded** path (Tavily → fetch pages → summarize → compose answer with URLs).
- **Stay separate from the UI** — The Next.js app in `../client` calls this service over HTTP; CORS is limited to `ALLOWED_ORIGIN` so only your frontend origin can call the API from the browser.
- **Keep secrets server-side** — API keys for OpenAI, Gemini, Groq, and Tavily live in `agent/.env`, never in the client.

## LangChain and LCEL here

**LangChain** ([docs](https://js.langchain.com/)) gives this service two main things: **chat model wrappers** (OpenAI, Gemini, Groq) and **runnable composition** via **LCEL**.

- **`@langchain/core`** — `RunnableSequence` chains steps in order; `RunnableBranch` picks the web or direct pipeline after the router; `RunnableLambda` wraps plain async functions (router, validation). See `src/search_tool/searchChain.ts`.
- **Provider packages** — `@langchain/openai`, `@langchain/google-genai`, `@langchain/groq` implement the chat APIs used in `src/shared/models.ts` for invokes and streaming-style message lists elsewhere in the pipeline.

**Why not only raw `fetch` to an LLM?** You could, but you would reimplement provider-specific payloads and lose a uniform **invoke** surface. The bigger win here is **LCEL**: the search flow is a small explicit graph (route → branch → validate) instead of nested conditionals scattered across the codebase.

**Tavily** is *not* part of LangChain in this repo—it is called directly from `src/utils/webSearch.ts` with `fetch`. LangChain orchestrates everything *after* you have search hits and page text.

## Learnings this codebase exposes

| Topic | Where it shows up |
| ----- | ----------------- |
| **LCEL composition** | `src/search_tool/searchChain.ts` — `RunnableSequence`, `RunnableBranch`, and lambdas chain router → branch → validation. |
| **Branching without an extra router LLM** | `src/search_tool/routeStrategy.ts` — Rules and regexes choose `web` vs `direct`. |
| **RAG-like web pipeline** | `src/search_tool/webPipeline.ts` + `src/utils/webSearch.ts`, `openUrl.ts`, `summarize.ts` — retrieve, load text, summarize, then answer with citations. |
| **Schema-first HTTP API** | `src/routes/search_lcel.ts` + `src/utils/schemas.ts` — Zod parses input; output shape is guarded before respond. |
| **Structured output + repair** | `src/search_tool/finalValidate.ts` — `SearchAnswerSchema.safeParse`; optional LLM repair if the first pass does not match `{ answer, sources }`. |
| **Multi-provider LLMs** | `src/shared/models.ts` — Switch OpenAI / Gemini / Groq via `MODEL_PROVIDER` and env keys. |
| **Validated env (partial)** | `src/shared/env.ts` — Zod parses LLM/Tavily-related variables with defaults; Tavily usage checks `TAVILY_API_KEY`. |
| **Express basics** | `src/index.ts` — `dotenv`, CORS, `express.json()`, mount `/search`. |

For install steps, `.env` variables, and end-to-end data flow, see the **[root README](../README.md)**.

### Quick start

```bash
cd agent
npm install
cp .env.example .env   # fill keys
npm run dev
```

Default port in `.env.example` is **5174**.
