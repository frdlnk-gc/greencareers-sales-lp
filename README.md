# GreenCareers · Sales-Recruiting-LP

Statische Landing-Page + Quiz für die Sales-Manager-Stelle bei GreenCareers.

**Live:** https://karriere.green-careers.de

## Was hier liegt

- `index.html` — Landingpage (Hero mit Video-Hintergrund, Trust-Bar, Karrierepfad, Founder, FAQ, etc.)
- `quiz.html` — 8-stufiges Bewerbungs-Quiz mit Reject-Pfaden
- `assets/` — Bilder (Team, Founder, Hero-Poster) und Videos (Hero-Loop, Sales-Floor-Loop)
- `CNAME` — GitHub-Pages-Domain (karriere.green-careers.de)

## Wie der Quiz-Submit funktioniert

Quiz → POST an `https://gc-tracking-dashboard.vercel.app/api/recruiting/applications/webhook`
→ Bewerbung landet in Supabase-Tabelle `applications_sales`
→ Sichtbar im Dashboard unter `/dashboard/recruiting`

Webhook-Auth: `X-Webhook-Secret` Header (kein echtes Geheimnis, weil im Frontend — dient nur als Anti-Spam-Hürde).

## Hosting

GitHub Pages aus dem `main`-Branch. Bei Push automatisch live.

DNS: CNAME-Eintrag bei Ionos:
```
karriere  CNAME  frdlnk-gc.github.io
```
