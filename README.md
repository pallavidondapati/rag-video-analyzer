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

### Why BAAI/bge-small-en-v1.5?

Tried a few options. Wanted something free, lightweight, runs on CPU, and actually decent at retrieval. BGE Small hit that balance. Larger models improved retrieval a bit but the latency and memory cost wasn't worth it here.

### Why chunk_size=500 with overlap=50?

Trial and error. Not a magic number.

- **256** — retrieval was precise but context kept getting cut off mid-sentence. Answers felt incomplete.
- **1000** — chunks started mixing unrelated ideas together. Retrieval got messy.
- **500** — felt right. The 50-character overlap helped stop information falling between chunk boundaries.

For longer videos I'd probably test hierarchical chunking but for 1-5 minute content this worked fine.

### Why hybrid retrieval (MMR + BM25)?

Started with semantic search only. Worked most of the time but kept missing creator names, exact numbers, specific hashtags. Dense retrieval just isn't built for exact keyword matches.

Adding BM25 fixed that. Both together was noticeably better than either alone.

MMR helped too — without it the retriever kept coming back with four chunks saying basically the same thing. Useless for a comparison task.

### Why Groq + Llama 3.3 70B?

Speed, and it's free. Groq is noticeably faster than most free inference options which matters when users are sitting there waiting.

GPT-4o would probably reason more consistently. If this were a real product I'd benchmark both and decide based on quality vs cost. For this project, free and fast won.

### Why faster-whisper?

Instagram has no transcript API so audio has to be transcribed locally. Compared both whisper and faster-whisper — faster-whisper was significantly quicker on CPU with similar output. Easy decision.

---

## Problems I Actually Ran Into

### Instagram blocking requests

This was more annoying than I expected. yt-dlp can pull reel data but Instagram blocks a lot of unauthenticated requests. Had to export browser cookies and pass them into yt-dlp before things started working consistently. Without cookies a lot of reels just silently failed with no useful error message.

### Whisper latency

A 60-second reel takes around 40-50 seconds to transcribe on CPU. Fine for a demo. For real users this would need to move into background workers immediately — nobody is going to wait a minute for processing.

### Railway blocking YouTube transcripts
youtube_transcript_api._errors.IpBlocked
Spent a while thinking I'd broken something in the code. Eventually figured out YouTube blocks cloud provider IP ranges. The exact same code that ran fine on my laptop was hitting a wall from Railway's servers. Not a bug. Just a hosting problem I couldn't code my way out of.

If I keep working on this, I'd drop youtube-transcript-api entirely and use Whisper for everything.

### yt-dlp in hosted environments

Even separate from the YouTube issue, yt-dlp behaved differently once deployed. Cookie handling, rate limits, authentication — all worked differently than local development. Deployment exposed a lot of edge cases I hadn't hit locally.

### CORS

Frontend was making requests to FastAPI and getting blocked by the browser before the response got through. The backend was actually fine — but the CORS error was hiding that. Took longer to debug than it should have because the real error wasn't visible until I tested outside the browser.

---

## What Breaks at 10,000 Users

**Synchronous transcription**  
Everything runs in sequence right now. Multiple users submitting at the same time will all queue behind each other waiting for Whisper. Needs background workers, a job queue, and a way to return a job ID immediately while processing happens separately.

**FAISS**  
Rebuilding an in-memory index per request doesn't scale. Would move to Qdrant or pgvector with persistent storage.

**Instagram cookies**  
Browser cookies expire. Production needs proper cookie rotation or a third-party extraction service. Current approach is demo-only.

**YouTube transcript reliability**  
Cloud IPs getting blocked isn't fixable in application code. Production needs proxy infrastructure or Whisper as the primary path with youtube-transcript-api only as a fast fallback when available.

**Compute**  
Concurrent Whisper jobs will hit CPU and memory limits fast. The architecture needs to split into separate services — API layer, retrieval, transcription workers — and scale each independently.

---

## What This Project Is Really About

Less about "just RAG" and more about the messy stuff that comes before it — data extraction is harder than LLM reasoning, cloud environments break local assumptions, and retrieval quality depends more on chunking strategy than on which embedding model you pick.

---

## Honest Limitations

Works well for:
- Demo-scale usage
- Short videos, single user
- Quick comparisons

Not ready for:
- High concurrency
- Stable Instagram scraping at scale
- Cloud-based YouTube transcript extraction without a proxy

---

## Stack

**Backend** — FastAPI, LangChain, FAISS, BM25, yt-dlp, faster-whisper  
**Models** — BAAI/bge-small-en-v1.5, Llama 3.3 70B via Groq  
**Frontend** — Next.js 14, React

---

## Environment Variables

```env
GROQ_API_KEY=your_groq_api_key_here
```

Optional:
```env
HF_TOKEN=your_huggingface_token_here
```

---

## Running Locally

```bash
# backend
cd backend
pip install -r requirements.txt
python main.py

# frontend
cd files
npm install
npm run dev
```

You'll need a `cookies.txt` in the backend folder for Instagram reels. Export it from Chrome using "Get cookies.txt LOCALLY" extension while logged into Instagram. Without it most reels will fail silently.

This was the biggest deployment headache.

Everything worked locally. After deploying to Railway, YouTube transcript extraction started failing. The error was:
