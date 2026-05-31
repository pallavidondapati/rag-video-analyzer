# RAG Video Analyzer

I built this for a technical screening.

The goal was to compare a YouTube video and an Instagram Reel using RAG. A user pastes two URLs, the system extracts content from both videos, stores it in a vector store, and allows questions that compare the two creators.

Some example questions:

* What are the main differences between these creators?
* Where do they agree?
* Which creator is more positive about a topic?
* Why might one video have received more engagement?

The backend is built with FastAPI and LangChain. For retrieval I used FAISS, BM25, and BGE embeddings. Responses are generated using LLaMA 3.3 70B through Groq.

## Why FAISS?

Honestly, because this project only compares two videos at a time.

After chunking, there are usually only a few dozen chunks in memory. Running Pinecone or Weaviate for that felt unnecessary.

FAISS is free, local, and takes almost no effort to set up.

The downside is that the index gets rebuilt whenever new videos are processed. That's fine for a demo project but probably not something I'd keep if this had real traffic.

If I were building this for production I'd probably move to Qdrant or pgvector.

## Why chunk size 500 with overlap 50?

This came from trial and error.

I first tried smaller chunks and retrieval became too fragmented. A lot of context was lost because sentences and ideas were being split apart.

Then I tried larger chunks. Context improved, but retrieval quality got worse because chunks started containing multiple unrelated ideas.

500 with a small overlap ended up being the best balance for the videos I tested.

It's not a magic number. It's just what worked best after a few experiments.

## Why hybrid retrieval?

I started with vector search only.

Most of the time it worked well, but it occasionally missed things like creator names, exact phrases, hashtags, and numbers.

BM25 was better at those exact matches.

Using both together gave better results than either one by itself.

MMR also helped avoid getting four nearly identical chunks back from retrieval.

## What gave me the most trouble?

Instagram and YouTube.

Instagram extraction worked with yt-dlp, but some requests failed until I started passing browser cookies. Once cookies were added things became much more reliable.

YouTube was even more frustrating.

Locally everything worked. Transcript extraction, embeddings, retrieval, all fine.

After deploying to Railway, transcript extraction immediately started failing.

The logs showed:

youtube_transcript_api._errors.IpBlocked

At first I assumed I'd broken something.

After a lot of debugging I found out YouTube blocks many cloud provider IPs. The exact same code worked on my laptop but failed from Railway.

That ended up being the biggest deployment issue in the project.

If I continue working on this, I'd probably stop depending on youtube-transcript-api and generate transcripts directly with Whisper instead.

## What breaks at 10,000 users?

The first thing that breaks is transcription.

Right now everything runs synchronously. If lots of users start uploading videos, requests will spend most of their time waiting for transcription jobs to finish.

The second issue is FAISS. Rebuilding indexes is fine for demos but not for large-scale usage.

The third issue is Instagram extraction. Cookie-based approaches work but they're not something I'd trust long-term.

I'd eventually move to:

* Qdrant or pgvector for storage
* Background workers for transcription
* Separate services for extraction and retrieval

## Tech Stack

Backend

* FastAPI
* LangChain
* FAISS
* BM25 Retriever
* yt-dlp
* youtube-transcript-api

Models

* BAAI/bge-small-en-v1.5
* LLaMA 3.3 70B (Groq)
* faster-whisper

Frontend

* Next.js 14
* React

## Running Locally

Backend

```bash
cd backend
pip install -r requirements.txt
python main.py
```

Frontend

```bash
cd files
npm install
npm run dev
```

## Final Thoughts

The AI part was actually the easiest part of this project.

Most of the time went into dealing with transcript extraction, deployment issues, CORS problems, cloud restrictions, and getting retrieval quality where I wanted it.

The project works, but it also taught me that building around AI models is usually harder than calling the model itself.
