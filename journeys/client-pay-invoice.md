# Journey — Client Pays an Invoice

The other half of the closed loop. If clients can't pay without effort, the whole platform fails to replace the WhatsApp+manual-IBAN workflow.

## Scene

Tuesday morning, 1 June 2026, 10:00. Anna is on the train commuting to work. Her phone buzzes. Email subject: "Rechnung 2026-0142 – Mai 2026."

## The walkthrough

1. **10:00:00** — Opens email on iPhone Safari. Subject + body in DE (per `people.language` — Anna lives in Ticino but reads German). Body: "Liebe Anna, hier ist die Rechnung für Mai. Gesamt: CHF 1'820.50." + a `Rechnung ansehen` button.
2. **10:00:08** — Taps the button. Magic-link auto-login (token in URL → JWT → role=client → single membership → no role picker). Lands on `/client/invoices/142`. LCP under 2 seconds on 4G.
3. **10:00:12** — Sees the invoice detail: stable name + logo plaque in Source Serif 4. Her horse Bella's name. Period 2026-05-01 → 2026-05-31. Total `xl 28/36` in JetBrains Mono tabular. Line items table below.
4. **10:00:30** — Taps "PDF herunterladen." 800 KB download. Safari opens the PDF in its viewer.
5. **10:00:45** — Switches to UBS Mobile Banking. Taps the QR scanner. Points at the phone screen (or holds the printed PDF if she prefers paper). UBS recognises the QR — pre-fills:
   - Empfänger: La Fattoria, Lugano
   - IBAN: CH... (the stable's QR-IBAN)
   - Betrag: 1'820.50 CHF
   - Referenz: the 27-digit QRR
6. **10:01:00** — Anna reviews the pre-filled fields, taps "Bezahlen." FaceID. Done.
7. **10:01:05** — Closes the banking app. Forgets about it. Walks to her office.
8. **10:30 — async** — Our `/api/cron/bexio-poll` (every 30 min, slice 09) runs. UBS notified Bexio (camt.054 SEPA notification). Bexio reports the invoice paid. Our poll flips `invoices.status='paid'`, `paid_at=NOW()`, `paid_source='bexio'`. The change reaches Anna's app state via Realtime if she has the tab open; otherwise next render.
9. **Following week** — Anna logs in to check her tab for June. The May invoice in `/client/invoices` shows the moss-green `bezahlt` pill. She doesn't notice or care. The platform did the work invisibly.

## What this surfaces

- **Magic-link auto-login must work cross-device**: Anna's email is on her phone, not the device she first registered with. Token-not-session validation is the contract.
- **Email language follows `people.language`, not stable default**: Anna in Ticino reads German. Hard-coding to stable default would feel wrong.
- **QR-bill is the closed loop**: without QR, Anna manually types IBAN + amount + 27-digit reference into UBS. That's the workflow we replace; any QR validation regression (slice 08 SIX-validator gate) breaks the value prop.
- **Reconciliation latency is invisible**: 30-min poll lag means Anna never sees an in-flight "Zahlung ausstehend" state — she pays, the app eventually agrees. The product is calm.
- **No-Bexio fallback** (per the 2026-05-13 accounting choice): stables not on Bexio rely on owner manually marking paid; client experience is identical up to step 7.

## Failure paths

- **QR-bill schema error** (slice 08 `SwissQrSchemaError`): email never goes out; owner sees draft stuck in failed state with offending field highlighted. Anna sees nothing until owner fixes it. From her POV, the cycle just runs a day late.
- **Magic link expired** (15-min security window): Anna clicks an old link → "Link abgelaufen. Neuen anfordern." Tap → new link in 10s. Continues with one extra step.
- **Bexio reconciliation broken**: Anna paid, status stays `sent` for days. Owner sees sync-issues banner, falls back to manual mark-paid (slice 08 item 6). Owner emails Anna an apology copy if needed.
- **Anna's bank doesn't support QR scan**: she enters QRR + IBAN by hand from the PDF. Slower (1 min vs. 15 sec) but always works — the PDF prints all fields legibly.

## Success criteria

- From email-open to payment-confirmed in UBS: < 90 seconds
- Status reconciled in our app within 30 minutes
- Anna never sees an in-app error during the happy path
- Anna pays from her phone, not from a desktop
- Owner does not need to chase Anna with a reminder before the due date (mostly)
