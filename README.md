# LCEL Web Agent

A full-stack AI search assistant that answers user questions with **optional real web grounding**. The backend uses **LangChain Expression Language (LCEL)** to chain steps: a lightweight router decides whether to answer **directly** from the LLM or to run a **web pipeline** (search → fetch pages → summarize → compose an answer with **source URLs**). Request bodies and the final API shape are validated with **Zod**.

<!-- > **Note:** The UI has **no chat memory** across turns in the backend; each request is independent. The browser keeps a local chat history only for display. -->

<!-- > **Web search requires a [Tavily](https://tavily.com) API key**; without it, routes that choose the web path will fail when search runs. -->

---

## Why LangChain, LCEL, Tavily, and Zod?

### What is LangChain?

**LangChain** is an ecosystem for building LLM-powered applications in TypeScript (and Python). It provides **abstractions for models, messages, and composable workflows** so you are not stuck hand-wiring every step. In this repo we use the **JavaScript/TypeScript** packages (`@langchain/core` plus provider integrations like `@langchain/openai`), not the full “agents with dozens of tools” stack—here the focus is **orchestration** and **chat models**.

### Why LangChain in this project?

You _could_ call OpenAI (or Gemini, Groq) with plain `fetch`. LangChain is still useful here because:

1. **One API for several providers** — `ChatOpenAI`, `ChatGoogleGenerativeAI`, and `ChatGroq` share a similar invoke pattern (`shared/models.ts`), so switching `MODEL_PROVIDER` is localized.
2. **LCEL runnables** — Steps are expressed as **runnables** that can be sequenced and branched (`RunnableSequence`, `RunnableBranch`, `RunnableLambda`), which keeps the web vs direct split explicit and testable (`searchChain.ts`).
3. **Messages and model config** — System/human messages and parameters like temperature are structured consistently across summarization, composition, and repair passes.

So LangChain is not “magic”; it is **structure** for multi-step LLM workflows and multi-vendor chat models.

### What is LCEL?

**LCEL** (LangChain Expression Language) is the **composable runnable** style in `@langchain/core`: you build a pipeline from small units, pipe outputs to inputs, and branch on conditions. In this project the graph is: **router → branch (web path | direct path) → final validation**. That matches how you think about the product (“first decide mode, then run the right pipeline”) without a separate orchestration framework.

### Why Tavily?

**Tavily** is a search API aimed at LLM apps: you send a natural-language query and get ranked web results suitable for grounding. The **web path** uses Tavily to **retrieve** URLs, then the agent **fetches and summarizes** pages locally before writing an answer with **source links**. Alternatives exist (other search APIs, custom crawlers); Tavily keeps retrieval simple and API-driven.

### Why Zod?

**Zod** validates **shapes at runtime**: the HTTP body (`{ q }`), environment-related config in the agent, and the **`{ answer, sources }`** response contract. That catches bad input early and makes the API predictable for the Next.js client and tools like Postman.

---

## What This Project Demonstrates

- **Heuristic routing** — keyword and pattern rules (`routeStrategy.ts`) pick `web` vs `direct` without an extra LLM call for routing.
- **LCEL composition** — `RunnableSequence`, `RunnableBranch`, and `RunnableLambda` wire router → branch → final validation (`searchChain.ts`).
- **Tool-style utilities** — Tavily for search, Node `fetch` + `html-to-text` for page extraction, LLM calls for summarization and final answers (`webPipeline.ts`, `utils/`).
- **Multi-provider LLMs** — switch **OpenAI**, **Google Gemini**, or **Groq** via `MODEL_PROVIDER` (`shared/models.ts`).
- **Structured output guardrails** — `SearchAnswerSchema` ensures `{ answer, sources }`; a repair pass uses the LLM if the first parse fails (`finalValidate.ts`).

---

## Data Flow (End to End)

```
User types a question (min 5 characters in UI)
        │
        ▼
Browser sends POST { q: "..." } to NEXT_PUBLIC_API_URL/search
        │
        ▼
Express (agent/src/index.ts)
  — CORS limited to ALLOWED_ORIGIN
  — JSON body parser
        │
        ▼
POST /search (agent/src/routes/search_lcel.ts)
  — SearchInputSchema.parse(req.body)
  — runSearch(input) → LCEL chain
        │
        ▼
routerStep (agent/src/search_tool/routeStrategy.ts)
  — Validates q, sets mode: "web" | "direct"
        │
        ▼
RunnableBranch
  ├─ mode === "web"  → webPath
  └─ else            → directPath
        │
        ├─ directPath: single Chat model invoke → { answer, sources: [], mode: "direct" }
        │
        └─ webPath (webPipeline.ts):
              webSearch (Tavily API) → top results
              openUrl (fetch + html-to-text) per result → summarize (LLM) per page
              ComposeStep: LLM answers from page summaries → sources = result URLs
        │
        ▼
finalValidateAndPolish (finalValidate.ts)
  — SearchAnswerSchema.safeParse; on failure, LLM "repair" to match schema
        │
        ▼
Express responds 200 with { answer, sources }
        │
        ▼
Next.js UI renders assistant message + source links (page.tsx)
```

---

## Project Structure

```
lcel-web-agent/
├── agent/                         # Express + LangChain LCEL API (default port 5174)
│   ├── src/
│   │   ├── index.ts               # Express app, CORS, mounts /search
│   │   ├── routes/
│   │   │   └── search_lcel.ts     # POST /search → Zod input → runSearch()
│   │   ├── search_tool/
│   │   │   ├── searchChain.ts     # LCEL: router → branch → finalValidate
│   │   │   ├── routeStrategy.ts   # Heuristic web vs direct routing
│   │   │   ├── webPipeline.ts     # Tavily → openUrl → summarize → compose
│   │   │   ├── directPipeline.ts  # Direct LLM answer (no web)
│   │   │   ├── finalValidate.ts   # Zod + optional JSON repair via LLM
│   │   │   └── types.ts           # candidate shape (answer, sources, mode)
│   │   ├── shared/
│   │   │   ├── env.ts             # Zod-validated environment variables
│   │   │   └── models.ts          # ChatOpenAI / ChatGoogleGenerativeAI / ChatGroq
│   │   └── utils/
│   │       ├── schemas.ts         # Zod schemas for I/O and web results
│   │       ├── webSearch.ts       # Tavily HTTP client
│   │       ├── openUrl.ts         # Fetch URL, HTML → plain text
│   │       └── summarize.ts       # LLM summarization of page text
│   ├── .env                       # API keys, PORT, MODEL_PROVIDER, …
│   ├── .env.example
│   ├── package.json
│   └── tsconfig.json
│
└── client/                        # Next.js frontend (default port 3000)
    ├── src/
    │   ├── app/
    │   │   ├── page.tsx           # Chat UI; fetch(API_URL/search)
    │   │   ├── layout.tsx
    │   │   └── globals.css        # Tailwind v4 + theme
    │   ├── components/ui/         # Button, Card, Input, Separator, Textarea (shadcn-style)
    │   └── lib/
    │       ├── config.ts          # NEXT_PUBLIC_API_URL
    │       └── utils.ts           # cn() helper
    ├── .env                       # NEXT_PUBLIC_API_URL
    ├── .env.example
    ├── components.json            # shadcn configuration
    ├── package.json
    └── tsconfig.json
```

---

## Tech Stack

| Layer             | Stack                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------- |
| **API**           | Node.js, Express, `tsx` for dev                                                          |
| **Orchestration** | `@langchain/core` runnables (LCEL)                                                       |
| **LLM**           | `@langchain/openai`, `@langchain/google-genai`, `@langchain/groq`                        |
| **Validation**    | `zod`                                                                                    |
| **Web search**    | Tavily REST API (`utils/webSearch.ts`)                                                   |
| **HTML cleanup**  | `html-to-text`                                                                           |
| **Frontend**      | Next.js 16, React 19, Tailwind CSS v4, Radix primitives, shadcn-style UI, `lucide-react` |

---

## Agent (Backend) Dependencies

Install inside the `agent/` folder (this repo already includes `package.json`; use the steps below to reproduce or understand order).

### 1. Initialize the project (if starting from scratch)

```bash
npm init -y
```

### 2. Install TypeScript and dev tooling (install first — needed to run `.ts` files)

```bash
npm install -D typescript tsx @types/node @types/cors @types/html-to-text
```

| Package               | Why                                                    |
| --------------------- | ------------------------------------------------------ |
| `typescript`          | Type-checking and TS sources                           |
| `tsx`                 | Run TypeScript in dev (`npm run dev` uses `tsx watch`) |
| `@types/node`         | Node.js typings                                        |
| `@types/cors`         | Typings for CORS middleware                            |
| `@types/html-to-text` | Typings for HTML-to-text conversion                    |

### 3. Install runtime dependencies

```bash
npm install express cors dotenv zod html-to-text
npm install @langchain/core @langchain/openai @langchain/google-genai @langchain/groq
npm install @types/express
```

| Package                   | Why                                                                     |
| ------------------------- | ----------------------------------------------------------------------- |
| `express`                 | HTTP server; exposes `POST /search`                                     |
| `cors`                    | Restricts browser origins via `ALLOWED_ORIGIN`                          |
| `dotenv`                  | Loads `.env` (see `import "dotenv/config"` in `index.ts`)               |
| `zod`                     | Validates env (`shared/env.ts`), request body, and `SearchAnswer` shape |
| `html-to-text`            | Strips HTML from fetched pages in `openUrl.ts`                          |
| `@langchain/core`         | `RunnableSequence`, `RunnableBranch`, `RunnableLambda`, messages        |
| `@langchain/openai`       | `ChatOpenAI` when `MODEL_PROVIDER=openai`                               |
| `@langchain/google-genai` | `ChatGoogleGenerativeAI` when `MODEL_PROVIDER=gemini`                   |
| `@langchain/groq`         | `ChatGroq` when `MODEL_PROVIDER=groq`                                   |
| `@types/express`          | Express typings (listed as a runtime dependency in this project)        |

**Also present in this repo’s `package.json`:** `@langchain/classic`, `@google/generative-ai` — typically pulled for compatibility with the LangChain / Google GenAI stack; the application code imports the chat model wrappers above.

### 4. Dev script (already in `package.json`)

```json
"scripts": {
  "dev": "tsx watch src/index.ts"
}
```

### 5. Create `.env` in the agent root

Copy from `.env.example` and fill in keys for your chosen `MODEL_PROVIDER`:

```
MODEL_PROVIDER=gemini
OPENAI_API_KEY=
GOOGLE_API_KEY=
GROQ_API_KEY=
TAVILY_API_KEY=

PORT=5174
ALLOWED_ORIGIN=http://localhost:3000
OPENAI_MODEL=gpt-4o-mini
GEMINI_MODEL=gemini-2.0-flash-lite
GROQ_MODEL=llama-3.1-8b-instant
SEARCH_PROVIDER=tavily
```

Set **one** of the LLM provider API keys according to `MODEL_PROVIDER` (`openai` | `gemini` | `groq`). Set **`TAVILY_API_KEY`** for web search. Match **`ALLOWED_ORIGIN`** to your Next.js dev URL (usually `http://localhost:3000`).

> The sample file may include `RAG_MODEL_PROVIDER`; the validated config in `shared/env.ts` does not read that variable — you can ignore it unless you extend the code.

---

## Client (Frontend) Dependencies

Install inside the `client/` folder.

### 1. Create the Next.js app (if starting from scratch)

```bash
npx create-next-app@latest client
```

Choose TypeScript, Tailwind CSS, App Router, and `src/` directory to align with this project.

### 2. Add shadcn-style UI (if recreating)

```bash
npx shadcn@latest init
npx shadcn@latest add button card input separator textarea
```

### 3. Packages used by this client (`package.json`)

**Dependencies**

| Package                                              | Why                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------------------- |
| `next`                                               | App Router, dev server, production build                            |
| `react` / `react-dom`                                | UI                                                                  |
| `@radix-ui/react-slot` / `@radix-ui/react-separator` | Accessible primitives for composed components                       |
| `class-variance-authority`                           | Variant styling for UI components                                   |
| `clsx` + `tailwind-merge`                            | Conditional classes; merge Tailwind safely (`cn` in `lib/utils.ts`) |
| `lucide-react`                                       | Icons (if used in expanded UI)                                      |

**Dev dependencies**

| Package                                | Why                                     |
| -------------------------------------- | --------------------------------------- |
| `tailwindcss` / `@tailwindcss/postcss` | Tailwind v4                             |
| `tw-animate-css`                       | Animation utilities used with the theme |
| `typescript` / `@types/*`              | TypeScript                              |

### 4. Create `.env` in the client root

Point the browser to your agent base URL (no trailing path — the app appends `/search`):

```
NEXT_PUBLIC_API_URL=http://localhost:5174
```

---

## Getting Started

### 1. Start the agent API

```bash
cd agent
npm install
npm run dev
```

The server listens on the port from `PORT` (default **5174** in `.env.example`).

### 2. Start the client

```bash
cd client
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000), enter a query (at least **5** characters), and press **Send**.

### 3. Quick checks

- **Direct-style questions** (short, no “latest/news/top N” patterns) use the LLM only; **sources** may be empty.
- **Web-style queries** (long text, recent years like 2024–2039, or keywords such as “top”, “best”, “vs”, “price”, “news”, …) trigger Tavily + page fetch + summarization; **sources** lists URLs.

---

## Key Files Reference

| File                                   | Role                                            |
| -------------------------------------- | ----------------------------------------------- |
| `agent/src/search_tool/searchChain.ts` | Exports `runSearch` and the composed LCEL graph |
| `agent/src/shared/models.ts`           | Single place to swap LLM provider               |
| `agent/src/utils/webSearch.ts`         | Tavily integration                              |
| `agent/src/utils/schemas.ts`           | `SearchInputSchema`, `SearchAnswerSchema`, etc. |
| `client/src/lib/config.ts`             | Reads `NEXT_PUBLIC_API_URL`                     |
| `client/src/app/page.tsx`              | `POST` to `${API_URL}/search` with `{ q }`      |

---

## Key Concepts Demonstrated

- **LCEL branching** — `RunnableBranch` selects web vs direct pipeline after a deterministic router.
- **RAG-like web path** — retrieve (Tavily) → load documents (fetch + `html-to-text`) → summarize chunks → generate grounded answer with citations.
- **Schema-first API** — Zod on the wire and for environment configuration.
- **CORS + public API URL** — the client calls the agent directly; ensure `ALLOWED_ORIGIN` matches your frontend origin in development and production.
