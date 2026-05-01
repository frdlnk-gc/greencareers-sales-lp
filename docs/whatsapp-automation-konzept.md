# WhatsApp-Automation für Sales-Bewerbungen — Konzept

**Stand:** 2026-05-01
**Ziel:** Bewerber landet im Quiz → bekommt WhatsApp-Nachricht (am nächsten Werktag um 8:30) → automatisierter Bot terminiert ihn auf einen freien 15-Min-Slot in Frederik's Kalender → Termin wird gebucht und bestätigt.

---

## Big Picture

```
[Quiz Submit]
   ↓ (Webhook)
[Supabase: applications_sales]
   ↓ (cron / scheduled trigger am nächsten Werktag 8:30)
[WhatsApp Cloud API → erste Nachricht]
   ↓ (Bewerber antwortet)
[LLM-Bot verarbeitet Antwort, schaut Kalender, schlägt Slot vor]
   ↓ (Bewerber bestätigt)
[Google Calendar: Termin gebucht + Mail-Einladung]
   ↓
[WhatsApp Bestätigung]
   ↓
[Status in Supabase: "15min_geplant"]
```

---

## Tech-Stack

| Aufgabe | Tool | Warum |
|---|---|---|
| **WhatsApp-Versand** | WhatsApp Cloud API (Meta direkt) | 1.000 Conversations/Monat gratis, offiziell, kein Drittanbieter-Lock-in |
| **Workflow-Engine** | n8n self-hosted (auf Railway oder Vercel-Funktion) | Kostenlos, visuelle Flows, kann Schedule + LLM + APIs |
| **LLM-Bot-Logik** | OpenAI GPT-4o-mini oder Claude Haiku 4.5 | Cents pro Conversation, schnell |
| **Calendar** | Google Calendar API | Kostenlos, offiziell, Frederik's Kalender |
| **Datenbank** | Supabase (haben wir) | Eine zentrale Source of Truth |

**Geschätzte Kosten:**
- WhatsApp: 1k gratis/Monat, danach ~$0.05 pro Conversation in DE
- OpenAI/Claude: <$0.01 pro Conversation
- n8n: kostenlos (self-host)
- Calendar: kostenlos
- **= praktisch null bei <1.000 Bewerbern/Monat**

---

## Setup-Schritte (Reihenfolge)

### 1. WhatsApp Cloud API einrichten (1-2 Tage Wartezeit)

**Vorher prüfen:**
- [ ] Meta-Business-Account vorhanden? Falls nicht: business.facebook.com
- [ ] Telefonnummer bereit, die für WhatsApp Business genutzt werden kann (NICHT die persönliche Frederik-Nummer, sondern eine separate — z.B. Twilio-Nummer oder eine Zweit-SIM)

**Schritte:**
1. business.facebook.com → "WhatsApp" hinzufügen
2. App erstellen unter developers.facebook.com → Type: Business
3. WhatsApp-Produkt zur App hinzufügen
4. Phone Number registrieren + verifizieren
5. Permanent Access Token generieren (NICHT den 24h-Test-Token!)
6. Meta-Approval für **Marketing-/Utility-Templates** beantragen (für die initiale Nachricht — dauert 1-3 Tage)

**Was wir am Ende haben:**
- `WHATSAPP_PHONE_ID` (von Meta)
- `WHATSAPP_TOKEN` (Permanent)
- Geprüfte Templates (z.B. `bewerbung_initial_kontakt`)

### 2. Template definieren

WhatsApp ERLAUBT initiale Nachrichten an User NUR über genehmigte Templates. Unser Template:

```
Template-Name: bewerbung_initial_kontakt
Sprache: de
Body:
Hi {{1}},
Frederik hier von GreenCareers.

Ich melde mich, weil du dich bei uns als Sales Manager beworben hast — die Stelle: https://karriere.green-careers.de

Ist die Stelle noch interessant für dich?

Beste Grüße
Frederik
```

`{{1}}` = Vorname-Variable.

### 3. Workflow-Engine wählen

**Empfehlung: n8n self-hosted auf Railway** ($5/Monat oder gratis Trial)

Alternative: Make.com (komplexer für unseren Use-Case, aber GUI ist anfänger-freundlicher)

### 4. Flow 1: Erste Nachricht am nächsten Werktag

**Trigger:** Schedule (jeden Werktag 8:30 Uhr Berlin-Zeit)

