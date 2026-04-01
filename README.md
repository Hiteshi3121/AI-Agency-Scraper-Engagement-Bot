<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/198e7e10-f3da-4735-8e1b-69e9f34ed46b" /># AI Agency Scraper + Engagement Bot
### n8n Automation Assignment — Round 1

---

## 📌 Project Overview

A fully automated Instagram prospecting and outreach pipeline built entirely in n8n. The system scrapes AI-agency-relevant leads from Instagram, scores them, sends personalised AI-generated DMs via Groq, handles multi-step follow-ups, detects trigger events, and delivers daily performance reports — all without manual intervention.

---

## 🏗️ Workflows (5 total)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| AI Agency Scraper + Engagement Bot | Daily 9 AM | Scrape Instagram leads → score → store in Google Sheets |
| DM Generator + Sender | Daily 10 AM | Generate personalised DMs via Groq → update sheet |
| Follow-up Sequence | Daily 11 AM | Follow up leads with no reply after 2+ hours |
| Event Detection | Every hour | Detect new_follower / bio_update events → auto-respond |
| Daily Report | Daily 8 PM | Summarise daily activity → email + Google Sheets |

---

## 🛠️ Tools & API Choices — Reasoning

### n8n
The assignment requires n8n specifically. It provides visual workflow building, native Google Sheets integration, HTTP Request nodes for any API, and a Loop/SplitInBatches node perfect for processing leads one at a time with delays.

### Apify (Free Tier) — Instagram Hashtag Scraper
Used for scraping Instagram posts by hashtag keywords (`smallbusiness`, `marketingagency`, `digitalmarketing`, `businessautomation`). Apify's free tier provides $5/month in credits — sufficient for 20–30 result scraping runs needed for this assignment. The Instagram Hashtag Scraper is the most reliable free-tier actor for extracting `ownerUsername`, `caption` (used as bio proxy), and `profile_url` from public posts.

**Why not the official Instagram API?** Instagram's Graph API requires a business account and app review — not feasible for a 2-day assignment.

### Groq API (Free Tier) — llama-3.3-70b-versatile
Used for all AI-generated DM and follow-up content. Groq's free tier is fast (sub-second inference), generous, and requires no credit card. The `llama-3.3-70b-versatile` model produces high-quality, contextually relevant outreach messages when given bio + platform context. Every DM references the lead's specific business — never generic.

### Google Sheets API (Service Account)
Acts as the central lead database. Chosen over a traditional database because it's free, human-readable, shareable, and natively supported in n8n. The sheet has a single source of truth for all lead states: `dm_sent`, `dm_text`, `reply_received`, `followup_dm_text`, `lead_score`, `follow_status`.

### Gmail (OAuth2)
Used for daily report delivery. Native n8n node, zero additional setup beyond OAuth consent.

---

## ⚙️ Setup Instructions

