# Judges Said

**A German labour-law precedent explorer.** Describe an employment situation in plain German or
English and get the real court decisions that match — each citing the court, the Aktenzeichen, the
decision date, and the statutory provisions the ruling turns on.

It shows **precedent, never predictions**. That is a compliance boundary enforced in code, not a
stylistic preference.

**▶ Live: [judges-said-l49g.onrender.com](https://judges-said-l49g.onrender.com)** — the API runs separately
at `judges-said.duckdns.org`. The database sleeps after a week of inactivity, so a first request
after a long idle period can be slow while it wakes.

```
"mein Arbeitgeber hat mir während der Probezeit während einer Krankschreibung gekündigt"

  → Arbeitsgericht Aachen · 8 Ca 1327/16 d · 08.09.2016 · Urteil
       § 622 BGB · § 46 ArbGG · § 5 BUrlG
  → Arbeitsgericht Bonn · 4 Ca 139/16 · 12.04.2017 · Beschluss
       § 181 BGB · § 5 ArbGG · § 48 ArbGG · § 17a GVG
  → Arbeitsgericht Düsseldorf · 1 Ca 18/24 · 28.06.2024 · Urteil
       § 1 KSchG · § 4 KSchG · § 168 SGB IX · § 22 AGG
```

*Real decisions from the indexed corpus. Court, Aktenzeichen, date, and cited provisions are
stored fields, not generated text; the § lists are abbreviated here for width.*

---

## Why it exists

German legal research sits behind Beck-online and juris subscriptions. Employees, small-business
HR staff, and works-council members are exactly the people who need to know how courts have
actually ruled on their situation, and exactly the people who cannot justify the licence fee.

Court decisions and statutes carry **no copyright at all** in Germany (§ 5 UrhG, *amtliches Werk*).
The evidence is public domain. Only access to it is expensive.

---

## What it does

| | |
| --- | --- |
| **Hybrid retrieval** | Semantic (pgvector HNSW) and lexical (Postgres German full-text search) legs, fused with Reciprocal Rank Fusion |
| **Bilingual queries** | Ask in German or English — the semantic leg works cross-lingually, the lexical leg gets German terms |
| **Bilingual answers** | Switch answer language without re-running retrieval, so both versions cite the same decisions |
| **Verbatim German evidence** | Court names, Aktenzeichen, ECLI, dates, § references, and quoted passages are never translated |
| **Grounded answers only** | Every claim is checked against the retrieved German source text before delivery; unverifiable answers are refused |
| **Refuses out of scope** | Outcome predictions and person/employer searches are rejected before a model is ever called |

---

## Stack

| Layer | Choice |
| ----- | ------ |
| Backend | Python 3.12+ · FastAPI · Uvicorn · Pydantic v2 |
| LLM orchestration | PydanticAI (typed output, bounded tools) |
| LLM provider | OpenAI — answers and thread titles only |
| Embeddings | `intfloat/multilingual-e5-base` (768-dim) run **locally** via sentence-transformers |
| Frontend | Vite · React 19 · TypeScript · Tailwind v4 · shadcn/ui · Vercel AI SDK (UI only) |
| Database | Supabase Postgres + `pgvector` |
| Retrieval | pgvector HNSW + Postgres `german` full-text search, fused with RRF |
| Migrations | SQLAlchemy models + Alembic |
| Auth | Supabase Auth, email only |
| Hosting | Hetzner Cloud VPS (backend, Docker + nginx + Let's Encrypt) · Render Static Site (frontend) |

No Next.js, no SSR, no separate managed vector database, and no LLM calls from the browser.
Embeddings never leave the machine — there is no external embedding API.

**→ [docs/Architecture.md](docs/Architecture.md)** covers the turn lifecycle, why the two retrieval
legs get different inputs, the grounding validator, the data model, and the API.

---

## The corpus

**Source: [Open Legal Data](https://de.openlegaldata.io)**, a non-profit German legal open-data
project whose `robots.txt` explicitly directs consumers to its API, dumps, and HuggingFace datasets
— bulk use is invited, not merely tolerated.

Currently indexed — the pilot slice is **labour jurisdiction only** (`Arbeitsgerichtsbarkeit`):

| | Documents | Chunks | Coverage |
| --- | --- | --- | --- |
| Court decisions | 1,070 | 33,581 | 42 courts, decisions from 1999–2025 |
| Statute sections | 5,680 | 7,844 | 18 statute books |
| **Total** | **6,750** | **41,425** | |

Decision full text arrives as clean HTML, so there is **no OCR stage anywhere** in the pipeline.
Ingestion is idempotent and resumable; embedding is the whole cost, at roughly 4.5 chunks/sec on
CPU.

See **[docs/Corpus_Download_Guide.md](docs/Corpus_Download_Guide.md)** for the download walkthrough
and the Open Legal Data API quirks worth not rediscovering.

---

## Getting started

**Prerequisites:** Python 3.12+ and [uv](https://docs.astral.sh/uv/) · Node 20+ and pnpm · a
Supabase project · an OpenAI API key · a HuggingFace token (read scope) for the corpus dumps.

### 1 — Corpus

```bash
export HF_TOKEN=hf_...              # PowerShell: $env:HF_TOKEN = "hf_..."

uv run scripts/download_dump.py     # court decisions  → data/downloads/cases/
uv run scripts/download_laws.py     # statute sections → data/downloads/laws/
```

### 2 — Backend

```bash
cd backend
cp .env.example .env                # Supabase, DATABASE_URL, OPENAI_API_KEY
uv sync
uv run alembic upgrade head
uv run ingest_to_db.py --limit 50   # smoke run; omit --limit for everything
uv run uvicorn app.main:app --reload
```

Use the **direct** Supabase connection (port 5432), not the transaction pooler (6543): migrations
create extensions and build HNSW indexes, which need session-level access.

### 3 — Frontend

```bash
cd frontend
cp .env.example .env                # VITE_API_BASE_URL, VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
pnpm install
pnpm dev                            # http://localhost:5173
```

### Tests

```bash
cd backend     && uv run pytest     # 104 tests, no network or database required
cd ../frontend && pnpm lint && pnpm build
```

The grounding, retrieval, and ingestion tests are deliberately LLM-free — a control that can only
be verified by asking a model is not a control.

---

## Legal boundaries

Requirements, not suggestions. Two are enforced in code.

1. **RDG (Rechtsdienstleistungsgesetz).** Retrieving and citing decisions is *information*.
   Assessing a specific person's case is a regulated *legal service*. The product stays on the
   legal side of that line by surfacing analogous rulings and refusing to predict outcomes —
   enforced in the grounding validator, because a prompt is not a compliance control.
2. **No crawling of official court portals.** `rechtsprechung-im-internet.de` is `Disallow: /` for
   every agent except `DG_JUSTICE_CRAWLER`. Case law comes from Open Legal Data only, and the
   downloaders are rate-limited with an identifying User-Agent regardless.
3. **§ 5 UrhG provenance** is recorded per document in `license_note` rather than left implicit.
4. **Personal data.** Decisions are pseudonymized, not anonymized, and labour decisions routinely
   carry **GDPR Article 9 special-category data** — health in sick-pay cases, union membership in
   works-council cases. Residual identifiers are redacted at ingestion before anything becomes
   searchable, personal identifiers are never indexed as searchable fields, and aggregate
   person-queries ("has anyone sued employer X") are refused outright.

**Non-goals:** predicting case outcomes · individual legal advice or document drafting · person or
employer search · crawling court portals · translating the corpus · languages beyond German and
English · multi-tenancy, billing, or paywalls · a mobile app.

---

## Repository layout

```text
Judges_said/
├── README.md      this file
├── docs/          architecture, corpus guide, build checklists
├── scripts/       corpus downloaders (Open Legal Data → data/downloads/)
├── data/          corpus payloads (gitignored, re-downloadable)
├── backend/       FastAPI service
└── frontend/      React SPA
```

---

## Status

The vertical slice works end to end locally: sign in, ask in German or English, receive a grounded
answer with citation chips, open the cited passage, switch answer language, and get a clear refusal
when the corpus has no comparable case.

Remaining work is **deployment**: Railway services for both sides, migrations and ingestion against
production Supabase, structured logging on failed turns, and verification on the deployed URL.

---

## Attribution and licence

Court decisions and statutes are public domain under **§ 5 UrhG** (*amtliches Werk*). Corpus data
comes from **[Open Legal Data](https://de.openlegaldata.io)**, whose provenance is recorded per
document in `license_note`.

This tool provides **legal information, not legal advice**. It surfaces how German courts have
ruled in comparable cases. It does not assess anyone's individual case and does not predict how any
dispute will turn out.

## Demo
https://github.com/user-attachments/assets/cbfa0f90-aab7-48d8-809e-c605d569d227

