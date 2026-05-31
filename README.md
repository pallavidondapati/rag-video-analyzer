# RAG Video Analyzer

Built this project as part of a technical screening.

The idea is simple: take a YouTube video and an Instagram Reel, extract their content, store it in a vector database, and allow users to ask questions across both pieces of content at once.

Instead of summarizing a single video, the system focuses on comparing viewpoints, identifying similarities and differences, and answering questions using information retrieved from both sources.

---

# Demo

Users provide:

- A YouTube URL
- An Instagram Reel URL

The application:

1. Extracts content from both sources
2. Chunks and embeds the text
3. Stores embeddings in FAISS
4. Uses hybrid retrieval (MMR + BM25)
5. Sends relevant context to LLaMA 3.3 70B via Groq
6. Returns a comparison-focused response

---

# Architecture

```text
YouTube Video
       |
       v
Transcript Extraction
       |
       v
Chunking
       |
       v
Embeddings (BGE Small)
       |
       v
FAISS Vector Store
       |
       +----------------+
                        |
Instagram Reel          |
       |                |
       v                |
Transcript / Metadata   |
       |                |
       +----------------+
                        |
                        v
               Hybrid Retrieval
               (MMR + BM25)
                        |
                        v
                 Groq LLaMA 3.3
                        |
                        v
                Comparison Response
```

---

# Why These Choices

## FAISS over Pinecone / Weaviate

This application compares only two videos at a time.

Using a managed vector database would add complexity and cost without providing much value at this scale.

FAISS is:

- Fast
- Local
- Free
- Easy to prototype with

For a production system serving thousands of creators, I would likely move to:

- Qdrant
- pgvector
- Weaviate

to support persistence and horizontal scaling.

---

## Hybrid Retrieval (MMR + BM25)

During testing I found that semantic retrieval alone wasn't enough.

Examples:

- Creator names were occasionally missed
- Hashtags weren't always retrieved
- Exact numbers sometimes disappeared

BM25 handled keyword matching well, but struggled with semantic meaning.

Combining both approaches produced significantly better results.

MMR was added to reduce duplicate chunks and increase retrieval diversity.

---

## Chunk Size = 500, Overlap = 50

I experimented with several chunking strategies.

### 256

Pros:

- High precision

Cons:

- Frequently lost context
- Split ideas mid-sentence

### 1000

Pros:

- More context

Cons:

- Retrieval became noisy
- Less relevant chunks were returned

### 500 with 50 overlap

This produced the most consistent retrieval quality for videos in the 1–5 minute range.

---

## BAAI/bge-small-en-v1.5

Requirements:

- Free
- CPU friendly
- Good retrieval performance

BGE Small offered a strong balance between speed and quality without requiring GPUs.

---

## Groq + LLaMA 3.3 70B

I chose Groq primarily because of inference speed.

Response latency was dramatically lower than most alternatives I tested.

The free tier was also generous enough for experimentation.

For a production product I would benchmark:

- GPT-4o
- Claude
- LLaMA

using real creator questions before making a final decision.

---

## faster-whisper

I initially looked at OpenAI Whisper.

Eventually switched to faster-whisper because:

- Faster CPU inference
- Similar transcription quality
- Better suited for free deployment environments

For short reels, the difference was significant.

---

# Biggest Challenges

## Instagram Retrieval

Getting Instagram content turned out to be more difficult than expected.

Instagram blocks many unauthenticated requests.

While yt-dlp works extremely well, several requests failed until browser cookies were passed into the extraction process.

Current solution:

- Export browser cookies
- Pass cookie file to yt-dlp

This works well for development but would need improvement for production use.

---

## YouTube Transcript Retrieval During Deployment

This was the biggest issue I encountered.

Everything worked correctly on my local machine.

After deploying to Railway, transcript extraction started failing with:

```text
youtube_transcript_api._errors.IpBlocked
```

The problem wasn't my code.

YouTube blocks many cloud-hosted IP addresses from:

- Railway
- Render
- AWS
- Google Cloud

As a result:

- Local development worked
- Cloud deployment failed

This was an important lesson because it exposed a real-world deployment issue that doesn't appear during local testing.

Potential solutions:

- Residential proxies
- Authenticated requests
- Dedicated transcript providers
- Downloading audio and generating transcripts with Whisper instead

---

## yt-dlp Challenges

yt-dlp worked reliably during development but introduces a few production concerns:

- Instagram rate limiting
- Cookie expiration
- Platform changes breaking extraction

Any production version would need a more robust content acquisition strategy.

---

# What Breaks at 10,000 Users?

## FAISS

Current implementation rebuilds the vector index per request.

At scale this would become inefficient.

I'd replace it with:

- Qdrant
- pgvector
- Weaviate

---

## Whisper

Transcription currently happens synchronously.

This means requests wait until transcription finishes.

At larger scale I would introduce:

- Celery
- Redis
- Background workers

so users receive a job ID while processing occurs asynchronously.

---

## Instagram Cookies

Current extraction depends on cookies.

Cookies eventually expire.

A production system would require:

- Automatic refresh mechanisms
- Alternative extraction pipelines
- Official API integrations if available

---

## Single Backend Instance

Currently one service handles:

- Retrieval
- Embeddings
- Transcription
- LLM calls

At scale I would separate:

- API layer
- Retrieval service
- Transcription workers

to improve reliability and throughput.

---

# Tech Stack

## Backend

- FastAPI
- LangChain
- FAISS
- BM25 Retriever
- yt-dlp
- youtube-transcript-api

## AI

- Groq
- LLaMA 3.3 70B
- BAAI/bge-small-en-v1.5
- faster-whisper

## Frontend

- Next.js 14
- React

---

# Local Setup

## Backend

```bash
cd backend

pip install -r requirements.txt

cp .env.example .env

# Add GROQ_API_KEY

python main.py
```

## Frontend

```bash
cd files

npm install

npm run dev
```

---

# Lessons Learned

The hardest part of this project wasn't vector search or prompt engineering.

It was dealing with real-world engineering constraints:

- YouTube blocking cloud IPs
- Instagram authentication requirements
- Deployment-specific failures
- Retrieval quality tuning
- Latency trade-offs

Building the prototype was relatively straightforward.

Making it work reliably outside my laptop was where most of the learning happened.

---

# Final Thoughts

One thing this project reinforced is that building AI systems is often less about the model and more about everything around it.

Retrieval quality, data acquisition, deployment limitations, rate limits, and infrastructure decisions had a much larger impact on the final experience than swapping one LLM for another.

The current version is intentionally optimized for simplicity and experimentation, but the architecture leaves a clear path toward scaling if usage grows.