### Prerequisites
- n8n Cloud account (free trial) or local n8n instance
- Apify account (free tier) — [apify.com](https://apify.com)
- Groq API key (free) — [console.groq.com](https://console.groq.com)
- Google Cloud project with Sheets API + Drive API enabled
- Google Service Account JSON key file
- Gmail account with OAuth2 configured in n8n
- Dedicated test Instagram account (never use personal accounts)

---

### Step 1 — Google Sheets Setup

1. Create a new Google Sheet named **AI Agency Lead Tracker**
2. In **Sheet 1** (rename to `AI Agency Lead Tracker`), add these headers in Row 1:

```
username | platform | bio | follower_count | following | profile_url | status | dm_sent | dm_sent_date | reply_received | followup_dm_text | followup_dm_sent_date | lead_score | follow_status | follow_timestamp | dm_text
```

3. Create a second tab named **Event Log** with headers:
```
event_date | username | platform | event_type | event_description | profile_url
```

4. Create a third tab named **Daily Report** with headers:
```
date | follows_sent | dms_sent | replies_received | conversion_rate
```

5. Share the sheet with your Google Service Account email (e.g. `n8n-bot@your-project.iam.gserviceaccount.com`) with **Editor** access.

---

### Step 2 — n8n Credentials Setup

**Google Sheets (Service Account):**
1. In n8n → Settings → Credentials → Add Credential → Google Sheets API
2. Select **Service Account** authentication
3. Open your downloaded JSON key in a text editor
4. Copy `client_email` → paste into Service Account Email field
5. Copy `private_key` → paste into Private Key field
6. Save → test connection ✅

**Groq API (Bearer Token):**
1. Add Credential → HTTP Bearer Auth
2. Token: your Groq API key (`gsk_...`)
3. Save ✅

**Gmail (OAuth2):**
1. Add Credential → Gmail OAuth2
2. Follow OAuth consent flow
3. Save ✅

**Apify:**
1. No n8n credential needed — token is passed directly in HTTP Request URLs
2. Copy your Apify API token from console.apify.com → Settings → Integrations

---

### Step 3 — Import Workflows

1. In n8n → go to Workflows
2. Click **Import** (top right menu)
3. Import each JSON file in this order:
   - `AI_Agency_Scraper___Engagement_Bot.json`
   - `DM_Generator___Sender.json`
   - `Follow-up_Sequence.json`
   - `Event_Detection.json`
   - `Daily_Report.json`

---

### Step 4 — Update Credentials in Each Workflow

After importing, open each workflow and update:
- All **Google Sheets** nodes → select your `Google Sheets account` credential
- All **HTTP Request (Groq)** nodes → select your `Bearer Auth account` credential
- **Gmail** node in Daily Report → select your Gmail credential
- All **HTTP Request (Apify)** nodes → replace `YOUR_APIFY_TOKEN` with your actual token

---

### Step 5 — Update Google Sheet ID

In every Google Sheets node across all workflows, the Document ID is currently set to:
```
13oZSG5IcDr1jLwa89avkSASXOyeXy1LNJva3EB_4Shw
```
Replace this with your own Sheet ID (found in your Google Sheet URL between `/d/` and `/edit`).

---

### Step 6 — Update Report Email

In the **Daily Report** workflow → `Send a message` node → update the `sendTo` field to your own email address.

---

### Step 7 — Run Workflows

Run in this order for a complete demo:

```
1. AI Agency Scraper    → populates leads into Google Sheet
2. DM Generator         → generates + logs personalised DMs
3. Event Detection      → detects events, generates response DMs
4. Follow-up Sequence   → sends follow-ups to non-responders
5. Daily Report         → emails performance summary
```

Click **Execute workflow** manually to run each one immediately, or activate the schedule triggers for automatic daily execution.

---

## 🧠 Lead Scoring System

Each scraped lead is automatically scored 1–10 before being saved:

| Signal | Score |
|--------|-------|
| Bio contains: `founder`, `ceo`, `agency`, `automation`, `ai`, `saas` | +3 each |
| Bio contains: `marketing`, `business`, `ecommerce`, `consultant`, `entrepreneur`, `startup` | +2 each |
| Bio contains: `digital`, `growth` | +1 each |
| Bio length > 50 chars | +2 |
| Bio length > 100 chars | +1 |
| Valid profile URL | +1 |

**Leads scoring below 4 are automatically filtered out** — quality over quantity.

---

## 📡 Event Detection Logic

The Event Detection workflow runs hourly and handles 2 trigger types:

| Event | Condition | Action |
|-------|-----------|--------|
| `new_follower` | `follow_status = followed` AND `dm_sent = no` | Generate first DM via Groq → update sheet |
| `bio_update` | Bio contains AI keywords AND `dm_sent = yes` AND no followup yet | Generate re-engagement DM via Groq → update sheet |

> **Note on `new_post` event:** True new-post detection requires real-time platform webhooks or a paid Apify Profile Monitor actor — not available on the free tier. This would be the first addition with a paid budget (see improvements section below).

---

## 🐦 Twitter / X — Known Limitation

Twitter scraping via Apify free-tier actors returned `noResults` during testing, likely due to X's aggressive anti-scraping measures on free endpoints. With a paid Apify subscription or the official X API Basic tier, Twitter outreach at 100+ DMs/day is fully achievable using the same n8n workflow structure — only the scraper actor and data processing code node would need updating.

---

## ⏱️ Rate Limiting & Anti-Spam Design

- **Randomised delays** between DMs: `Math.floor(Math.random() * 60) + 30` seconds (30–90s random)
- **One lead per loop iteration** — never batch sends
- **Dedicated test accounts only** — never personal Instagram profiles
- Daily DM limit respected by design (workflow runs once/day, processes available leads sequentially)

---

## 📁 Repository Structure

```
ai-agency-automation-bot/
│
├── workflows/
│   ├── AI_Agency_Scraper___Engagement_Bot.json
│   ├── DM_Generator___Sender.json
│   ├── Follow-up_Sequence.json
│   ├── Event_Detection.json
│   └── Daily_Report.json
│
├── sample_output/
│   ├── SCREENSHOTS_OF_WORKFLOWS_AND_GOOGLE_SHEET.pdf
│
├── screen_recording.mp4 (https://drive.google.com/drive/folders/107MgqGChupUKvf2Nix-Vb6ZsotF1g7GR?usp=sharing) //unable to upload the file due to size. therefore sharing the recording link here 
└── README.md
```

---

## 🔧 What I Would Improve With More Time or a Paid Budget

With more time and a paid budget, the most impactful improvements would be:

**Real follow-back detection** — Currently `follow_status` is set manually as a simulation. With a paid Apify Instagram Profile Monitor actor or a webhooks-capable Instagram Business API integration, follow-back events would trigger automatically in real time rather than being simulated via the hourly Event Detection poll.

**True geo-filtering** — The free Instagram Hashtag Scraper has no location filter. A paid Apify Instagram Location Scraper would allow scraping by city or coordinates, targeting local businesses precisely. Google Maps Scraper (at $0.004/place) could also enrich leads with verified business addresses before outreach.

**Twitter / X platform** — Free-tier Twitter scraping proved unreliable during testing. The X API Basic tier ($100/month) or a paid Apify actor would enable the full 100 DMs/day on X as required by the assignment spec, using the same n8n workflow structure already built.

**Reply detection** — Currently `reply_received` is updated manually. With Instagram's Messaging API (requires app review), incoming DM replies could automatically update the sheet and halt follow-up sequences for that lead — closing the full automation loop.

**Lead enrichment** — Adding a Clearbit or Apollo.io lookup step after scraping would enrich leads with company size, revenue, and verified email — dramatically improving DM personalisation and conversion rates beyond bio-only context.

*(199 words)*

---

## Demo Screen recording link : 

https://drive.google.com/drive/folders/107MgqGChupUKvf2Nix-Vb6ZsotF1g7GR?usp=sharing

## 📊 Sample Output

**Google Sheet — Lead Tracker:**
```
username        | platform  | lead_score | dm_sent | dm_sent_date | dm_text (truncated)
----------------|-----------|------------|---------|--------------|----------------------
ecomemeka       | instagram | 8          | yes     | 2026-04-01   | Hi Ecomemeka, I came across...
dindiaagency    | instagram | 7          | yes     | 2026-04-01   | Hey Dindia, love what you're...
planred.pymes   | instagram | 6          | yes     | 2026-04-01   | Hi Planred, noticed your focus...
```

**Daily Report Email (sent at 8 PM):**
```
Date:               2026-04-01
Follows Sent:       17
DMs Sent:           37
Replies Received:   0
Conversion Rate:    0%
```

**Event Log:**
```
event_date  | username     | event_type   | event_description
------------|--------------|--------------|------------------------------------------
2026-04-01  | ecomemeka    | new_follower | ecomemeka is now followed — ready for first DM
2026-04-01  | dindiaagency | bio_update   | dindiaagency bio has AI-relevant keywords — re-engage
```

---

*Built with n8n + Apify + Groq API + Google Sheets*
*AI Automation Intern — Assignment 01 | April 2026*
