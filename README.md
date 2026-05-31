# RAG Video Analyzer

Built this for a technical screening. The task was to compare a YouTube video and an Instagram Reel using RAG — paste two URLs, extract content from both, store it in a vector database, and let users ask questions that compare the two creators.

Things like:
- What do both creators agree on?
- Which one is more positive about this topic?
- Why might one video have gotten more engagement?

The AI part was honestly the easiest part. Most of my time went into transcript extraction, cloud IP blocks, CORS issues, and cookie handling. More on that below.

---

## Technology Decisions

### Why FAISS and not Pinecone or Weaviate?

This project compares two videos at a time. After chunking, that's usually a few dozen chunks in memory — nowhere near enough to justify spinning up a managed vector database.

FAISS is free, runs locally, and requires almost no setup. Adding Pinecone would mean another service to manage, network latency, and usage limits, for no real benefit at this scale.

The tradeoff is that the index gets rebuilt every time new videos are processed. That's fine for a demo. For production I'd move to Qdrant or pgvector.

### Why BAAI/bge-small-en-v1.5?

I tested a few options. I wanted something free, lightweight, CPU-friendly, and decent at retrieval. BGE Small hit that balance well. Larger embedding models improved retrieval slightly but cost more in latency and memory. Not worth it for a screening project.

### Why chunk_size=500 with overlap=50?

Trial and error. Not a magic number.

- **256 characters** — retrieval was precise but context kept getting lost. Sentences split in weird places, answers felt fragmented.
- **1000 characters** — chunks started containing multiple unrelated ideas. Retrieval got noisy.
- **500 characters** — felt like the right balance. The 50-character overlap helped prevent information falling between chunks.

For longer videos (10+ minutes) I'd probably revisit this and test hierarchical chunking.

### Why hybrid retrieval (MMR + BM25)?

Started with semantic search only. It worked well most of the time, but occasionally missed creator names, exact numbers, specific hashtags, and quoted phrases. Dense retrieval just isn't built for exact matches.

Adding BM25 fixed that. Using both together was noticeably better than either one alone.

MMR helped too — without it, retrieval kept returning four chunks that were basically identical. That's useless.

### Why Groq + Llama 3.3 70B?

Speed, and it's free. Groq is noticeably faster than most free inference options, which matters for a chat interface where users are waiting on responses.

GPT-4o would probably give more consistent reasoning. If I were building this commercially I'd benchmark GPT-4o, Claude Sonnet, and Llama side by side and make a decision based on quality versus cost. For this project, free and fast won.

### Why faster-whisper?

Instagram doesn't provide transcripts the way YouTube does, so audio has to be transcribed locally. I compared OpenAI's Whisper and faster-whisper. faster-whisper was significantly faster on CPU with similar output quality. Easy choice.

---

## Problems I Actually Ran Into

### Instagram blocking requests

Getting Instagram content to work reliably was the most frustrating part of the project. yt-dlp can pull reel data, but Instagram blocks a lot of unauthenticated requests. I had to export browser cookies and pass them into yt-dlp before things became consistent. Without cookies, a lot of reels just silently failed.

### Whisper latency

A 60-second reel takes roughly 40–50 seconds to transcribe on CPU. For a demo, that's tolerable. For anything with real users, it would need to move into background workers immediately.

### Railway blocking YouTube transcripts

This was the biggest deployment issue.

Everything worked locally. Transcript extraction, embeddings, retrieval — all fine. After deploying to Railway, YouTube transcript extraction started failing immediately. The error was:
youtube_transcript_api._errors.IpBlocked
I spent a while assuming I'd broken something. Eventually figured out that YouTube blocks cloud provider IP ranges. The exact same code that ran fine on my laptop was hitting a wall from Railway's infrastructure. It wasn't a bug. It was a hosting problem I had no control over.

If I keep working on this, I'd stop depending on youtube-transcript-api entirely and generate transcripts with Whisper instead.

### yt-dlp on hosted environments

Even separate from the YouTube issue, yt-dlp behaved differently in hosted environments. Cookie handling, rate limits, and authentication all worked differently than local development. Deployment revealed a lot of edge cases I hadn't encountered locally.

### CORS

The frontend was making requests to the FastAPI backend and getting blocked by the browser before the response ever made it through. The actual backend response was fine — but the browser's CORS error was hiding that. Took longer to debug than it should have because the real error wasn't visible until I bypassed the browser entirely.

---

## What Breaks at 10,000 Users

### Synchronous transcription

Everything runs in sequence right now. If many users submit videos simultaneously, they all queue up waiting for Whisper to finish. This needs background workers, a job queue (Celery + Redis), and a system that returns a job ID immediately and notifies users when processing is done.

### FAISS

Rebuilding an in-memory index per request doesn't scale. I'd move to Qdrant or pgvector with persistent storage.

### Instagram cookies

Browser cookies expire. A production system would need proper cookie rotation, account management, or a third-party extraction service. The current approach is fine for demos, not for anything at scale.

### YouTube transcript reliability

Cloud IPs getting blocked isn't something I can fix in code. A production version needs proxy infrastructure, fallback audio transcription, or a different approach entirely — probably Whisper as the primary path, with youtube-transcript-api only as a fast path when available.

### Compute

Multiple simultaneous Whisper jobs would hit CPU and memory limits fast. The architecture would need to separate into distinct services — API layer, retrieval service, and transcription workers — and scale each independently.

---

## What This Project Demonstrates

This project is less about "just RAG" and more about real-world system design tradeoffs:

- Data extraction is harder than LLM reasoning
- Cloud environments break assumptions made locally
- "Simple demos" expose production bottlenecks fast
- Retrieval quality depends heavily on chunking and hybrid search, not just embeddings

---

## Honest Limitation Summary

Currently optimized for:
- Demo-scale workloads
- Single-user / low concurrency usage
- Short-lived video comparisons

Not yet production-ready for:
- High concurrency
- Stable Instagram scraping at scale
- Cloud-based YouTube transcript extraction

---

## Tech Stack

**Backend**
- FastAPI
- LangChain
- FAISS
- BM25 Retriever
- yt-dlp
- youtube-transcript-api
- faster-whisper

**Models**
- BAAI/bge-small-en-v1.5
- Llama 3.3 70B via Groq

**Frontend**
- Next.js 14
- React

---

##  Environment Variables

Create a `.env` file in the `backend` directory:

```env
GROQ_API_KEY=your_groq_api_key_here
```

Optional depending on your setup:

```env
HF_TOKEN=your_huggingface_token_here   # if using gated models
REDIS_URL=your_redis_url               # if adding Celery later
```

---

##  Running Locally

**Backend**

```bash
cd backend
pip install -r requirements.txt
python main.py
```

**Frontend**

```bash
cd frontend
npm install
npm run dev
```

You'll also need a valid `cookies.txt` file if you're testing Instagram reels. Without it, a lot of reels will silently fail.
