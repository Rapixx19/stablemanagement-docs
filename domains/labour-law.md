# Domain: Swiss Labour Law — Time Clock Compliance

The compliance map for slice 14 (time clock + geofence). Pilot at La Fattoria is internal use, low risk; formal Treuhänder review required before first external paying stable.

## Legal framework

Two laws govern working-time records:

| Law | Authority | Topic |
|---|---|---|
| Arbeitsgesetz (ArG), Art. 46 | SECO | Employer must record working time |
| Arbeitsgesetzverordnung 1 (ArGV 1), Art. 73 | SECO | What records contain, retention 5 years |
| nFADP (Federal Act on Data Protection, in force 1 Sep 2023) | FDPIC | Personal data handling |
| Code of Obligations (OR) Art. 321a et seq. | — | Employee fidelity, employer duties |

## What ArGV 1 Art. 73 requires

Working-time records must contain, per employee per day:

- Daily working time (start, end, breaks)
- Day off (if applicable)
- Vacation, sickness, military service, etc.
- Special compensations (overtime hours)

Records retained **5 years** from the end of the pay period they relate to.

## Where we satisfy it

| Requirement | How we cover it |
|---|---|
| Daily start / end | `time_punches` rows with `kind in (in, out)` |
| Breaks | `time_punches` with `kind in (break_start, break_end)` |
| Auto-close | `kind = auto_out` punch if worker forgot |
| Day off | `unavailability` row with `reason in (vacation, personal)` |
| Sick day | `unavailability` row with `reason = sick` |
| 5-year retention | DB retention policy + audit log immutability |
| Export | CSV from `/workers/[id]/hours` page |

## nFADP — geolocation

The geofence requires GPS access. nFADP rules:

| Principle | Our compliance |
|---|---|
| Lawfulness | Worker consents via `worker_consent` (slice 12) |
| Proportionality | Only check geofence boundary, never store coords |
| Purpose limitation | Geofence used only for time-clock verification |
| Data minimization | Boolean `geo_within_fence` only, never lat/lng |
| Information duty | Consent text in DE/FR/IT explains exact data flow |
| Right of access | Workers can export their punches |
| Right of rectification | Owner can annotate (immutable original); workers can dispute |
| Right of deletion | After 5-year retention, anonymized or deleted |

## The consent text (slice 12 ships this)

Required points (DE/FR/IT):

> This app records your working time for the stable's legal records.
>
> When you clock in or out, your phone briefly checks whether you are within ~100 meters of the stable. We store **only** "you were on-site" or "you were off-site" — never your actual location coordinates.
>
> If you punch from off-site, your supervisor is notified for review. They may approve it (e.g., transport day) or ask about it.
>
> Records are kept for 5 years per Swiss law (ArGV 1 Art. 73). You can request a copy or correction at any time.
>
> Click I understand to continue.

## Off-site punches

The product does not block them. A worker may legitimately work off-site (transport, competition, vet appointment). The system flags + notifies + lets owner approve. This matches Swiss labour-law principle that records must reflect actual hours, not artificially restricted ones.

## Owner overrides

Owner cannot delete a punch. Owner can:
- Annotate ("worked from competition site, approved")
- Approve an off-site punch (sets `approved_by_user_id`)
- Mark a punch as disputed (worker contests it)

The original punch row is immutable. All changes go to `audit_log`.

## Overtime

Currently **not** auto-calculated. CSV export gives the raw data; payroll software (Bexio Payroll, Comatic, Sage 50, etc.) handles overtime per CH labour law (max 50h/week for office workers, 45h for industrial/most others).

V1.1 may add an overtime banner ("this week so far: 47h, watch out").

## Sunday and night work

Special rules apply (premium pay, max hours, etc.). V1 does not enforce or compute. CSV export shows the raw timestamps; payroll handles. This is an acceptable trade-off for stable-management software (most stable workers are part-time or own-account, not under strict ArG schedules).

For stables hiring full-time grooms under ArG, V1.1 should add an "ArG-compliant mode" toggle.

## What we do NOT cover

- Wage compliance (minimum wage, GAV/CCT compliance) — out of scope
- Pension contributions (BVG) — out of scope
- Tax withholding — out of scope
- Foreign worker permits — out of scope
- AHV / AVS reporting — out of scope (deferred to payroll software)

We are working-time records, not full payroll.

## Treuhänder review

For external paying stables, before launch:
- Show product to a Swiss Treuhänder
- Confirm time-clock module satisfies Art. 46 + Art. 73
- Get a written sign-off on data flow + retention
- Update consent text to incorporate any feedback

Pilot at La Fattoria: skipped (family operation, low risk). Documented as outstanding work item.

## Audit log scope

Every punch + every annotation + every approval writes to `audit_log`:

```
{
  entity: 'time_punches',
  action: 'insert' | 'update' | 'annotate',
  before: ..., after: ...,
  actor_user_id, at, ip, user_agent
}
```

Read-only role for compliance audits: `compliance_readonly` Postgres role with SELECT on `time_punches`, `unavailability`, `audit_log` (slice 16 sets this up).

## Hard rules

- Raw GPS never persisted, even briefly
- Punches immutable; annotations only
- 5-year retention enforced (cannot purge before)
- Consent versioned; bumps require re-consent
- Off-site punches recorded, not blocked
- Owner cannot delete a worker's punch
- Audit log appended for every change
