# RAGFlow on Coolify → wired to NVIDIA NIM

Self-hosted RAG for large pediatric textbooks. Ingest each book once; queries
retrieve only the relevant passages, so token cost scales with the *topic*, not
the 2,000+ pages.

Tuned for a 16 GB shared VPS (v6node). Uses the **slim** RAGFlow image because
all AI (chat, embeddings, rerank) is served by your NIM endpoint.

---

## 1. Put these files in a Git repo
Create a repo (e.g. `ragflow-coolify`) containing:
- `docker-compose.yaml`
- `.env.example` (reference only — do not put real secrets here)

## 2. Create the resource in Coolify
1. **New Resource → Docker Compose** → connect this Git repo.
2. Coolify reads `docker-compose.yaml` automatically.
3. Go to **Environment Variables** and add the real values from `.env.example`
   (generate each with `openssl rand -hex 24`).
4. On the **`ragflow`** service, set the domain to `ragflow.drkathiravan.uk`
   and target port **80**. Leave the other services with no domain.
5. Point your Cloudflare Tunnel hostname `ragflow.drkathiravan.uk` at Coolify's
   proxy (same pattern as your other services).

## 3. Deploy + create admin
- Deploy. First boot takes 2–3 min while Elasticsearch goes healthy.
- Open `https://ragflow.drkathiravan.uk`, register the **first** account — it
  becomes admin.
- After that, set `REGISTER_ENABLED=0` in Coolify env and redeploy to close signups.

## 4. Wire NVIDIA NIM (in the RAGFlow UI)
Avatar (top-right) → **Model providers** → add **OpenAI-API-Compatible**:
- **Base URL:** `https://integrate.api.nvidia.com/v1`
- **API key:** your `nvapi-...`

Add three models under that provider:
- **Chat** — your daily-driver NIM model (e.g. the MiniMax M2.7 you use, or
  `meta/llama-3.3-70b-instruct`). This writes the summaries.
- **Embedding** — `nvidia/llama-3.2-nv-embedqa-1b-v2` (or `baai/bge-m3` for
  strong multilingual). This indexes the books.
- **Rerank** — `nvidia/llama-3.2-nv-rerankqa-1b-v2`. This is the single biggest
  lever for "don't omit anything" — it re-scores retrieved chunks so the right
  passages surface.

Then **System Model Settings** → set the defaults to the three models above.

## 5. Ingest the books
1. **Knowledge Base → Create.**
2. Chunking method: **Book** (preserves chapter/heading hierarchy).
3. Embedding model: the NIM embedding model.
4. Upload PDFs → **Parse**. Big books take a while on CPU — do this during
   low-usage windows (see memory notes).
5. Inspect the parsed chunks in the UI to confirm nothing got mangled.

## 6. Tune for maximum recall (comprehensive summaries)
In the Knowledge Base / Assistant retrieval settings:
- **Top N (top-k):** 16–32 (default ~6 is too low for "leave nothing out").
- **Similarity threshold:** low, ~0.1–0.2, so borderline-relevant passages
  still get pulled.
- **Keyword similarity weight:** raise it — this is the hybrid search that keeps
  exact drug names / dosages / eponyms from being missed.
- **Rerank model:** the NIM reranker. Retrieve wide, then let rerank tighten.
- **Prompt:** tell the assistant to summarize the retrieved context
  *section by section without dropping clinical detail, dosages, or values.*

---

## Memory notes (important for your box)
Approx. steady-state footprint: ES ~4 GB, ragflow ~2–3 GB (spikes to ~5 GB while
parsing on CPU), MySQL ~1 GB, MinIO + Redis ~1 GB → **~8–9 GB idle, ~11 GB
mid-ingest.** That leaves headroom, but during the *initial* parse of several
2,000-page books, pause the heavy stuff (Jellyfin transcodes, Neko) so the
DeepDoc parser isn't starved. Once books are indexed, ongoing query load is light.

If it still feels tight, switch the doc engine to **Infinity** (lighter than
Elasticsearch): set `DOC_ENGINE=infinity` and swap the `es01` service for
RAGFlow's Infinity service — see upstream docs.

## If a container won't go healthy
Image/tag drift is the usual cause. Fall back to RAGFlow's official `docker/`
folder pinned to the same tag, and drop in **your `.env` values** (the memory
tuning is the part that matters):

```
git clone https://github.com/infiniflow/ragflow.git
cd ragflow && git checkout v0.26.2
# edit docker/.env → set MEM_LIMIT and passwords, then point Coolify at docker/
```

Note: RAGFlow images are **x86-only** — fine for your KVM VPS.
