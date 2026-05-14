# Domain: Scaling (V1.1+ Watch List)

Things we're not optimizing for in V1 but will need to think about. Locked in `reviews/2026-05-07-eng-review.md` as E-PERF-4 plus several long-term items.

## Scope

**Not V1 work.** This doc exists so we don't forget what's coming. Default: don't preemptively optimize. Revisit when a metric crosses a threshold below. Pilot is one stable; everything here is "what happens at 30 stables and beyond."

## Realtime channel fan-out

**Current**: Supabase Realtime subscriptions on `tasks` and `messages` filtered by `(stable_id, user_id, ...)`. One channel per worker on the today view, one per thread on comms.

**Watch threshold**: > 200 concurrent channels per stable. Pilot has ~5 workers + ~20 clients = ~25 channels. Won't hit in V1.

**V1.1 plan**: shard channels by `stable_id` partition, pre-compute event filters server-side, push pre-filtered payloads. Fallback if Realtime falls over: long-poll on `/api/worker/today` and `/api/comms/thread/[id]`.

## DB partition strategy beyond audit_log

**Current**: only `audit_log` is partitioned yearly (slice 00).

**Watch threshold**: any table exceeds 50M rows OR any single query crosses 500ms p95 due to seq scans.

**V1.1 candidates** (size at 30-stable scale, 5 years):
- `time_punches` ≈ 750k rows — not urgent
- `tab_lines` ≈ 600k rows — not urgent
- `messages` — can grow fast if any stable becomes chatty; likely first to need partitioning

Partition strategy: range on `created_at`, yearly for billing-class tables, quarterly for messages.

## LLM cost growth

**Current**: ~$0.10 per PDF extraction (slice 05), capped 5 imports/day per stable, $20/month hard block per stable. Prompt caching on system + schema (90% discount on static portion).

**Watch threshold**: aggregate monthly spend > $500 across all stables, OR > 30% of stables hit their $20 cap.

**V1.1 plan**:
- Paid SaaS tier with higher cap
- Switch low-confidence extractions to Haiku (already specified as fallback in `domains/ai-extraction.md`)
- Cache extracted catalogs by file hash (rare re-upload, but cheap insurance)

## Bexio rate-limit at multi-stable scale

**Current**: 50 req/sec per OAuth token (per-stable). Queue caps at 30 req/sec per stable.

**Watch threshold**: 100+ stables on Bexio simultaneously — per-token limit holds, but our combined egress IP might attract attention.

**V1.1 plan**: distributed queue with per-tenant priority, monitor 429 rate per stable + globally, escalate to Bexio sales for higher-tier limits if it becomes a constraint.

## Vercel function instance limits

**Current**: Fluid Compute reuses instances. PDF routes capped at 2 concurrent per instance (slice 08 E-PERF-3).

**Watch threshold**: > 100 invoices generated per minute (end-of-month for 50+ stables simultaneously).

**V1.1 plan**: dedicated PDF generation Vercel project with reserved capacity, or split PDF generation to a queue worker with a separate Vercel function tree.

## Connection pool exhaustion

**Current**: Supabase connection pooler in transaction mode, default pool size (15–30 typical).

**Watch threshold**: pooler waits > 50ms p95 OR active connections > 80% of pool.

**V1.1 plan**: pool size increase via Supabase plan upgrade; route cron jobs through direct connection (already direct per slice 00); long-term, consider PgBouncer side-car if Supabase pooler ceiling is hit.

## Frontend bundle size

**Current**: per-tree bundle budgets enforced in CI (slice 00 E-PERF-5). Worker < 150KB gzip.

**Watch threshold**: any tree's bundle exceeds budget by > 10%.

**V1.1 plan**: route-level code splitting, lazy-load heavy primitives (e.g., PDF preview pane), tree-shake `lucide-react` icon imports to per-icon imports.

## Storage cost (PDFs + horse documents)

**Current**: Supabase Storage, no archival. Each PDF ~100KB, ~200 invoices/stable/year = 20MB/stable/year. Horse documents bigger but rare uploads.

**Watch threshold**: aggregate Storage > 50GB OR per-stable > 5GB.

**V1.1 plan**: archival to Backblaze B2 / Cloudflare R2 for invoices > 2 years old, kept queryable via signed URL.

## Search performance

**Current**: ILIKE on people.full_name, horses.name. Linear scan in V1 (pilot has ~20 clients, ~30 horses → fine).

**Watch threshold**: > 5000 searchable rows in any single stable.

**V1.1 plan**: pg_trgm index + per-stable filter on the trigram-search query. No Algolia / Typesense in V2 unless a stable explicitly asks.

## When this doc updates

- A watch threshold is hit → corresponding V1.1 item moves to a real slice or into `domains/v1-deferred.md`
- A new scaling concern emerges in pilot → add a section here, not in a slice spec
- V1.1 ships → split this doc into "Resolved" + "Still watching"
