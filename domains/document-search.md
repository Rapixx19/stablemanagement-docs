# Domain: Document Search (Reducto + pgvector)

How every PDF, photo, and contract uploaded to Stableplatform becomes searchable by natural-language query. Locked 2026-05-14. Anchors slice 01b unified document vault.

## Pipeline

```
Upload → Storage write → Reducto extract → Embedding → pgvector index → Search
```

1. **Upload** — owner or client uploads a file via slice 01b document vault. Size ≤ 25 MB, allowed MIME `application/pdf`, `image/jpeg`, `image/png`. File written to Supabase Storage at `{stable_id}/{entity_kind}/{entity_id}/{document_id}.{ext}`. `documents` row inserted with `reducto_status='pending'`.
2. **Reducto extract** — Vercel Cron at `/api/cron/reducto-drain` (every 60 s) picks up `pending` rows, calls Reducto's `/extract` endpoint with the file URL (signed, 1h expiry). Reducto returns layout-aware text + detected language. Update row: `extracted_text`, `extracted_text_lang`, `reducto_job_id`, `reducto_status='processing'` while in flight, then `'done'` on success.
3. **Embedding** — once `extracted_text` lands, a follow-up job calls the embedding model (`text-embedding-3-small`, 1536 dim) on chunks of up to 512 tokens. We store a single embedding per document (average of chunk embeddings) in `documents.embedding`. For long docs (> 8 chunks), we keep a sidecar `document_chunks` table — see "Long documents" below.
4. **pgvector index** — `ivfflat` cosine-similarity index on `documents.embedding` (existing in slice 01b schema).
5. **Search** — `/owner/search?q=…` issues `SELECT ... ORDER BY embedding <=> $query_embedding LIMIT 20` filtered by `stable_id` (RLS) and `entity_kind` if scoped.

## Why Reducto, not just text extraction

PDFs from stables look ugly: scanned passports, faxed vet reports, contract PDFs with table layouts, hand-written notes photographed and saved as PDF. Plain `pdf-parse` gets ~60 % of the text wrong. Reducto's layout-aware extraction:

- Handles scanned PDFs (OCR built in)
- Preserves table structure
- Detects language (DE/FR/IT/EN per V1 scope)
- Returns clean Markdown that embeds well

Cost: ~$0.005 per page. La Fattoria pilot estimate: ~500 docs/year × 4 pages = 2000 pages = $10/year. Negligible.

## Search UX

Three entry points:

1. **On a horse profile** — search box scoped to that horse's docs only
2. **On a client profile** — scoped to that client's docs + their horses' docs
3. **Global** — `/owner/search` searches across all entities in the stable

Results show: matching document with file thumbnail, parent entity ("Horse: Diva" / "Client: Marta Keller"), snippet of `extracted_text` around the match, similarity score (0–1). Click to open the source PDF.

Query examples:
- "Diva influenza vaccination" → finds Diva's vet report from May
- "Marta Versicherung 2026" → finds Marta's renewed insurance contract
- "Hufschmied Akira letzter Termin" → finds Akira's most recent farrier record
- "Vertrag Familie Keller" → finds Keller boarding contract

## Long documents (V1.1)

V1 stores **one embedding per document** (averaged across chunks). Works fine for 1–4 page docs. For long contracts (10+ pages), this loses local detail.

V1.1 path: add `document_chunks (document_id, chunk_index, text, embedding)` table. Search ranks by chunk-level similarity, dedupes to one result per document. Not needed for pilot.

## Failure modes

| Error class | Trigger | UX | Recovery |
|---|---|---|---|
| `ReductoApiUnreachable` | Reducto 5xx / timeout | Doc stays `pending`, owner sees badge "Indexierung läuft" on the doc card | Cron retries 5× over 1h then DLQ; surfaces in `/owner/settings/integrations/reducto` |
| `ReductoExtractEmpty` | Reducto returns 0 characters (corrupt PDF, blank scan) | Doc shows "Kein durchsuchbarer Inhalt" badge | Owner uploads a better scan, or manually adds text via "Notizen am Dokument" field |
| `EmbeddingApiTimeout` | OpenAI 5xx | Doc stays in `processing` | Auto-retry, DLQ after 5 attempts |
| `ContentTooLarge` | File > 100 pages | Doc stored, indexing skipped (`reducto_status='skipped'`) | Owner sees "Zu lang für Suche — V1.1 splittet automatisch in Chunks" |

DLQ count surfaces on Owner Dashboard alerts band (slice 16) and in the dedicated settings page.

## Privacy + nFADP

- **Extracted text contains PII** by construction (names, dates of birth on passports, medical details on vet reports). Treated as personal data per `domains/data-retention.md`.
- **Redaction propagates** — when a person is redacted (right to erasure), all their `documents` rows get `redacted_at` set, `extracted_text` and `embedding` are scrubbed to NULL. File in Storage deleted; only the audit row remains.
- **Reducto data-residency** — Reducto processes in EU region; data not retained beyond the extraction job. Reducto's contract specifies no model-training use of customer data. Verify SOC 2 + DPA before pilot.
- **Owner consent on upload** — first upload screen shows: "Dieses Dokument wird durch unseren KI-Dienst (Reducto) verarbeitet, um den Inhalt durchsuchbar zu machen. Die Datei bleibt in der Schweiz/EU." with continue / cancel.

## Configuration

`packages/lib/document-search/` exposes:
- `enqueueExtraction(documentId)` — called from upload handler
- `searchDocuments({ stableId, query, entityKind?, entityId?, limit?, threshold? })` — returns ranked matches
- `embed(text)` — wraps OpenAI embedding API with caching

Vercel env vars:
- `REDUCTO_API_KEY` (sensitive)
- `OPENAI_EMBEDDING_API_KEY` (sensitive)
- `REDUCTO_API_BASE` (default `https://api.reducto.ai`)

## Cost cap

- Reducto: $50/month per stable hard block (≈ 10'000 pages, way above any real stable's volume)
- OpenAI embeddings: $20/month per stable hard block
- Both logged in `cron_runs` for the relevant cron job; alert at 50 % of cap

## Acceptance criteria (folded into slice 01b)

- [ ] Upload a 4-page vet PDF in DE → `reducto_status='done'` within 60 s; `extracted_text` populated; embedding generated; doc appears in search for query "vaccination"
- [ ] Search "Diva impfung" returns Diva's vet docs ranked by similarity, with snippet
- [ ] Stable A's docs never appear in stable B's search (RLS test)
- [ ] Client A's search at `/client/search` returns only docs about A's horses + A's own client docs
- [ ] `ReductoApiUnreachable`: mock 503 for 5 attempts → doc enters DLQ; banner on alerts panel
- [ ] Redaction: redactPerson(personId) sets `redacted_at` + nulls `extracted_text` + `embedding` for all their docs within the same TX
- [ ] Owner consent screen blocks upload until "Ich verstehe" is ticked
- [ ] Cost cap: stable hitting $50 Reducto spend in a month blocks further extractions; owner sees banner; existing search still works

## When this doc updates

- Reducto deprecates / replaced → swap the extract step; embedding pipeline unchanged
- Chunked-embedding (V1.1) ships → add "Long documents" implementation details
- Multi-modal search (image search within photos) → new section
