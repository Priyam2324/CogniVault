# COGNIVAULT

**A local-first AI assistant that reads your PDFs, images, and Word docs, and remembers them — across every conversation you have with it, forever.**

Upload a document in one chat, ask about it in a completely different chat weeks later, and it answers correctly. Mention an offhand personal preference in passing, and it recalls it only when actually relevant — never as unprompted noise tacked onto an unrelated answer.

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

## ENGINEERING CHALLENGES & RESOLUTIONS

### CHALLENGE 1: PREVENTING UNRELATED FACTS FROM BLEEDING INTO RESPONSES

**Setup:** Recall the Monday/Wednesday example above — a meeting-format preference in one chat, a coffee order in a completely unrelated chat.

**Symptom:** An early version of the relevance filter worked by counting words shared between the current question and each stored memory. Asking "how do I like to run meetings?" would sometimes pull in the coffee-order fact too, unprompted — because both memories happened to be phrased as "my [thing] is [preference]," which is more than enough word overlap to pass a naive filter, even though the two facts have nothing to do with each other.

**Fix:** Stopped hand-rolling relevance with word counting. Cognee's vector similarity search already ranks results by actual semantic closeness, which doesn't confuse two memories just because they share sentence structure. The score Cognee returns is what gets thresholded, and the model is explicitly told to use only what's genuinely relevant to the current question and ignore the rest of what comes back.

### CHALLENGE 2: ENHANCING RESPONSE DEPTH FOR BROAD QUERIES

**Symptom:** Specific questions worked reliably; broad, low-specificity ones sometimes returned too little to give a real answer.

**Root cause:** Retrieval was capped low and sliced without a strong ordering guarantee — fine for a narrow question with one obvious answer, not enough breadth for a question that's supposed to summarize everything.

**Fix:** Raised the retrieval count and let the model synthesize a coherent summary from whatever comes back, instead of maintaining a separate "summary mode" code path with its own memory-selection rules.

### CHALLENGE 3: RESOLVING MID-CONVERSATION RE-GREETINGS

**Symptom:** A cheerful reintroduction showing up out of nowhere, several messages into an already-ongoing chat.

**Root cause:** Each request to the model was effectively stateless — a system prompt plus only the single current message, no actual conversation history attached. The model had no way to know it had already said hello, because as far as that specific API call was concerned, this was the first thing anyone had said to it.

**Fix:** Chat history is now pulled from SQLite and passed as real prior turns on every request, alongside a flag the prompt uses to decide whether a greeting is even appropriate. Continuity turned out to require actual conversation state, not a stronger instruction to not repeat itself.

### CHALLENGE 4: CONTEXTUALIZING AMBIGUOUS FOLLOW-UP QUESTIONS

**Setup:** Continuing the Q3 budget report example — after asking "what was the biggest line item in the Q3 report," a natural follow-up might just be *"and how does that compare to last quarter's?"*

**Symptom:** Taken on its own, that follow-up shares almost no vocabulary with the document chunk that actually holds the answer — no mention of the report, the vendor, or even "budget."

**Fix:** The query used for recall is no longer just the current message in isolation — it's the last few turns of conversation concatenated with the current question. That gives the embedding search real nouns and names to match against (carried over from earlier in the exchange), instead of a bare, ambiguous follow-up with nothing concrete to anchor to.

### CHALLENGE 5: REDUCING LATENCY BY COLLAPSING MULTIPLE LLM CALLS

**Symptom:** An earlier version ran a separate classification call before every response, just to decide *how* to answer — a full extra network round-trip on every single message.

**Fix:** Collapsed to one call. The model is handed the ranked memory and instructed, in the same prompt that generates the answer, to use what's relevant and ignore what isn't. A dedicated classification pass wasn't adding accuracy — it was adding latency and cost for a decision the model could already make correctly while answering.

### CHALLENGE 6: DETECTING AND HANDLING SILENT MEMORY WRITE FAILURES

**Symptom:** No errors, no crashes — memory just occasionally, invisibly, failed to stick.

**Root cause:** Logging for the memory layer was set to suppress everything but the most critical failures, so a genuine write failure produced no signal anywhere.

**Fix:** Logging level raised so failures are actually visible, and every memory read/write call is wrapped so an exception degrades gracefully to "answer without that memory" instead of silently losing data or crashing the request.

### CHALLENGE 7: SECURING COMMITTED API CREDENTIALS

**Symptom:** An API key hardcoded as a literal fallback value in the code, rather than pulled from the environment.

**Fix:** The key is now read from the environment with no default — the app refuses to start if it isn't set, rather than silently running on a checked-in credential.

> **Action needed:** if you've ever committed a real key like this anywhere, rotate it with the provider before sharing the repo or the environment file with anyone else.

---

## KEY ADVANTAGES & VALUE PROPOSITION

The honest pitch isn't "a chatbot that remembers things" — plenty of tools claim that by stuffing your last few messages back into the prompt. The actual value here is narrower and more concrete:

- **It survives context resets.** Closing the tab, starting a new chat, coming back next month — none of it resets what the assistant knows about you, because that knowledge was never tied to a single conversation window in the first place.
- **Documents become part of what it knows, not just something it read once.** A file you uploaded isn't a one-time context injection that vanishes when the chat ends — it's retrievable the same way a fact you mentioned in passing is, and it can be connected to *other* documents through the graph even when the wording doesn't overlap at all.
- **It's yours.** SQLite file, Cognee's graph and vector stores, uploaded files — all on disk, all inspectable, nothing routed through a third-party memory API. Copy the whole thing to another machine and it keeps working.
- **It's a memory layer, not just a chatbot.** Because the long-term store is addressed through a plain, generic interface, nothing stops a second, entirely different tool from reading the same memory space. The chatbot here is one client of that memory, not the memory's owner.

That combination — persistent, local, document-aware, and structurally reusable by other tools — is the "nice to have." Most memory-enabled chatbots give you one of those. This one is architected to give you all four at once, mainly by refusing to let the conversation window and the knowledge store be the same thing.
