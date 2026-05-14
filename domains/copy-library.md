# Domain: Copy Library (DE / FR / IT)

The authoritative trilingual string library. DE is canonical (per `DESIGN.md`); FR and IT translate from DE, not from English. Locked in `reviews/2026-05-07-ceo-review.md` as DES-1.

## Status

**V1 skeleton.** This doc holds 2–3 anchor strings per category that lock the style. A native-speaker pass before pilot fills in the remaining ~7 per category. The final corpus lives in `packages/i18n/locales/{de,fr,it}/` keyed by surface; this doc is the style spine and the lint reference.

The forbidden-phrase lint (slice 00, DES-3) treats this doc as the negative-space reference — anything in `DESIGN.md` "Forbidden" list fails CI in production string files.

## Categories

### 1. Invoice email subject

| DE | FR | IT |
|---|---|---|
| Rechnung {invoice_number} – {month_year} | Facture {invoice_number} – {month_year} | Fattura {invoice_number} – {month_year} |
| Erinnerung: Rechnung {invoice_number} überfällig | Rappel : facture {invoice_number} en retard | Sollecito: fattura {invoice_number} scaduta |
| Stornierung: Rechnung {invoice_number} | Annulation : facture {invoice_number} | Annullamento: fattura {invoice_number} |

### 2. Magic-link email subject

| DE | FR | IT |
|---|---|---|
| Ihr Anmelde-Link für {stable_name} | Votre lien de connexion pour {stable_name} | Il tuo link di accesso per {stable_name} |
| Anmelde-Link (gültig 15 Minuten) | Lien de connexion (valable 15 minutes) | Link di accesso (valido 15 minuti) |

### 3. Error toast

| DE | FR | IT |
|---|---|---|
| Speichern fehlgeschlagen – bitte erneut versuchen | Échec de l'enregistrement – veuillez réessayer | Salvataggio non riuscito – riprovare |
| Verbindung verloren – Daten werden bei Wiederherstellung synchronisiert | Connexion perdue – synchronisation à la reconnexion | Connessione persa – sincronizzazione al ripristino |

### 4. Empty state — first run

| Surface | DE | FR | IT |
|---|---|---|---|
| Invoice list | Noch keine Rechnungen versandt | Aucune facture envoyée | Nessuna fattura inviata |
| Catalog | Noch keine Leistungen erfasst | Aucune prestation enregistrée | Nessuna prestazione registrata |
| Today's tasks (worker) | Nichts geplant. Mach dir einen Kaffee. | Rien de prévu. Prends un café. | Niente in programma. Prendi un caffè. |
| Requests inbox | Keine offenen Anfragen | Aucune demande en attente | Nessuna richiesta in sospeso |

### 5. Empty state — filtered

| DE | FR | IT |
|---|---|---|
| Keine Treffer. Filter zurücksetzen. | Aucun résultat. Réinitialiser les filtres. | Nessun risultato. Reimposta i filtri. |

### 6. Confirmation dialog (destructive)

| Element | DE | FR | IT |
|---|---|---|---|
| Title (archive horse) | Pferd archivieren? | Archiver le cheval ? | Archiviare il cavallo? |
| Body | Diese Aktion kann nicht rückgängig gemacht werden. Die Audit-Spur bleibt erhalten. | Cette action est irréversible. La trace d'audit est conservée. | Questa azione non può essere annullata. Il tracciato di controllo viene mantenuto. |
| Destructive CTA | Archivieren | Archiver | Archiviare |
| Cancel CTA | Abbrechen | Annuler | Annulla |

### 7. Success toast

| DE | FR | IT |
|---|---|---|
| Gespeichert | Enregistré | Salvato |
| Rechnung versandt | Facture envoyée | Fattura inviata |
| Zahlung erfasst | Paiement enregistré | Pagamento registrato |

### 8. Validation error (inline, under the field)

| Field type | DE | FR | IT |
|---|---|---|---|
| Required | Pflichtfeld | Champ obligatoire | Campo obbligatorio |
| Invalid email | Ungültige E-Mail-Adresse | Adresse e-mail invalide | Indirizzo e-mail non valido |
| Invalid UID-MWST | UID-Format ungültig (Beispiel: CHE-123.456.789) | Format UID invalide (exemple : CHE-123.456.789) | Formato UID non valido (esempio: CHE-123.456.789) |

### 9. Button labels

| Action | DE | FR | IT |
|---|---|---|---|
| Save | Speichern | Enregistrer | Salva |
| Cancel | Abbrechen | Annuler | Annulla |
| Confirm | Bestätigen | Confirmer | Conferma |
| Send | Senden | Envoyer | Invia |
| Mark paid | Als bezahlt markieren | Marquer comme payé | Segna come pagato |
| Try again | Erneut versuchen | Réessayer | Riprova |

### 10. Table column headers

| Column | DE | FR | IT |
|---|---|---|---|
| Date | Datum | Date | Data |
| Client | Klient | Client | Cliente |
| Amount | Betrag | Montant | Importo |
| Status | Status | Statut | Stato |
| Period | Periode | Période | Periodo |
| Horse | Pferd | Cheval | Cavallo |

## Worker imperative-vs-noun form (slice 13 lock)

Per `reviews/2026-05-07-design-review.md`, worker task strings use **noun-form** Swiss-German barn vocabulary, not verb imperatives:

- **Right**: "Fütterung Aisle A · 07:00"
- **Wrong**: "Füttere Aisle A · 07:00"

FR: "Distribution foin couloir A · 07:00" (noun form). IT: "Distribuzione fieno corsia A · 07:00".

## Forbidden phrases (production string files only)

Mirrors `DESIGN.md` for grep clarity. Slice 00 lint greps `packages/i18n/locales/**/*.json` and fails CI on any of:

- `Welcome to`, `Willkommen bei`, `Bienvenue dans`, `Benvenuto in`
- `Unlock the power of`
- `Your all-in-one`
- `Let's get started`
- Any `!` in any string

## When this doc updates

- Native-speaker pre-pilot review pass completes → replace the "V1 skeleton" status note with pass date + reviewer name; expand each category to ~10 entries
- New surface added → add the relevant rows
- New forbidden phrase added to `DESIGN.md` → mirror here for the grep
