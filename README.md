# WhatsApp AI Receptionist (Bookings)

*[العربية](README.ar.md)*

An AI receptionist that answers customers on WhatsApp, checks real-time availability, books the appointment straight into Google Calendar, and follows up with a confirmation and a reminder — no human needed for routine bookings.

**Built for:** salons, spas, dental clinics, barbershops/hair salons, physiotherapy clinics, and any appointment-based local business.
**Case study included:** *Lanna Grand Spa* (a spa & wellness salon in Oman) — see [Case study](#case-study-lanna-grand-spa) below.

---

## 1. What it does

```
Customer on WhatsApp
      │  "Hi, do you have a slot tomorrow at 5pm for a massage?"
      ▼
┌─────────────────────────────┐
│  WhatsApp AI Booking         │   1. Understands the request (Arabic or English)
│  Receptionist  (n8n)         │   2. Checks Google Calendar availability
│                               │   3. Books the appointment if free, or
│                               │      proposes alternative times if not
└─────────────────────────────┘
      │  "Confirmed! Swedish Massage tomorrow at 5:00 PM 🌿"
      ▼
Customer receives confirmation on WhatsApp
      │
      │  (~24h before the appointment)
      ▼
┌─────────────────────────────┐
│  Appointment Reminder        │   Hourly job scans Google Calendar and sends
│  Scheduler  (n8n)            │   a WhatsApp reminder, once per booking
└─────────────────────────────┘
```

Two n8n workflows ship in this repo:

| Workflow | File | Trigger | What it does |
|---|---|---|---|
| **WhatsApp AI Booking Receptionist** | [`n8n/whatsapp-ai-booking-receptionist.json`](n8n/whatsapp-ai-booking-receptionist.json) | Incoming WhatsApp message | Understands the customer's request, checks Google Calendar availability, books the slot, replies with a confirmation — all in one conversation. |
| **Appointment Reminder Scheduler** | [`n8n/appointment-reminder-scheduler.json`](n8n/appointment-reminder-scheduler.json) | Every hour | Finds bookings starting in ~24 hours and sends each customer one WhatsApp reminder (deduplicated, so nobody gets reminded twice). |

Both are built with the **n8n Workflow SDK** and are live, validated n8n workflows (not mockups) — see [Workflow internals](#4-workflow-internals) for how each node is wired.

---

## 2. Tech stack

- **[n8n](https://n8n.io)** — orchestration / workflow engine
- **WhatsApp Business Cloud API** (Meta) — the messaging channel
- **Google Calendar API** — availability + booking source of truth
- **OpenAI (GPT-5 family)** — the conversational AI Agent that drives the booking flow
- **n8n Data Table** — lightweight dedup store so reminders are sent exactly once per booking

No custom backend, database, or hosting is required beyond an n8n instance (n8n Cloud or self-hosted).

---

## 3. Setup guide

### 3.1 Prerequisites

1. An **n8n instance** (n8n Cloud, or self-hosted v1.90+ for Data Table support).
2. A **Meta developer app** with the **WhatsApp Business Platform** product added, plus a WhatsApp Business phone number (see [Meta's quickstart](https://developers.facebook.com/docs/whatsapp/cloud-api/get-started)).
3. A **Google Cloud project** with the Google Calendar API enabled and OAuth consent configured, and a Google Calendar for the business (can be the owner's own calendar to start).
4. An **OpenAI API key**.

### 3.2 Credentials to create in n8n

| Credential name (used by the workflows) | Type | Where to get it |
|---|---|---|
| `WhatsApp Trigger account` | WhatsApp Trigger API | Meta App → Client ID + Client Secret. n8n auto-registers the webhook with Meta on activation — the "Verify token" Meta asks for is this **node's own ID**, not a value you invent. |
| `WhatsApp Business Cloud account` | WhatsApp API | Meta App → permanent access token + the WhatsApp Business Account phone number ID. |
| `Google Calendar account` | Google Calendar OAuth2 | Standard Google OAuth2 flow in n8n (Client ID/Secret from Google Cloud Console, then connect and authorize). |
| `OpenAI account` | OpenAI API | Your OpenAI API key. |

Create these under **n8n → Credentials** using exactly these names so the imported workflows auto-match them; otherwise reassign each node's credential after import.

### 3.3 Import the workflows

1. In n8n: **Workflows → Import from File**, pick [`n8n/whatsapp-ai-booking-receptionist.json`](n8n/whatsapp-ai-booking-receptionist.json).
2. Repeat for [`n8n/appointment-reminder-scheduler.json`](n8n/appointment-reminder-scheduler.json).
3. Create a **Data Table** named `whatsapp_reminders_sent` with three string columns: `eventId`, `customerPhone`, `sentAt`. The reminder workflow uses it to avoid sending the same reminder twice.

### 3.4 Configure for your business

In **WhatsApp AI Booking Receptionist**:
- Open the **AI Receptionist** node and edit the system message: replace the business name, service list, durations, prices and opening hours with the real ones.
- Both **Google Calendar** nodes default to `primary` (the connected account's own calendar) — point them at the real business calendar via the resource picker if it's a different one.
- The two WhatsApp credentials must be filled in before the trigger can activate.

In **Appointment Reminder Scheduler**:
- Point **Get Appointments ~24h Out** at the same business calendar.
- Replace `YOUR_WHATSAPP_PHONE_NUMBER_ID` on **Send WhatsApp Reminder** with the real `phone_number_id` from Meta.
- Keep the `Phone: <number>` line in booking descriptions if you ever create calendar events another way — the reminder workflow parses the phone number out of the event description.

### 3.5 Go live

Activate both workflows. Send a test WhatsApp message to the business number to confirm the booking flow end to end, then create a test appointment ~24-25 hours out to confirm the reminder fires.

---

## 4. Workflow internals

### WhatsApp AI Booking Receptionist

`WhatsApp Trigger → Normalize Message → Filter (text only) → AI Agent → Send WhatsApp Reply`

The **AI Agent** (OpenAI GPT-5.4-mini by default) has:
- **Conversation memory** keyed per WhatsApp number, so it remembers context across a customer's messages.
- **Check Calendar Availability** tool — calls Google Calendar's freebusy check before ever promising a slot.
- **Create Booking** tool — creates the calendar event (with customer name, phone and service embedded in the description) once a slot is confirmed free.

The agent is instructed to: reply in whichever language the customer used, never invent availability without calling the tool first, only book once per confirmed slot, and offer alternatives when the requested time is busy.

### Appointment Reminder Scheduler

`Schedule Trigger (hourly) → Get Calendar Events (24-25h window) → Extract Phone → Filter → Dedup check (Data Table) → Send WhatsApp Reminder → Mark as Sent (Data Table)`

Running hourly and scanning a rolling 24-25 hour window means every booking gets exactly one reminder pass a day ahead of time, and the Data Table dedup guards against double-sends if the schedule ever overlaps or re-runs.

---

## 5. Case study: Lanna Grand Spa

**Lanna Grand Spa** — a spa & wellness salon offering massage, facials, mani-pedi and hair styling — is the reference business baked into the default system prompt and README examples in this repo.

- **Problem:** front-desk staff spent hours a day on WhatsApp answering "are you free at X?", double-booking slots when messages were missed, and forgetting to remind clients — leading to no-shows.
- **Solution:** this WhatsApp AI Receptionist handles the entire booking conversation autonomously, checks the real calendar before confirming anything, and sends an automatic reminder 24 hours ahead.
- **Result (typical for this profile of business):** front desk reclaims hours per day, availability is always checked live (no double-bookings), and no-shows drop thanks to automatic reminders — while customers get instant replies in Arabic or English, any time of day.

Use Lanna Grand Spa's setup as the template: swap in the real service list, prices, durations and hours, connect the real calendar and WhatsApp number, and the same two workflows run as-is.

---

## 6. Pricing

| Market | Setup fee | Ongoing |
|---|---|---|
| **Local (Oman)** | 350–700 OMR (ر.ع) one-time setup | 40–70 OMR/month (hosting, monitoring, support) |
| **Online / international** | $89–149 one-time setup | — |

Setup covers: WhatsApp Business API onboarding, Google Calendar connection, tailoring the AI receptionist to the business's services/hours/tone, and testing. The monthly fee covers n8n hosting, OpenAI usage, and ongoing support/tweaks.

---

## 7. Repository layout

```
n8n/
  whatsapp-ai-booking-receptionist.json   # Import-ready n8n workflow: inbound WhatsApp -> AI Agent -> booking
  appointment-reminder-scheduler.json     # Import-ready n8n workflow: hourly reminder job
README.md                                  # This file
```

---

## 8. Customization ideas

- **Multiple staff / resources:** switch the calendar tools from a single `primary` calendar to a resource picker per staff member, and have the agent ask which stylist/therapist the customer wants.
- **Deposits for high-value bookings:** add a payment-link tool (e.g. Stripe) the agent calls before confirming bookings above a price threshold.
- **No-show tracking:** log every booking and its reminder status to a Data Table or Google Sheet for reporting.
- **Multi-language beyond Arabic/English:** the system prompt already tells the agent to mirror the customer's language — extend the instruction and service copy to any language your customers use.