**Logik:**
```
1. Hole Supabase-Bewerbungen mit:
   - status = "neu"
   - created_at >= gestern UND <= heute morgen
2. Für jede Bewerbung:
   - Sende WhatsApp-Template "bewerbung_initial_kontakt" mit {{1}} = vorname
   - Update Supabase: status = "whatsapp_initial_sent"
   - Logge Timestamp
```

### 5. Flow 2: Antwort-Verarbeitung mit LLM

**Trigger:** WhatsApp Webhook (Meta sendet bei jeder eingehenden Message)

**Logik:**
```
1. WhatsApp-Webhook empfangen → User-Number, Message-Body
2. Hole Bewerbung aus Supabase (Match über telefon-Feld)
3. Hole Conversation-History aus Supabase (neue Tabelle: whatsapp_messages)
4. Hole verfügbare Calendar-Slots (siehe Slot-Logik unten)
5. LLM-Call (System-Prompt + Conversation + Slots) → Antwort + Action
   Mögliche Actions: ANSWER (nur antworten), BOOK_SLOT (Termin buchen),
   ESCALATE (an Frederik weiterleiten)
6. Bei BOOK_SLOT:
   - Google Calendar: Event erstellen (siehe unten)
   - Status in Supabase: "15min_geplant"
   - Sende Bestätigungs-WhatsApp
7. Bei ANSWER: Sende LLM-Antwort als WhatsApp
8. Bei ESCALATE: Push-Notification an Frederik (Mail oder Slack)
9. Speichere alle Messages in whatsapp_messages-Tabelle
```

### 6. Slot-Logik (Google Calendar)

**Konfiguration:**
- Verfügbare Tage: Mittwoch + Donnerstag (oder Mo+Mi+Do — final entscheiden)
- Verfügbare Zeit: 15:00 – 18:00
- Slot-Dauer: 15 Min
- Puffer: 5 Min zwischen Slots
- Slots: 15:00–15:15, 15:20–15:35, 15:40–15:55, 16:00–16:15, 16:20–16:35, 16:40–16:55, 17:00–17:15, 17:20–17:35, 17:40–17:55
- = **9 Slots pro Tag × 2 Tage = 18 Slots/Woche**

**Verfügbarkeits-Check:**
```python
für jeden potentiellen slot:
  → Google Calendar API: events?timeMin=slot_start&timeMax=slot_end
  → wenn keine Events drin (auch keine "verfügbar"-Blocker): Slot ist frei
```

**Wichtig:** Frederik blockt seinen Kalender mit `"transparency: transparent"` Events für Zeiten, die er PRIVAT braucht — die werden bei freebusy.query NICHT als "busy" erkannt → Slot bleibt frei. Für echte Termine nutzt er reguläre Events.

### 7. Termin-Buchung

```javascript
calendar.events.insert({
  calendarId: 'primary',
  resource: {
    summary: `Tel. Kennenlernen ${vorname} ${nachname} & Frederik Linke von GreenCareers GmbH`,
    description: `15-Min Telefonat. Frederik ruft an: ${telefon}\n\nBewerbung: https://gc-tracking-dashboard.vercel.app/dashboard/recruiting/${application_id}`,
    start: { dateTime: slot_start_iso, timeZone: 'Europe/Berlin' },
    end:   { dateTime: slot_end_iso,   timeZone: 'Europe/Berlin' },
    attendees: [{ email: bewerber_email, displayName: `${vorname} ${nachname}` }],
    // KEIN conferenceData → kein Google Meet Link, weil Telefon-Termin
  },
  sendUpdates: 'all'  // Bewerber bekommt Mail-Einladung
})
```

### 8. WhatsApp-Bestätigung nach Buchung

```
Perfekt {{vorname}}, ich habe den Termin eingetragen.

📞 Telefonat am {{datum}} um {{uhrzeit}}
Ich rufe dich auf der Nummer {{telefon}} an.

Du hast eine Bestätigung per E-Mail bekommen.

Bis dahin!
Frederik
```

---

## LLM-System-Prompt (Skizze)

```
Du bist ein freundlicher, professioneller Recruiting-Bot für GreenCareers.
Du sprichst im Namen von Frederik Linke (Founder) per WhatsApp mit Sales-Bewerbern.

