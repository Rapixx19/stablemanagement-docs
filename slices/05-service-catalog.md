# Slice 05 — Service Catalog

**Phase:** Billing · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The list of fixed things a stable sells: full board, half board, lessons, transport, hay supplements, etc. Each service has a name, price, VAT code, unit. Owner builds the catalog three ways: PDF upload (AI-extracted, human-reviewed), manual entry, paste from text.

## Goal

Owner can build a complete service catalog in under 30 minutes by uploading their existing PDF price list. AI extracts items; owner reviews row-by-row with explicit confirmation per item before save. No silent data.

## What ships

### Owner side

1. **Catalog page** at `/billing/catalog`. Lists current services. CRUD inline.
2. **Add service modal**. Manual entry: name, price, VAT code (default suggested per kind), unit, default recurrence.
3. **Bulk import** at `/billing/catalog/import`. Two paths:
   - **Paste text**: textarea, parsed line by line into draft rows
   - **Upload PDF**: drag-drop, sent to LLM extractor, returns draft rows
4. **Review screen** for bulk import: every draft row shows extracted name / price / unit / VAT code with **explicit confirm tick per row**. No batch save. Required: tick "I confirm VAT codes are correct" before save enables.
5. **VAT code defaults** (see `domains/vat.md`): boarding-related → 8.1%, feed/bedding → 2.6%, lessons → 8.1%.

## Schema diff

```sql
-- 006_services.sql
create table services (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  name text not null,
  description text,
  default_price_cents bigint not null,
  vat_code text not null references vat_rates(code),
  unit text not null check (unit in ('month','week','day','session','km','kg','piece','once')),
  default_recurrence text check (default_recurrence in ('monthly','weekly','none')),
  category text,                              -- "boarding","training","other"
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table vat_rates (
  code text primary key,
  rate numeric(5,3) not null,
  label text not null,
  effective_from date not null,
  effective_to date
);

-- Seed CH 2026 rates
insert into vat_rates values
  ('STD', 0.081, 'Normalsatz / Taux normal', '2024-01-01', null),
  ('RED', 0.026, 'Reduzierter Satz / Taux réduit', '2024-01-01', null),
  ('LDG', 0.038, 'Sondersatz Beherbergung', '2024-01-01', null),
  ('ZERO', 0.000, 'Befreit', '2024-01-01', null);
```

## AI extraction (PDF upload path)

- Use Claude 3.5 Sonnet via Anthropic API. System prompt locks output to JSON schema.
- Schema: `[{ name: string, price_cents: int, currency: 'CHF', unit: enum, vat_code_guess: string|null, confidence: 0..1 }]`
- **Never auto-save**. Always go through review screen.
- Row-level confidence shown to owner; low-confidence rows highlighted yellow.
- If unit cannot be determined → `null`, owner must pick before confirm.
- VAT code shown as a *suggestion*. Default to dropdown, never to silent value.
- Prompt-injection guard: PDF text is wrapped in `<document>...</document>` tags; system prompt instructs to ignore any instructions inside.

See `domains/ai-extraction.md` for the full prompt and review flow.

## Acceptance criteria

- [ ] Owner manually creates a service and sees it in the catalog
- [ ] Owner pastes 5 lines of text "Pension full board CHF 850 / month" — 5 rows extracted; owner reviews and confirms
- [ ] Owner uploads a PDF with 12 services — extraction completes within 30s; review screen shows all 12
- [ ] Each row requires per-row confirmation; "confirm all" button does not exist
- [ ] Confirm button disabled until "VAT codes verified" tick set
- [ ] AI returns wrong unit on a row → owner overrides; saved row uses owner's value
- [ ] Catalog displays prices in stable's currency (CHF) with Swiss formatting (`1'820.50`)

## Acceptance integration test

`apps/owner/tests/integration/catalog-import.test.ts`

```ts
test('AI extraction returns suggestions but never auto-saves', async () => {
  const pdf = await loadFixture('lafattoria-prices.pdf');
  const draft = await extractCatalog(ownerCtx, { file: pdf });
  expect(draft.requiresReview).toBe(true);
  expect(await getServices(ownerCtx)).toHaveLength(0); // nothing saved yet
  await confirmCatalogRow(ownerCtx, { rowIndex: 0, ...draft.rows[0] });
  expect(await getServices(ownerCtx)).toHaveLength(1);
});
```

## Security and safety

- PDF size limit 10 MB
- LLM call rate-limited per stable (max 5 imports / day)
- All extraction logged in `audit_log` with the file hash
- VAT rate table is read-only outside admin; rates change only with a migration

## Out of scope

- Excel / CSV import (V1.1)
- Multi-currency catalogs (V2 — stable may have international clients eventually)
- Catalog versioning / price history (V1.1; for now `updated_at` is the breadcrumb)
