# COGNIVAULT

**A local-first AI assistant that reads your PDFs, images, and Word docs, and remembers them — across every conversation you have with it, forever.**

Upload a document in one chat, ask about it in a completely different chat weeks later, and it answers correctly. Mention an offhand personal preference in passing, and it recalls it only when actually relevant — never as unprompted noise tacked onto an unrelated answer.

<span style="color: red;">⚠️ **AI Usage Disclosure:** This repository was created and documented with the assistance of Claude and Gemini.</span>

---

## WHY COGNIVAULT?

Cognee (the memory engine underneath) turns raw conversations and documents into a structured, searchable knowledge store — that's the "cogni-" part. Everything lives on disk, under your control, with no cloud memory store — that's the "vault" part. Cognivault: a vault for what the assistant knows about you.

---

## CORE CAPABILITIES

- You upload a PDF, a photo of handwritten notes, or a `.docx`, in any chat.
- The backend extracts the text (OCR for images and scanned PDF pages, direct text extraction for text-based PDFs and Word docs), summarizes it, and answers whatever you asked about it.
- That document — and the summary of what you asked about it — gets written into **long-term memory**, tied to *you*, not to that one conversation.
- Start a brand-new chat and ask about something you uploaded or mentioned in an entirely different chat — it recalls the relevant pieces and answers, even though the current chat has no prior messages about it.
- Every chat is saved, renameable, deletable, and restorable exactly where you left off, with attachments still clickable.

---

## TECH STACK

CogniVault is built entirely on local-first, lightweight, and robust technologies:

