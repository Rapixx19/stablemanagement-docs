# Journey — Worker's First Day

The most fragile install in the product. Worker is outdoors, on a phone, possibly with gloves, possibly behind barn-cat WiFi. If onboarding breaks here, the worker just uses paper and the platform never reaches the barn floor.

## Scene

Monday, 8 June 2026, 06:30. Marco arrives at La Fattoria for his first day. Owner added him as a worker the previous Friday afternoon. Marco's phone: iPhone 13, iOS 18, Safari default, slow 4G in the barn.

## The walkthrough — owner side

1. **Friday 17:00** — Owner opens `/owner/people/new`. Enters: name `Marco Bianchi`, type `worker`, email `marco@example.ch`, language `de`. Submit. `people` row + `invitations` row created. Resend queues the email.
2. **Friday 17:00:05** — Marco receives the email. Subject: "Stable Crew · Einladung von La Fattoria." He's on the train home; ignores it for now.

## The walkthrough — worker side

3. **Monday 06:32** — Marco walks to the barn. Remembers the email. Opens it on his phone. Taps "Anmelden."
4. **06:32:08** — Safari opens `/worker/home` via the magic link → JWT minted, role=worker, single membership → no role picker. Lands on the home page.
5. **06:32:10** — First-time install prompt at top of screen (slice 12): "Tipp: Marco, füge die App zum Startbildschirm hinzu. Tippe auf ⤴ und wähle 'Zum Home-Bildschirm hinzufügen'." Includes the arrow asset pointing at Safari's share icon.
6. **06:33** — Marco follows: taps the share icon, scrolls, finds "Zum Home-Bildschirm," taps. Confirms the app name "Stable Crew." Standalone icon appears on his home screen.
7. **06:33:30** — Taps the icon. App opens in standalone mode (no Safari chrome). Lands at `/worker/home` (the PWA `start_url`).
8. **06:33:35** — Time-clock consent screen (slice 12). Text in DE explains: "Wir speichern beim Einstempeln nur 'auf dem Hof' oder 'nicht auf dem Hof' — keine genauen Koordinaten." Marco accepts. `worker_consent` row inserted with the version of the consent text he saw.
9. **06:34** — Lands on Home. Empty state: pen-line illustration of a coffee cup + the copy `Nichts geplant. Mach dir einen Kaffee.` in `Inter Tight md` on warm paper. No tasks yet because nothing's assigned.
10. **06:34** — Owner at the kitchen table with coffee on his desktop opens `/owner/workers`. Sees Marco listed: "Online · gerade angemeldet." Drags the 07:00 feed task template onto Marco's column.
11. **06:34:02** — Marco's phone receives the Realtime delta (slice 13). Empty-state illustration crossfades out (220ms reveal motion). Today's Tasks block appears with "Fütterung Aisle A · 07:00" — round check circle, italic subtitle, 44px tap row.
12. **06:34:10** — Marco taps the clock-in card. Browser asks for location permission. Approves. Geofence check returns `within=true`. Live timer starts in JetBrains Mono. Moss-green "auf dem Hof" pill in header.
13. **07:00 — 07:30** — Marco feeds Aisle A. At 07:30 taps the check circle on the Fütterung task. 140ms tap-response fill. Task slides to "done" section using the 260ms list-reorder motion.
14. **End of day** — Marco clocks out. Owner sees the hours log auto-populate in `/owner/workers/{id}/hours`.

## What this surfaces

- **The iOS install prompt with the arrow asset is load-bearing**: without it, workers stop at "Safari shows a website" and never install the PWA. Worth the screenshot effort to nail.
- **Empty-state copy sets the entire worker voice**: `Nichts geplant. Mach dir einen Kaffee.` is warmer than any standard SaaS empty state. The worker reads this and understands the product cares about them.
- **Realtime delivery under 2 seconds is visible to the worker**: owner adds a task, worker sees it, the loop closes. This is the moment the product earns trust.
- **Geofence consent must frame what we don't store** (raw coords), not just what we do. nFADP-aware language reduces friction.
- **Owner's "Online · gerade angemeldet" status closes the same loop in reverse** — the platform feels alive on both sides.

## Failure paths

- **Email never arrives** (spam, `gmx.ch` deliverability quirks): owner sees no `accepted_at` on the invitation after 24h. Can resend via the invitations panel, or hand-share the link via WhatsApp temporarily.
- **Marco skips the install**: app still works in Safari but no home-screen icon and no push notifications (V1.1). Owner can prompt at next sign-in. Mobile bundle budget still applies.
- **Location permission denied**: time-clock works but every punch has `geo_within_fence=null`. Owner gets the approval queue (slice 14 acceptance). Marco can re-grant in Safari settings.
- **No signal in the barn**: IndexedDB queues task completions; reconciles on reconnect (slice 13 EDGE-6). Pending pill on each task while offline; clears when server confirms.
- **Marco's first day is a Sunday with no scheduled shift**: home shows the empty state, no tasks. Owner can DM via worker chat (slice 13) or wait for Monday.

## Success criteria

- From email-tap to standalone PWA icon: < 90 seconds
- First task assignment visible on Marco's phone: < 2 seconds after owner drops it
- Worker bundle < 150 KB gzip (slice 12 budget)
- Marco doesn't ask "wie funktioniert das?" — the UI is self-explanatory in DE
- Marco's first clock-in succeeds without owner intervention
