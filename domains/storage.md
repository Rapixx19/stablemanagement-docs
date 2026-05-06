# Domain: Storage — Files, Documents, Media

Where uploaded files live, how we keep tenants isolated, virus scanning, retention, cost control.

## Buckets

Three Supabase Storage buckets, all in Frankfurt:

| Bucket | Purpose | Public | Max file |
|---|---|---|---|
| `horse-documents` | Passports, vet reports, contracts | private | 25 MB |
| `horse-photos` | Profile photos, gallery | private | 10 MB |
| `invoices` | Generated PDFs | private | 5 MB |
| `attachments` | Message + broadcast attachments | private | 10 MB |

All private. Downloads via signed URL with 5-minute expiry.

## Path conventions

Folder structure encodes tenancy. The first folder segment is always `stable_id`. The RLS policy enforces the rest.

```
horse-documents/{stable_id}/{horse_id}/{document_id}.{ext}
horse-photos/{stable_id}/{horse_id}/{photo_id}.{ext}
invoices/{stable_id}/{invoice_id}.pdf
attachments/{stable_id}/{message_id}.{ext}
```

Never put two stables' files in the same folder. Never use sequential filenames; always UUID.

## RLS on Storage

Supabase Storage has its own policies. See `domains/rls.md` for the exact pattern. Summary: the policy reads the first folder segment and checks it's in `auth.current_stable_ids()`.

## Virus scan — ClamAV

Every upload routes through a Railway-hosted ClamAV scan endpoint:

1. Client uploads to a temp prefix (`tmp/{stable_id}/{uuid}`)
2. Server action invokes ClamAV scan via HTTP
3. On clean: move to permanent path, return final URL
4. On infected: delete tmp, log incident, return 422 to client
5. On scan error: delete tmp, return 503; user retries

ClamAV signature DB updated daily via `freshclam` cron in the Railway service.

## File type validation

MIME-type sniffing on the server (don't trust client `Content-Type`). Use `file-type` npm. Reject:
- `.exe`, `.dll`, `.bat`, `.sh`, `.app` and all executables
- Macro-enabled Office formats (`.docm`, `.xlsm`)
- Encrypted ZIPs (we can't scan inside)

Accepted in V1:
- PDF
- JPG, PNG, WebP
- Plain text TXT

No video, no audio, no Office docs in V1 (cost + scan complexity).

## Image processing

Photos go through `sharp`:
- Resize to max 2000 × 2000
- Strip EXIF (privacy: removes GPS, camera serial)
- Re-encode as WebP for storage savings
- Generate thumbnail at 400 × 400 for list views

Originals discarded after processing. Storage savings: ~70% vs raw JPEG.

## Retention

| Bucket | Retention |
|---|---|
| `horse-documents` | Indefinite (operational records) |
| `horse-photos` | Indefinite |
| `invoices` | 10 years (Swiss bookkeeping requirement) |
| `attachments` | 2 years, then archived to cold storage (V1.1 — V1 keeps indefinitely) |

Hard-deleted file: also remove from Storage. Soft-deleted (audit-preserved): remove from Storage but keep DB row referencing the path (file gone, metadata preserved).

## Cost monitoring

Storage cost is non-trivial at scale. Per-stable quota:

- Pilot: 1 GB free
- V1 paid tier: 5 GB included, then CHF 0.10 / GB / month

UptimeRobot + custom Sentry checks at 50% and 75% of stable quota → owner email + admin Slack alert.

## CDN and bandwidth

Supabase Storage has built-in CDN. Use signed URLs; cache headers set per asset:

- Photos: `Cache-Control: private, max-age=86400`
- Documents: `Cache-Control: private, max-age=300, must-revalidate`
- Invoice PDFs: `Cache-Control: private, max-age=3600`

## Upload UX

- Drag-drop zone on document upload, photo upload, message attachment
- Progress bar for files > 1 MB
- Optimistic preview for images
- Error retry with exponential backoff (1s, 3s, 9s)
- Server enforces size limit; client displays it pre-upload

## Streaming / chunked uploads

Files > 5 MB use Supabase Storage's resumable upload (TUS protocol). Handles connection drops gracefully. Library: `@supabase/storage-js`.

## Backup

Supabase Storage replicates within Frankfurt region. Cross-region backup not in V1 (cost). For disaster recovery, see `runbooks/disaster-recovery.md`. Customer-provided export endpoint in slice 16 satisfies nFADP portability.

## Hard rules

- File path always starts with `{stable_id}/`
- Never serve unsigned URLs
- Always virus-scan before final write
- Always strip EXIF on photos
- Validate MIME server-side, not client-side
- Reject files > documented limit, with clear error
- Log every upload to `audit_log` (uploader, file_path, size, mime, scan_result)

## Common pitfalls

- Forgetting to delete the tmp file on scan failure → leak
- Trusting client filename → use server-generated UUID
- Storing the file path in the DB before scan complete → race
- Generating signed URL too long-lived → security risk
- Not handling Supabase Storage 5xx (retry; don't lose the upload)