- **Backend Framework:** [FastAPI](https://fastapi.tiangolo.com/) — ASGI Python framework for fast, robust API development.
- **ASGI Web Server:** [Uvicorn](https://www.uvicorn.org/) — Lightning-fast ASGI server with auto-reload capabilities.
- **Memory Engine:** [Cognee](https://github.com/cognee-io/cognee) — The cognitive memory engine coordinating the extraction, graphing, and vector indexing of knowledge.
- **Storage & Databases:**
  - **SQLite:** Stores verbatim chat transcripts, conversation logs, and file metadata (with WAL mode enabled for concurrent read/write stability).
  - **LanceDB:** Embedded, serverless vector database used by Cognee to store and query semantic text embeddings locally.
  - **Kuzu Graph:** Embedded graph database used by Cognee to build and traverse entity-relationship graphs.
- **LLM Orchestration:** [LiteLLM](https://github.com/BerriAI/litellm) — Provider-agnostic gateway for completion calls, allowing seamless switching between OpenAI, Anthropic, Groq, local models, etc.
- **Document & Media Parser:**
  - **pdfplumber:** Extracts text and structured tables from PDF documents.
  - **python-docx:** Extracts paragraphs and tables from Word documents (`.docx`).
  - **Pytesseract (OCR):** Wrapper around system Tesseract binary to extract text from images and photos.
  - **Pillow (PIL):** Handles image loading and processing for OCR.
- **Frontend:** Single-file HTML5 interface ([index.html](file:///c:/Users/R1NC/Projects/CogniVault/index.html)) with vanilla CSS styling and native JavaScript (fetch API, event handlers, and responsive layout) requiring zero build steps.

---
## SYSTEM ARCHITECTURE

<img width="2720" height="1856" alt="cognivault_architecture" src="https://github.com/user-attachments/assets/14dea8f1-0cbd-4050-a8a1-2a628fdc57f1" />

---

## DUAL-DATABASE STORAGE DESIGN

Cognivault deliberately keeps two separate storage systems, doing two different jobs. This split is the core design decision the whole project is built around.

| | SQLite | Cognee |
|---|---|---|
| **Stores** | Verbatim chat transcripts, chat metadata (names, timestamps), file references | Semantic, searchable *meaning* extracted from conversations and documents |
| **Scoped by** | `chat_id` — one row of history per conversation | `user_id` — one memory space per person, shared across *all* their chats |
| **Query style** | "give me messages 1 through 40 of chat X, in order" | "what does this person know that's relevant to *this* question" |

Here's why both are necessary, with a full scenario rather than an abstract description:

**Setup:** Say you use Cognivault for a mix of personal notes and work documents.
- On Monday, in a chat about scheduling, you mention: *"I'd rather do async standups than live meetings."*
- On Wednesday, in a totally unrelated chat about lunch recommendations, you happen to mention: *"My usual coffee order is a flat white."*
- On Thursday, in a third chat, you upload `Q3_Budget_Report.pdf` and ask a couple of questions about it.

**A week later**, in a brand-new chat with zero prior messages, you ask: *"What did I say about how I like to run meetings?"*

- If Cognivault only had **SQLite**, this question would fail outright — SQLite only knows about messages inside *this specific chat's* history, and this chat has never mentioned meetings. There's nothing to query.
- If Cognivault only had **Cognee** and no transcript log, it could probably answer the meetings question — but it couldn't tell you *when* you said it, couldn't show you the exact chat it came from, and couldn't let you scroll back and see the conversation in its original order the way you actually had it.

Together: SQLite reconstructs exactly what was said and when, inside any one conversation. Cognee answers "what do you know about me/this" regardless of which conversation something was originally said in. One is the transcript; the other is the understanding.

---

## UNDER THE HOOD: HYBRID SEARCH MECHANICS

Cognee isn't a single database — it's a memory engine built on top of two complementary retrieval systems working together on every query: a **vector store** and a **graph store**. Understanding what each one contributes is the key to understanding why recall in Cognivault behaves the way it does.

### THE VECTOR DATABASE: SEMANTIC SIMILARITY SEARCH

Cognee's default vector store is **LanceDB**. When something gets remembered — a chat exchange, a chunk of an uploaded document — Cognee turns it into an embedding (a numeric fingerprint of its meaning) and stores that alongside the text. When you ask a question, your question is embedded the same way, and the vector store returns whichever stored memories are numerically closest to it — i.e., closest in *meaning*, not in exact wording.

This is what makes recall resilient to phrasing. Ask "what's my preferred way to run meetings" or "how do I like meetings structured" — different words, same underlying question — and the vector store finds the same memory either way, because both questions land in roughly the same region of meaning-space.

It's also *exactly* how Cognivault avoids the classic failure mode of naive keyword-matching memory: pulling in your coffee order when you asked about meetings, just because both memories happen to be phrased as "I prefer X." Embeddings don't confuse the two — "async standups" and "flat white" aren't semantically close, even though the sentences describing them share common words like "my" and "usual."

### THE GRAPH DATABASE: RELATION-BASED ENTITY CONNECTIVITY

Cognee's default graph store is **Kuzu**. As part of ingesting text, Cognee also extracts entities (people, organizations, documents, concepts) and the relationships between them, and stores that as a graph of nodes and edges. This captures structure that pure semantic similarity can miss entirely.

Here's where that matters, continuing the scenario above:

**Setup:** Weeks after the budget report, in a chat about a supplier negotiation, you upload `vendor_contract.docx` — a contract naming *Alderbrook Logistics*. Later still, in yet another chat, you upload a scanned `delivery_invoice.pdf`, which OCR extracts as text mentioning *Alderbrook Logistics* and a *late delivery* penalty.

Now, in a new chat, you ask: **"Have we had any delivery problems with our vendors?"**

- **Vector search** does the first pass: it finds the invoice chunk easily, because "delivery problems" is semantically close to "late delivery penalty." Good — that's the vector store doing what it's good at.
- But the contract document never mentions "delivery problems" anywhere in its text. On pure semantic similarity, it might rank too low to surface at all.
- This is where the **graph** contributes something vectors structurally can't: both documents were ingested with *Alderbrook Logistics* extracted as the same entity node. The graph has an explicit edge connecting the invoice and the contract through that shared vendor. So graph traversal can pull the contract in too — not because its wording resembles the question, but because it's *connected* to the same entity the answer is actually about.

The result is a fuller, correctly-grounded answer: not just "yes, here's a late invoice," but "yes — and here's the contract with the vendor responsible," something a vector-only system would likely have missed and a graph-only system would never have found in the first place, since the graph alone has no way to start from a fuzzy natural-language question like "delivery problems."

### THE ETL PIPELINE: EXTRACT, COGNIFY, LOAD

Under the hood, Cognee processes everything it's given through a three-stage pipeline:

1. **Extract** — pull raw text out of whatever's being ingested (a chat message, a document chunk).
2. **Cognify** — this is where the two storage layers actually get built: text is embedded for the vector store, *and* entities and relationships are identified and written into the graph.
3. **Load** — both representations are persisted to disk, ready to be queried together.

When you ask a question, Cognee's retrieval draws on both layers in the same pass — vector similarity casts a wide net for anything that reads as relevant, and the graph supplies the explicit connections between entities that similarity alone can't see. That combination is what "hybrid retrieval" means in practice here, and it's the reason Cognivault can answer questions that span multiple documents uploaded in unrelated chats, weeks apart, without you ever having to tell it those documents were related.

### EXTERNAL MEMORY INTEROPERABILITY

Because both layers are addressed through a plain `cognee.recall(query, session_id=user_id)` / `cognee.remember(...)` interface, and everything runs on embedded, file-based stores (`DB_PROVIDER=sqlite` for Cognee's own internal bookkeeping, `VECTOR_DB_PROVIDER=lancedb`, `GRAPH_DATABASE_PROVIDER=kuzu`, with local embeddings via `fastembed` — nothing computed over an external API), the same memory space isn't locked to this one chatbot. A second, entirely different tool — a research agent, a CLI note-taker, a scheduled summarizer — can attach to the same `COGNEE_DIR` and read or write the same knowledge graph and vector index. Cognivault's chatbot is one client of that memory, not the memory's owner.

---

## KEY ADVANTAGES & VALUE PROPOSITION

The honest pitch isn't "a chatbot that remembers things" — plenty of tools claim that by stuffing your last few messages back into the prompt. The actual value here is narrower and more concrete:

- **It survives context resets.** Closing the tab, starting a new chat, coming back next month — none of it resets what the assistant knows about you, because that knowledge was never tied to a single conversation window in the first place.
- **Documents become part of what it knows, not just something it read once.** A file you uploaded isn't a one-time context injection that vanishes when the chat ends — it's retrievable the same way a fact you mentioned in passing is, and it can be connected to *other* documents through the graph even when the wording doesn't overlap at all.
- **It's yours.** SQLite file, Cognee's graph and vector stores, uploaded files — all on disk, all inspectable, nothing routed through a third-party memory API. Copy the whole thing to another machine and it keeps working.
- **It's a memory layer, not just a chatbot.** Because the long-term store is addressed through a plain, generic interface, nothing stops a second, entirely different tool from reading the same memory space. The chatbot here is one client of that memory, not the memory's owner.

That combination — persistent, local, document-aware, and structurally reusable by other tools — is the "nice to have." Most memory-enabled chatbots give you one of those. This one is architected to give you all four at once, mainly by refusing to let the conversation window and the knowledge store be the same thing.

---

## GETTING STARTED

For a complete local installation and setup guide, check out [setup.md](file:///c:/Users/R1NC/Projects/CogniVault/setup.md).

### Quick Commands:

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```
2. **Configure environment:**
   Copy `env.example` to `.env` and fill in your API keys and data directories.
3. **Run the backend:**
   ```bash
   python main.py
   ```
4. **Open the app:**
   Open `index.html` directly in your browser.
