<div align="center">

# 🎯 Document-Retrieval

### An agentic RAG chatbot that fetches its own missing knowledge

Most RAG apps search what they already have. This one decides when it *doesn't* have enough — and goes and gets it, live, from YouTube.

[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)](https://react.dev/)
[![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white)](https://langchain-ai.github.io/langgraph/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![pgvector](https://img.shields.io/badge/pgvector-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)

[View Demo](#-demo) · [Report Bug](../../issues) · [Request Feature](../../issues)

</div>

---

## 💡 The Idea

Ask a typical chatbot about a video it has never seen, and it either hallucinates an answer or tells you it can't help. **Document-Retrieval does neither.**

It's an agent that reasons about *what kind of gap* it's facing and picks the right move:

- **"I know exactly which video."** → search that video's transcript.
- **"I know the topic, not the source."** → search semantically across every indexed video.
- **"Nobody has indexed this yet."** → scrape the transcript, embed it, and answer once it's ready.

The result is a chatbot that behaves less like a lookup table and more like a research assistant — one that can go build its own evidence base mid-conversation.

## 🖼️ Demo

<div align="center">
<!-- Replace with an actual screen recording or screenshot of the chat UI -->
<img src="./client/public/demo-screenshot.png" alt="Document-Retrieval chat interface" width="800"/>
<br/>
<sub>💬 Ask a question → 🔍 agent picks a tool → 🌐 scrapes if needed → ✅ answers with sources</sub>
</div>

## ⚙️ How It Works

```mermaid
flowchart LR
    U[👤 User Question] --> A{🧠 LangGraph Agent}
    A -->|Video already known| R[📄 retrieveTool]
    A -->|Topic known, source unclear| S[🔎 retrieveSimilarVideosTool]
    A -->|Not indexed yet| T[🕸️ triggerYoutubeVideoScrapeTool]
    T --> BD[BrightData Scraper]
    BD -->|async job| WH[🔗 Webhook Callback]
    WH --> EMB[Chunk + Embed]
    EMB --> DB[(🗄️ pgvector / PostgreSQL)]
    R --> DB
    S --> DB
    DB --> ANS[✅ Answer to User]
```

**The key design decision:** scraping a live YouTube transcript with BrightData isn't instant — it's an external job. Instead of blocking the chat while it runs, the agent hands the job off, tells the user it's fetching the source, and resumes the conversation the moment a webhook confirms the transcript has been chunked, embedded, and indexed. Nothing about the UX breaks while the pipeline works in the background.

## 🧰 Agent Tools

The agent's intelligence isn't in a single giant prompt — it's in a small, well-scoped toolset that maps directly onto the three ways a question can fail:

| Tool | Job | When it fires |
|---|---|---|
| `retrieveTool` | Searches a specific video's transcript | User already knows the video |
| `retrieveSimilarVideosTool` | Semantic search across every indexed video | User knows the topic, not the source |
| `triggerYoutubeVideoScrapeTool` | Scrapes a new transcript and indexes it | The corpus is too thin to answer responsibly |

```ts
const tools = [
  retrieveTool,
  retrieveSimilarVideosTool,
  triggerYoutubeVideoScrapeTool,
];

const agent = createReactAgent({
  llm: claude,
  tools,
  checkpointer: new MemorySaver(),
});
```

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | React · TypeScript · Vite |
| **Agent Orchestration** | LangGraph (`createReactAgent`) |
| **LLM** | Claude |
| **Web Scraping** | BrightData |
| **Embeddings & Chunking** | LangChain · `RecursiveCharacterTextSplitter` |
| **Vector Store** | PostgreSQL + `pgvector` |
| **Async Pipeline** | Webhook-based scrape → index handoff |

**Chunking strategy:** transcripts are split with a chunk size of `1000` and an overlap of `200` characters — conservative enough to preserve context across chunk boundaries without exploding the index.

```ts
const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,
  chunkOverlap: 200,
});

const vectorStore = await PGVectorStore.initialize(embeddings, {
  postgresConnectionOptions,
});
```

## 📁 Project Structure

```
Document-Retrieval/
├── client/                  # React + TypeScript + Vite frontend
│   ├── public/
│   │   └── vite.svg
│   ├── src/
│   │   ├── assets/
│   │   ├── App.tsx
│   │   ├── App.css
│   │   ├── index.css
│   │   ├── main.tsx
│   │   └── vite-env.d.ts
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   └── eslint.config.js
├── server/                  # Agent, tools, retrieval & webhook logic
└── README.md
```

## 🚀 Getting Started

### Prerequisites

- Node.js ≥ 18
- PostgreSQL with the `pgvector` extension enabled
- API keys for: Claude (Anthropic), BrightData

### Installation

```bash
# Clone the repo
git clone https://github.com/riyav1606/Document-Retrieval.git
cd Document-Retrieval

# Install frontend dependencies
cd client
npm install

# Install backend dependencies
cd ../server
npm install
```

### Environment Variables

Create a `.env` file in `server/` with:

```env
ANTHROPIC_API_KEY=your_key_here
BRIGHTDATA_API_KEY=your_key_here
DATABASE_URL=postgresql://user:password@localhost:5432/document_retrieval
WEBHOOK_URL=your_public_webhook_url
```

### Run Locally

```bash
# Terminal 1 — backend
cd server
npm run dev

# Terminal 2 — frontend
cd client
npm run dev
```

Then open `http://localhost:5173`.

## 🗺️ Roadmap

- [ ] Support platforms beyond YouTube (podcasts, blog transcripts)
- [ ] Streaming responses while a scrape is in progress
- [ ] Multi-user session isolation
- [ ] Dockerized deployment
- [ ] Test coverage for the tool-selection logic

## 🎓 What This Project Demonstrates

- Designing **stateful, tool-using agents** with LangGraph rather than single-shot prompting
- Building **async, webhook-driven pipelines** so long-running jobs never block a conversational UX
- Practical **RAG engineering**: chunking strategy, embeddings, and vector search with pgvector
- Thinking about AI products as systems with an **acquisition layer**, not just a response layer

## 📄 License

Distributed under the MIT License. See `LICENSE` for details.

## 📬 Contact

**Riya** — [GitHub](https://github.com/riyav1606)

Project Link: [github.com/riyav1606/Document-Retrieval](https://github.com/riyav1606/Document-Retrieval)

---

<div align="center">
<sub>Built to explore what an AI app looks like when it can go find its own evidence.</sub>
</div>