Dein Job:
1. Wenn der Bewerber bestätigt, dass die Stelle interessant ist → biete einen 15-Min-Telefon-Termin an.
2. Verfügbare Slots: {{verfügbare_slots}}
3. Bei Zustimmung zu einem Slot → ACTION = BOOK_SLOT mit gewünschtem datetime
4. Bei Verschiebungswunsch → schlage anderen Slot vor
5. Bei Off-Topic-Fragen oder Ablehnung → ACTION = ESCALATE (Frederik übernimmt)
6. Bei No-Reply oder unklaren Antworten → freundlich nachhaken

Stil: Du-Form, kurz, präzise, freundlich. Verwende keine Skript-Floskeln.
Wenn der Bewerber emotional/unsicher schreibt → menschlich antworten, nicht Bot-mäßig.

Du darfst NICHT:
- Andere Themen als die Stelle und das 15-Min-Kennenlernen besprechen
- Slots erfinden, die nicht in der Liste stehen
- Versprechen über Gehalt, Closer-Aufstieg, etc. machen — das macht Frederik im Call

Antwort-Format (JSON):
{
  "action": "ANSWER" | "BOOK_SLOT" | "ESCALATE",
  "message": "Antwort an den Bewerber",
  "slot_datetime": "2026-05-08T15:00:00+02:00"  // nur bei BOOK_SLOT
}
```

---

## Datenbank-Erweiterung (Supabase)

```sql
-- Neue Tabelle für WhatsApp-Conversations
create table public.whatsapp_messages (
  id uuid primary key default gen_random_uuid(),
  application_id uuid references public.applications_sales(id) on delete cascade,
  direction text not null check (direction in ('outbound', 'inbound')),
  body text,
  whatsapp_message_id text,
  status text,  -- sent, delivered, read, failed
  created_at timestamptz default now(),
  raw_payload jsonb
);

create index idx_wa_app_id on public.whatsapp_messages(application_id);
create index idx_wa_created on public.whatsapp_messages(created_at desc);

-- applications_sales erweitern
alter table public.applications_sales
  add column if not exists whatsapp_initial_sent_at timestamptz,
  add column if not exists scheduled_call_at timestamptz,
  add column if not exists calendar_event_id text;
```

---

## Edge-Cases zu klären

1. **Bewerber antwortet nicht.** Nach 24h Reminder? Nach 72h "Letzte Nachricht"? Auf Status `self_dropout` setzen?
2. **Bewerber hat unflexible Zeiten.** Bot versucht 3× alternative Slots, sonst → ESCALATE.
3. **Mehrere Bewerber wählen denselben Slot gleichzeitig.** Race-Condition. Lösung: vor Buchung erneut Calendar prüfen.
4. **Bewerber will nicht per Telefon.** Bot bietet Google-Meet-Slot an? → ESCALATE.
5. **Bewerber stellt inhaltliche Fragen** (Gehalt, Aufgaben). Bot beantwortet NICHT — verweist auf den 15-Min-Call ("Kannst du da am besten direkt mit Frederik klären, im Call.").
6. **Bewerber ist genervt vom Bot, will mit Mensch sprechen.** Bot erkennt Trigger ("ist das ein Bot?", "rede ich mit einem Roboter?") → ESCALATE sofort, ohne weitere Nachfrage.
7. **WhatsApp-Status (gelesen / nicht zugestellt).** Im Dashboard sichtbar machen.

---

## Nächste konkrete Schritte (für morgen)

**Du:**
1. Meta-Business-Account checken: ist einer da?
2. Zweite Telefonnummer für WhatsApp Business klären (NICHT die persönliche)
3. Account bei n8n.io oder Railway anlegen für Self-Hosting (alternativ: Make-Account)
4. Frederik's Kalender-Setup checken: Sind die Mi/Do 15-18 Slots aktuell frei oder müssen wir die manuell freischaufeln?

**Ich (sobald die obigen 4 Punkte klar sind):**
1. SQL-Migration für `whatsapp_messages` schreiben
2. n8n-Flow 1 (Initial-Send) bauen
3. n8n-Flow 2 (LLM-Bot) bauen
4. Calendar-Integration mit Slot-Logik
5. Test-Run mit deiner eigenen Nummer als Bewerber

**Geschätzter Zeitaufwand bis Produktiv-Start:**
- Meta-Approval-Wartezeit: 1-3 Tage
- Implementation: 1.5 Tage Arbeit
- = realistisch **bis Ende der Woche live**, wenn morgen die Account-Sachen geklärt sind.
