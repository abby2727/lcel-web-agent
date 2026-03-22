# Client (Next.js frontend)

This folder is the **Next.js App Router UI** for [Search V1 (LCEL Web Agent)](../README.md). It is a chat-style page that sends each question to the agent API and shows the answer plus source links.

## What this part of the project is for

- **Give users a simple interface** — Type a question (at least 5 characters), send, and see assistant replies with optional **Sources** when the backend used the web path.
- **Talk to a separate backend** — The app does not embed the LLM; it `POST`s `{ q }` to `${NEXT_PUBLIC_API_URL}/search` (see `src/lib/config.ts` and `src/app/page.tsx`).
- **Keep “memory” in the browser only** — Chat history is React state on the client. Each request is independent on the server; there is no server-side session store for the conversation.

## How this relates to LangChain

This **client does not import LangChain**. The browser only sees a normal REST call: `POST` JSON `{ q }`, JSON back `{ answer, sources }`. All **LangChain / LCEL** logic lives in **`../agent`**: routing, Tavily-backed web steps, LLM calls, and Zod validation happen on the server. If you want to understand **what LangChain is for** and **why LCEL is used**, read the **[root README](../README.md)** (section *Why LangChain, LCEL, Tavily, and Zod?*) and **[agent/README.md](../agent/README.md)**.

## Learnings this codebase exposes

| Topic | Where it shows up |
| ----- | ----------------- |
| **BFF vs direct API call** | `page.tsx` — The browser calls the agent URL directly; you must align `NEXT_PUBLIC_API_URL` with where Express listens and `ALLOWED_ORIGIN` on the agent with this app’s origin (e.g. `http://localhost:3000`). |
| **Public env in Next.js** | `src/lib/config.ts` — Only `NEXT_PUBLIC_*` variables are exposed to the bundle; the base URL must be safe to publish (no secrets). |
| **Typed fetch + JSON** | `page.tsx` — `SearchResponse` models `{ answer, sources }`; non-OK responses are handled in the UI. |
| **Client-side state for chat UX** | `useState` for `query`, `loading`, and `chat` turns; `useRef` + `useEffect` for scroll-to-latest behavior. |
| **Composable UI** | `src/components/ui/*` — shadcn-style primitives (Button, Card, Input, Textarea, etc.) with Tailwind v4 (`globals.css`, theme). |
| **App Router structure** | `src/app/layout.tsx`, `page.tsx` — Default Next.js 16 layout and a single main page as the product surface. |

For backend behavior (when answers include sources, Tavily requirements, routing rules), see the **[root README](../README.md)** and **`../agent/README.md`**.

### Quick start

```bash
cd client
npm install
# .env: NEXT_PUBLIC_API_URL=http://localhost:5174  (agent base URL, no /search)
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) with the agent already running.
