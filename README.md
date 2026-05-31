# RAG Video Analyzer

Built this for a technical screening.

The goal was simple: compare a YouTube video and an Instagram Reel using RAG instead of treating them as separate pieces of content.

A user pastes two URLs and can ask questions like:

- What are the main differences?
- Where do the creators agree?
- Which creator is more positive about topic X?

The system retrieves relevant chunks from both videos and uses LLaMA 3.3 70B (via Groq) to generate the comparison.

## Stack

Backend:
- FastAPI
- LangChain
- FAISS
- BM25
- Groq

Frontend:
- Next.js

Models:
- BAAI/bge-small-en-v1.5
- LLaMA 3.3 70B
- faster-whisper

## Things that were annoying

### Instagram

Instagram doesn't make scraping easy.

yt-dlp worked, but for many reels I had to pass browser cookies because unauthenticated requests were getting blocked.

### YouTube deployment

This one surprised me.

Everything worked locally.

After deploying to Railway, transcript extraction started failing with:

youtube_transcript_api._errors.IpBlocked

Turns out YouTube blocks a lot of cloud provider IPs.

The code wasn't broken. The deployment environment was.

If I wanted this to work reliably in production I'd probably skip youtube-transcript-api entirely and just download audio and run Whisper.

## Why FAISS?

Honestly because it's two videos.

The average comparison generates maybe a few dozen chunks.

Running Pinecone for that felt unnecessary.

If this ever had real traffic I'd move to Qdrant or pgvector.

## What breaks first?

Probably transcription.

Whisper is running synchronously right now.

At higher traffic I'd move transcription into a queue and let workers process videos in the background.

The second problem would be Instagram cookies expiring.

The third would be rebuilding FAISS indexes for every request.

## If I had another week

- Add async transcription jobs
- Persist vector indexes
- Support multiple videos instead of just two
- Add source citations to responses
