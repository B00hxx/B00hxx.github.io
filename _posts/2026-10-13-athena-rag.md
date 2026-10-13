---
layout: post
title: "Making my lecture PDFs answer back (with receipts)"
description: "What I learned building AthenaAI, a local RAG study assistant: parsing PDFs properly, chunking without losing page numbers, prompts that cite, streaming, an ingestion queue that can kill its own parser, and testing it all without a GPU."
image: /assets/img/posts/athena-rag/cover.png
tags: [llm, rag, system-design]
---

Second post, and I'm already breaking my "keep it lowkey" record. If you read the one about semantic caching, welcome back. If you didn't, hi, I'm Baha, I write about things I built and the stuff I learned while building them. Nothing here will change the way you look at the world, but it might save you a few days of debugging.

This one is about AthenaAI. It started in 2025 as a team project with three friends: a study assistant where you upload a PDF and chat with it. That first version was built with Django, and it worked well enough for a demo. Then I came back to it this year with a different question in mind: what would it take for this thing to hold up outside a demo? I rebuilt the backend around that question, and this post is everything that came out of it.

The code is on GitHub: [B00hxx/Athena_AI](https://github.com/B00hxx/Athena_AI).

## What it does

You drop in a lecture PDF, a paper or a textbook chapter. Then you can ask it questions, and every answer comes with numbered citations like `[1]` that you can expand to see the exact page, section and passage the answer came from. You can also turn the document into flashcards, mark each one "Got it" or "Need review", and come back to them later.

Everything runs on your own machine. The LLM is Gemma 7B served by Ollama, so your notes never leave your laptop. That constraint shaped a lot of the decisions below, mostly because a 7B model on a laptop GPU is slow, and memory is tight.

## RAG in one picture

RAG stands for retrieval-augmented generation. The idea is simple: an LLM doesn't know what's in your lecture notes, and you can't paste a 40-page PDF into every question. So you find the few passages that are relevant to the question, and you give the model only those, along with the question. The model "reads" the passages and answers from them.

That splits the system into two pipelines that run at very different times:

![Ingest runs once per PDF in the background; answering runs on every question and reads from the same vector index](/assets/img/posts/athena-rag/pipeline.png)

Ingestion runs once per upload, in the background. The PDF gets parsed into text, cut into passages (chunks), and each chunk is turned into a vector by an embedding model. The vectors go into ChromaDB, a vector database, with the page number and section of each chunk attached.

Answering runs every time someone asks something. The question is embedded with the same model, so it lands in the same space as the chunks. ChromaDB returns the 5 chunks closest to it, and those go into the prompt with the question. The model streams back an answer that cites them.

If you read my semantic caching post, the retrieval part will look familiar: embed, then find the nearest vectors with an HNSW index. The ChromaDB collection is created with cosine distance (`hnsw:space: cosine`), and the embeddings are normalized, so cosine similarity is just a dot product. What's different here is everything around it.

My big lesson from this project: the quality of a RAG system is decided long before the LLM sees anything. If the text is extracted badly, or chunked badly, no prompt will save you.

## Step one: actually reading the PDF

PDFs are not documents in the way we think of them. A PDF is a list of instructions for drawing glyphs at coordinates on a page. There's no "paragraph", no "heading", no "table", just characters placed at positions. Headings are only "big bold text", and a two-column layout is two piles of text that a naive extractor happily reads left to right across both columns. Scanned slides are even worse: they're images, so there's no text at all.

I use [Marker](https://github.com/datalab-to/marker) for this. It runs layout detection models over each page, figures out what's a heading, a list, a table or a figure caption, runs OCR where there's no text layer, and outputs Markdown. So `## The Calvin Cycle` comes out as an actual Markdown heading, which matters a lot for the next step.

Marker has a `paginate_output` option that separates pages in the output with a marker line: the page index in braces, followed by 48 dashes. I split on that with a regex to get a list of `(page_number, markdown)` pairs:

```python
_MARKER_PAGE_RE = re.compile(r"^\{(\d+)\}-{48}$", re.MULTILINE)

parts = _MARKER_PAGE_RE.split(markdown)
# split() yields [preamble, page_idx, text, page_idx, text, ...]
pages = [(int(parts[i]) + 1, parts[i + 1]) for i in range(1, len(parts) - 1, 2)]
```

Keeping the page boundary from the very beginning is what makes exact citations possible later.

The cost: Marker's models take a few GB, and parsing is slow on a CPU (about a minute for a 3-page lecture on my laptop, around 10 seconds with the GPU). So there's a second parser, `pdftext`, behind one setting (`PDF_PARSER=pdftext`). It only reads the PDF's existing text layer with pypdfium2: near-instant, no layout, no OCR. It's also what the tests use. Every upload's extracted Markdown is saved next to the PDF, which turned out to be the most useful debugging tool in the whole project. When an answer is wrong, the first question is always "what did the parser actually see?"

## Chunking without losing your place

The embedding model I use, `all-MiniLM-L6-v2`, turns a piece of text into 384 numbers. One vector for a whole 40-page document would be useless, since it would average every topic into mush. So the document gets cut into passages of at most 1,000 characters, and each one gets its own vector.

The easy way is to glue all the pages together and cut every 1,000 characters. That's what a lot of tutorials do, and it breaks citations: a chunk can start on page 2 and end on page 3, or start in "Light reactions" and end in "Calvin cycle". Then what do you cite?

![Each page is split at its headings first, then into overlapping passages; a page that starts mid-section inherits the previous heading](/assets/img/posts/athena-rag/chunking.png)

So the splitter works in two levels. First, each page is cut at its Markdown headings into sections. Then each section is cut into passages with LangChain's `MarkdownTextSplitter`, which prefers to break at Markdown boundaries (headings, paragraphs, lines) before breaking mid-sentence. Passages overlap by 150 characters, so a sentence that sits on a cut appears in both neighbours instead of being half in each.

The detail I like most is the heading carry-over. A lot of pages start in the middle of a section that began on the previous page. The `section` variable lives outside the page loop, so a page with no heading at the top just inherits whatever section was open when the previous page ended:

```python
section = ""
for number, page_text in pages:
    text = page_text.strip()
    blocks, pos = [], 0
    for m in _HEADING_RE.finditer(text):
        if text[pos:m.start()].strip():
            blocks.append((section, text[pos:m.start()]))
        section, pos = m.group(1).strip("*_ "), m.start()
    blocks.append((section, text[pos:]))

    for block_section, body in blocks:
        for piece in splitter.split_text(body):
            piece = piece.strip()
            if len(piece) < 20:  # stray page numbers, lone headings
                continue
            chunks.append(Chunk(text=piece, page=number, section=block_section))
```

Every chunk ends up with an exact page and section, and they're stored as metadata next to its vector in ChromaDB, along with the document ID and the chunk's position in reading order.

### Embedding the heading, not storing it

One more trick. A chunk like "It takes place in the stroma of the chloroplast" is almost impossible to retrieve for the question "where does the Calvin cycle happen?", because the chunk never says "Calvin cycle". The heading above it does.

So the vector is computed from `section heading + passage`, while the text stored and shown to the user is the passage alone:

```python
def _embedding_text(chunk: Chunk) -> str:
    return f"{chunk.section}\n{chunk.text}" if chunk.section else chunk.text
```

The first version of AthenaAI did something heavier to solve the same problem: it asked the LLM to summarize every chunk before embedding it. That costs one LLM call per chunk at upload time, which on a local 7B model means waiting minutes before you can ask anything, and the summary can drop exactly the detail someone will ask about. Prefixing the heading gets most of the context for free.

## The prompt is a contract

Once retrieval returns its 5 chunks, they go into the prompt. This is the whole message list the model receives for one question:

![The model gets a system prompt with rules, the last three turns as plain text, and the current question with numbered excerpts](/assets/img/posts/athena-rag/prompt.png)

The system prompt is short and strict. Answer from the excerpts and cite them inline like `[1]` or `[2][3]`. If the excerpts don't contain the answer, say so plainly; general knowledge is allowed after that, but labelled as not coming from the document. And never invent page numbers, quotes or facts.

Each excerpt is numbered and carries its location, so the model sees `[1] (page 3, section "The Calvin Cycle")` followed by the text. The `[1]` in the answer then maps back to that exact chunk, which is what the "sources" panel in the UI shows. When web search is turned on (DuckDuckGo, 3 results), web results are numbered `[W1]`, `[W2]`, so the student can always tell what came from their document and what didn't.

Follow-up questions ("explain that again", "what about the second one?") need the conversation. But if every past turn carried its own 5 excerpts, a 10-question conversation would blow up the context window. So only the current question carries excerpts. The last 3 turns are sent as plain question/answer pairs. The context window is set to 8,192 tokens (`OLLAMA_NUM_CTX`), and this keeps every request well inside it no matter how long the conversation gets.

Honest caveat: a 7B model doesn't follow these rules perfectly. It sometimes cites `[2]` for something that's in `[4]`. The citations make that easy to catch, though, which is the whole point. An answer you can check is more useful than an answer that sounds confident.

## Streaming: one JSON object per line

On my laptop, a full answer from Gemma 7B takes a while to generate. Staring at a spinner for that long feels broken, so answers stream token by token, the way ChatGPT does it.

There are a few ways to stream over HTTP. I went with NDJSON (newline-delimited JSON): the response is a stream of lines, each one a complete JSON object.

![FastAPI yields one JSON line per event; nginx passes them through unbuffered; the browser splits on newlines](/assets/img/posts/athena-rag/streaming.png)

The first event is `sources`, sent before the LLM even starts, so the UI can show which passages are being used while the answer is still coming. Then one `token` event per piece of text, and finally `done` with the saved message's ID (or `error` if generation failed).

Three details matter here.

First, nginx buffers responses by default. It collects the backend's output and sends it to the browser in big pieces, which kills streaming completely: the answer shows up all at once at the end. The fix is one response header, `X-Accel-Buffering: no`, which tells nginx to pass this response through as it arrives.

Second, a network chunk is not a line. The browser reads the stream in chunks, and a chunk can end in the middle of a JSON object. If you call `JSON.parse` on each chunk, it works most of the time and fails on whichever chunk happens to end mid-object. The client keeps a buffer and only parses up to the last newline:

```ts
buffer += decoder.decode(value, { stream: true })
let newline
while ((newline = buffer.indexOf("\n")) >= 0) {
  const line = buffer.slice(0, newline).trim()
  buffer = buffer.slice(newline + 1)
  if (line) onEvent(JSON.parse(line))
}
```

There's a test for exactly this case: JSON lines arriving split across network chunks.

Third, the stream outlives the request. In FastAPI, the database session you get from a dependency is closed when the endpoint function returns. But with a `StreamingResponse`, the endpoint returns right away and the generator keeps running afterwards. So the generator opens its own session to save the conversation at the end. It saves the question and the answer together, only after the answer finished, so a failed generation never leaves a question without an answer in the history.

The endpoint is also a plain `def`, not `async def`, on purpose. Retrieval and the Ollama client are blocking calls. In a plain `def` endpoint, FastAPI runs them in a thread pool. In an `async def`, they would block the event loop and freeze every other request while one person waits for an answer.

## The part I didn't expect: ingestion

I expected the hard part of this project to be retrieval. It turned out to be ingestion.

Parsing a PDF with Marker can take minutes on a CPU, so the upload endpoint can't wait for it. It checks the file (content type, the `%PDF` signature at the start of the bytes, and size, by reading at most one byte past the limit), saves it under a server-generated UUID, and returns `202 Accepted` with status "processing". The frontend polls the document until it's "ready" or "failed". The actual work happens on a background worker.

Then the questions started. What if two people upload at once? Marker's models take several GB, so running two parses in parallel on a laptop is a good way to run out of memory. What if someone deletes a document while it's being parsed? What if Marker hangs on a weird PDF? What if it crashes? What if the server restarts in the middle?

![Uploads wait in line; one worker thread processes them in order, each in a child process that can be killed on delete, timeout or crash](/assets/img/posts/athena-rag/ingestion-queue.png)

The answer ended up having three layers.

The first layer is a queue with one worker. A `ThreadPoolExecutor` with `max_workers=1` processes documents one at a time, in upload order. Next to it, a plain Python list tracks the line, so the API can tell the UI "2 documents ahead of yours". All access to it goes through a lock, since the API threads and the worker thread touch it at the same time.

The second layer is a child process per parse, and this was the key decision. You can't kill a thread in Python, so if Marker hangs inside a thread, it hangs forever and the whole queue is stuck behind it. A process, you can kill. So each PDF is parsed in its own short-lived child process, which sends the pages back through a pipe. The worker waits by polling the pipe every half second, and on each tick it checks three things: was the document cancelled, did the process die, did we pass the deadline?

```python
deadline = time.monotonic() + timeout_s
while not receiver.poll(0.5):
    if document_id in _cancelled:
        raise IngestionCancelled()
    if not process.is_alive():
        raise died()
    if time.monotonic() > deadline:
        raise TimeoutError(f"Parsing took longer than {timeout_s / 60:.0f} minutes and was stopped.")
```

The `finally` block kills the process if it's still alive, whatever happened. That gives you a lot for free. Deleting a document mid-parse kills its parser, so the next upload starts right away. A parse that runs past `INGEST_TIMEOUT_MINUTES` (60 by default) is stopped and the document is marked failed with a readable error. If Marker crashes or gets killed for using too much memory, the API process is unaffected and the error reports the exit code. And when the process exits, all of Marker's GPU memory goes with it.

Child processes are started with `spawn` instead of the Linux default `fork`, because CUDA can't be used in a forked process.

The third layer is recovery on restart. The document's status lives in PostgreSQL, not only in memory. When the server starts, any document still marked "processing" was interrupted, so it gets put back in the queue. Failed documents can be retried from the UI, which calls `POST /documents/{id}/reprocess`.

### Sharing a 6 GB GPU

My laptop has an RTX 4050 with 6 GB of VRAM. Gemma 7B with an 8K context needs about 9 GB, so Ollama already runs it split between CPU and GPU. Marker also wants the GPU. Running both at once meant one of them got pushed into shared memory, which on Windows and WSL silently turns a one-minute parse into tens of minutes.

So with `FREE_GPU_FOR_INGESTION=true` (the default on the GPU stack), the worker asks Ollama to unload the model right before each parse, by sending a request with `keep_alive=0`. Marker gets the whole card, and the LLM reloads on the next question. It costs a few seconds on that next question. If you have 12 GB or more, you set it to false and everything stays loaded.

## Flashcards that cover the whole document

Flashcards looked like the easy feature. Take some text, ask the LLM for question/answer pairs, done.

The first problem was which text. You can't send 40 pages to the model, so you send a budget of 12,000 characters. The obvious way to fill it is to take chunks from the start until the budget runs out, and then every card is about the introduction.

![Taking the first chunks only covers the introduction; even intervals cover the whole document; a topic uses the best matches in reading order](/assets/img/posts/athena-rag/flashcard-sampling.png)

Without a topic, chunks are sampled at even intervals across the document: compute how many average-sized chunks fit in the budget, then take every n-th one from start to end. With a topic ("Calvin cycle"), it's a retrieval problem again: take the 20 chunks most similar to the topic, keep as many as fit in the budget, then put them back in reading order so the model reads them in a sensible sequence.

The second problem was getting JSON out of a 7B model. Ask nicely for JSON and you get JSON wrapped in a Markdown code fence, or a bare array instead of the object you asked for, or a friendly sentence before the JSON. Ollama supports structured outputs, so the request passes a JSON schema in the `format` field, and the model's generation is constrained to match it. The parser is still tolerant as a second line of defense: it finds the first `{` or `[` and the last `}` or `]`, parses what's between, and accepts both shapes. Cards without a question or an answer are dropped. If nothing usable comes back, the API returns a clear 502 instead of saving an empty set.

## Accounts, ownership, and the boring stuff

Users sign up and log in with JWTs: an access token that lasts 60 minutes and a refresh token that lasts 7 days. Each token has a `type` claim, and the API checks it, so a refresh token can't be used as an access token even though both are signed with the same key. Passwords are hashed with bcrypt. The frontend refreshes the access token automatically once when a request comes back 401, then retries it.

Every document, message and flashcard query goes through one helper, `get_owned_document`, that checks ownership. If the document belongs to someone else, it returns 404, not 403. A 403 would confirm that the ID exists, which lets someone probe for other people's documents. With 404, "doesn't exist" and "not yours" look the same.

All the vectors of all users live in one ChromaDB collection, and every query filters on `document_id`. That's simpler than one collection per document, and there's a test that checks a search in my document never returns passages from yours.

## Running it: five containers

![The browser talks only to nginx, which serves the React app and proxies /api to FastAPI; Postgres, ChromaDB and Ollama sit behind it](/assets/img/posts/athena-rag/architecture.png)

The whole thing runs with Docker Compose: PostgreSQL, Ollama, a one-shot `ollama-pull` job that downloads the model, the FastAPI backend (with ChromaDB embedded as a library, persisted to a volume), and nginx serving the React build.

The browser only ever talks to nginx. It serves the frontend and forwards `/api/*` to the backend, so the app and the API share an origin and there's no CORS to configure. The API, the database and Ollama are bound to `127.0.0.1`, so only nginx is reachable from the network. The backend container runs as a non-root user.

Startup order is enforced with health checks, which I learned is more subtle than `depends_on` makes it look. The backend can't start until Postgres is healthy AND the model download has finished, otherwise the first question fails with "model not found". So `ollama-pull` waits for Ollama to be healthy, the backend waits for `ollama-pull` to complete successfully, and nginx waits for the backend's health check. The backend has two health endpoints: `/health/live` (the process is up) and `/health/ready` (the database answers and the model is pulled).

A second Compose file, `docker-compose.gpu.yml`, gives the GPU to both Ollama and the backend and rebuilds the backend with CUDA PyTorch. `make up` runs it on CPU, `make gpu` on GPU. `make env` generates a `.env` with a random database password and JWT secret, because "please set a strong secret" in a README never works.

## Testing it with pytest

Same puzzle as my last project: the real system needs Marker, an embedding model, ChromaDB, Postgres and a 7B LLM, and the LLM never answers the same way twice. The approach I landed on this time: run everything for real except the two model-backed pieces.

![Unit tests cover pure logic; integration tests run the real app with a fake embedder and a fake LLM; end-to-end tests run against the live stack](/assets/img/posts/athena-rag/testing.png)

The integration tests drive the real FastAPI app over HTTP, with a real database (SQLite), a real ChromaDB index, real `pdftext` parsing and the real ingestion code. The test PDFs are generated by a tiny PDF writer in the test helpers: a 3-page lecture on photosynthesis, one topic per page, so retrieval is checkable.

The fake embedder is my favourite piece again. It's a hashed bag of words: each word is hashed with MD5 into one of 512 dimensions, and the vector counts the words. Two texts that share words end up with similar vectors, which is a terrible embedding model and a perfect test one. It's deterministic, it needs no download, and retrieval results still mean something. So a test can upload the lecture, ask "Which enzyme fixes carbon dioxide?", and assert that the top passage is on page 3.

The fake LLM returns a canned answer and records every prompt it receives. That turns prompt behaviour into something you can assert on: the retrieved excerpts are in the prompt, a follow-up question carries the previous turns, the history is capped at the configured number of turns, and a failed generation streams an error and saves nothing.

The ingestion queue tests are the ones I'm proudest of, because they run real child processes. The child targets are tiny module-level functions (they have to be importable, because `spawn` pickles them by name): one that sleeps for 60 seconds, one that calls `os._exit(3)`, one that reports an error. With those, the tests check that a timeout stops the process, a crash is reported with its exit code, and cancelling kills a running parser and unblocks the uploads behind it.

That's 110 backend tests at about 96% line coverage, plus 61 frontend tests with Vitest. A third layer, `e2e/`, runs against the live stack with real Marker and Gemma: it uploads the lecture, checks that the answer cites page 3 and mentions RuBisCO, and generates flashcards. It's skipped unless `ATHENA_E2E_URL` is set, since it needs the models running. GitHub Actions runs the lint, both test suites and the builds on every push, and validates both Compose files.

## How fast is it?

Measured on my laptop (20-thread CPU, RTX 4050 with 6 GB) with Gemma 7B, on a 3-page lecture PDF:

| Step | CPU only | GPU |
|---|---:|---:|
| Ingest the PDF (Marker + embeddings) | ~55 s | ~10 s |
| First answer after start (model loads) | ~80 s | ~10 s |
| Time to first token, warm | 20–30 s | 1–2 s |
| Embedding + retrieval per question | < 1 s | < 1 s |

The interesting row is the last one: retrieval is never the bottleneck. On GPU, about 7 of the 10 ingestion seconds are just loading Marker's models. On CPU, the LLM dominates everything, and the best advice is a smaller model (`gemma2:2b` or `qwen2.5:3b`, one line in `.env`).

## What's still rough

A few things I'd fix with more time:

- The ingestion queue lives in the memory of one process. That's fine with one Uvicorn worker, but with several, each would have its own queue and its own idea of "position in line". Moving it to Postgres or Redis would fix that.
- `all-MiniLM-L6-v2` was trained mostly on English. A lot of my own courses at Sup'Com are in French, where it's a weaker fit. A multilingual embedding model would be the first upgrade.
- Retrieval is purely dense (vectors only). Exact terms like an enzyme name or a formula can be missed when the question is phrased differently. Hybrid search (vectors plus keyword search like BM25) and a reranker over the top results would help.
- I have tests that retrieval works, but no evaluation of how well it works: no set of real questions with known correct passages to measure recall@5 on. That's the RAG version of the threshold benchmark from my last post, and it's missing.
- Generating a new flashcard set replaces the old one, review progress included.
- There are no database migrations yet. New tables are created on startup, but changing an existing table means a manual `ALTER TABLE`.
- Refresh tokens can't be revoked before they expire, since nothing about them is stored server-side.

## Wrapping up

When I started the rebuild, I thought RAG was "embed, search, prompt". Those three steps are maybe a fifth of the code. The rest is the stuff around them: reading the PDF properly, cutting it without losing where each piece came from, making the model show its sources, getting tokens to the browser as they're generated, and a background queue that survives crashes, timeouts and impatient users who delete things mid-upload. Same lesson as last time, honestly: the AI part was the smallest part.

Thanks for reading this far. If you've built a RAG system and found a better way to chunk, or you think I got something wrong, I'd really like to hear about it.
