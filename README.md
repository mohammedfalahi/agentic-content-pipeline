# 🤖 Agentic Content Creation & Distribution Pipeline

An end-to-end, AI-driven social media content production system built on **n8n**. From client onboarding to published posts — fully automated, with human-in-the-loop review at every critical stage.

---

## 📌 Overview

This pipeline takes a client from zero to a fully produced, approved, and ready-to-publish 30-day content calendar — without a custom frontend. The operator interface is **Google Forms + Google Sheets + Telegram**. The AI layer is **Google Gemini**. The source of truth is **Supabase**.

```
Google Form → Brand Intelligence → 30-Day Calendar → Content Production → Manager Review → Publish
```

---

## 🗂️ Workflow Files

| File | Phase | Description |
|------|-------|-------------|
| `PHASE_1_-_Client_Onboarding___Brand_Intelligence.json` | Phase 1 | Google Form trigger → website scraping → brand intelligence generation → Supabase + Sheets setup |
| `PHASE_2A_-_Content_Calendar_Parser.json` | Phase 2A | Gemini generates 30-day content calendar → saved to Supabase + Google Sheets → Google Doc sent for approval |
| `PHASE_2B__Telegram_Interactive_Approval_Gateway.json` | Phase 2B | Telegram button handler for Approve / Request Revision → updates Supabase + Sheets |
| `PHASE_2C_-_REVISION_REPLY_CAPTURE.json` | Phase 2C | State-machine revision capture → Gemini parses natural language instructions → updates Supabase + Sheets |
| `PHASE_3A___AUTOMATED_PRODUCTION_ENGINE.json` | Phase 3A | Copy generation + image generation + logo compositing + Drive upload + Sheets writeback per content row |
| `__Phase_3B___Manager_Approval_Sync_Workflow.json` | Phase 3B | Polls Supabase every 12h → detects `Manager_Approved` in Sheets → UPSERTs human edits back to Supabase |
| `Telegram_Master_Router_Workflow.json` | Router | Single Telegram bot trigger → routes callback queries and text messages to correct sub-workflows |
| `Error_Handler.json` | Global | Catches errors from all workflows → sends formatted alert to Telegram |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OPERATOR INTERFACES                       │
│   Google Form (onboarding)  │  Google Sheets (review)       │
│         Telegram Bot (approvals, alerts, commands)          │
└───────────────────┬─────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────┐
│                   n8n ORCHESTRATION LAYER                    │
│  Phase 1 → Phase 2A → Phase 2B/2C → Phase 3A → Phase 3B    │
└──────┬──────────────┬──────────────────┬────────────────────┘
       │              │                  │
┌──────▼──────┐ ┌─────▼──────┐  ┌───────▼────────┐
│  Supabase   │ │   Google   │  │  Google Drive  │
│  (master DB)│ │   Sheets   │  │  (assets)      │
└─────────────┘ └────────────┘  └────────────────┘
       │
┌──────▼──────────────────────────────────────────┐
│             AI / EXTERNAL SERVICES               │
│  Gemini 2.5 Flash (copy)  │  Gemini Image API   │
│  Firecrawl (web scraping) │  Google Docs API    │
└─────────────────────────────────────────────────┘
```

---

## 🔁 Phase-by-Phase Flow

### Phase 1 — Client Onboarding & Brand Intelligence
**Trigger:** New Google Form submission

1. Google Sheets trigger fires on new form response
2. Firecrawl scrapes the client's website (non-blocking — continues on fail)
3. Google Drive fetches the client's uploaded brand guidelines PDF (if provided)
4. Gemini synthesises all three sources into structured brand intelligence: `brand_guidelines`, `target_audience`, `keyword_collections`
5. A new client record is created in Supabase (`Clients` table) and a dedicated Google Sheet + Drive folder are provisioned
6. Phase 2A is called automatically with the client profile

---

### Phase 2A — Content Calendar Generation & Approval Setup
**Trigger:** Called by Phase 1

1. Gemini generates a 30-day content calendar as a strict JSON array (format, topic, image prompt, keywords, CTA, carousel slides)
2. All 30 rows saved to Supabase `Content_Calendar` with `status: Draft`
3. All 30 rows appended to the client's Google Sheet (`Content Calendar` tab)
4. A Google Doc is compiled with the full plan and moved to the client's Drive folder
5. A Telegram message is sent to the admin with a link to the Google Doc and two inline buttons: **✅ Approve Plan** / **✏️ Request Revision**

---

### Phase 2B — Telegram Approval Gateway
**Trigger:** Telegram inline button click (routed from Master Router)

- **Approve path:** Updates all 30 `Content_Calendar` rows to `Plan_Approved` in Supabase → mirrors to Google Sheets → sends confirmation → Phase 3A begins
- **Revision path:** Sets status to `Revision_Requested` in Supabase → creates a session in `pending_revisions` table → prompts admin to describe changes in plain text

> Auth check: Supabase `Authorized_Users` table is queried before any action. Only `Admin` or `Strategist` roles can proceed.

---

### Phase 2C — Revision Reply Capture
**Trigger:** Plain-text Telegram message (routed from Master Router)

1. Checks `pending_revisions` table for an active session for this `telegram_id` — silently exits if none
2. If `/cancel` is typed: deletes the session and confirms cancellation
3. Otherwise: Gemini parses the natural language message into `[{ day: N, comment: "..." }]`
4. Each day's comment is written to `Content_Calendar` in Supabase and the `Comment` column in Google Sheets
5. Confirmation message lists every day updated

---

### Phase 3A — Automated Production Engine
**Trigger:** Weekly cron (runs for all `Plan_Approved` rows in the current week)

For each content row:
1. Fetches client brand profile from Supabase
2. Gemini generates publish-ready copy: `post_copy`, `hashtags`, `hook`, `alt_text`, and for carousels: `slide_1`–`slide_7` (each with `headline`, `subtitle`, `body`)
3. Gemini Image API generates an image from the row's `image_prompt`
4. Client logo is downloaded, resized, and composited onto the image
5. Final image uploaded to the client's Google Drive folder
6. Google Sheets row updated with: post copy, hashtags, image link, Drive asset link
7. Supabase row updated with all generated content, `status: Pending_Manager`

---

### Phase 3B — Manager Approval Sync
**Trigger:** Schedule — every 12 hours

1. Fetches all `Pending_Manager` rows from Supabase
2. For each row: reads the corresponding Google Sheets row and checks the `Status` column
3. If `Status` = `Manager_Approved` (typed manually by the marketer):
   - Builds UPSERT payload from human-edited Sheets data (copy, hashtags, carousel slides)
   - Overwrites the AI draft in Supabase with the approved human version
4. Once all 30 rows for a client are approved: updates client status in Supabase + sends Telegram notification

---

### Telegram Master Router
**Trigger:** Single Telegram bot webhook (all messages)

Routes incoming payloads to the correct sub-workflow:
- `callback_query` present → Phase 2B (approval/revision button)
- Reply-to message present → Phase 2C (revision text reply)

---

### Error Handler
**Trigger:** Any workflow error (set as `errorWorkflow` in Phase 3A and 3B)

Sends a formatted Telegram alert including: workflow name, failed node, error message, timestamp, and execution link.

---

## 🗄️ Database Schema (Supabase)

### `Clients`
| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID PK | Auto-generated |
| `client_name` | Text | |
| `website_url` | Text | |
| `brand_guidelines` | JSONB | `{ tone, messaging, values, slogans }` |
| `target_audience` | JSONB | `{ demographics, segments, pain_points, income_tier, competitor_context }` |
| `keyword_collections` | JSONB | SEO/AEO/AIEO keyword groups |
| `sheets_id` | Text | Google Sheets document ID |
| `drive_folder_id` | Text | Google Drive folder ID |
| `logo_url` | Text | Public URL to client logo |
| `status` | Text | Current pipeline stage |
| `telegram_id` | BigInt | Admin's Telegram ID for notifications |

### `Content_Calendar`
| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID PK | |
| `client_id` | UUID FK | → `Clients.id` |
| `day_number` | Int | 1–30 |
| `format` | Text | `post`, `carousel`, `article`, `infographic`, `video` |
| `core_topic` | Text | |
| `image_prompt` | Text | |
| `keywords` | Text | Comma-separated |
| `cta` | Text | |
| `posting_date` | Date | UAE timezone (UTC+4) |
| `carousel_content` | JSONB | `{ slide_1..slide_7: { headline, subtitle, body } }` — null if not carousel |
| `post_copy` | Text | AI-generated or human-edited caption |
| `hashtags` | Text | |
| `hook` | Text | First-line scroll-stopper |
| `alt_text` | Text | Accessibility/SEO image description |
| `image_url` | Text | Google Drive image link |
| `drive_folder_url` | Text | |
| `status` | Text | See status flow below |
| `approved_by` | BigInt | Telegram ID of approver |
| `approved_at` | Timestamp | |
| `updated_at` | Timestamp | |
| `revision_flagged` | Boolean | |
| `revision_count` | Int | |
| `target_platform` | Text | LinkedIn, Instagram, Facebook, Pinterest |
| `strategic_angle` | Text | |
| `keyword_to_use` | Text | Primary keyword for this post |
| `cta_url` | Text | |
| `arabic_required` | Boolean | |

### `Authorized_Users`
| Field | Type | Notes |
|-------|------|-------|
| `telegram_id` | BigInt PK | |
| `user_name` | Text | |
| `role` | Text | `Admin`, `Strategist`, `Digital_Marketer` |
| `is_active` | Boolean | |

### `pending_revisions`
| Field | Type | Notes |
|-------|------|-------|
| `telegram_id` | BigInt | Session owner |
| `client_id` | UUID | Which client is being revised |

---

## 📊 Content Status Flow

```
Draft
  └─► Plan_Approved        (admin taps Approve in Telegram)
        └─► Pending_Manager  (Phase 3A finishes producing assets)
              └─► Manager_Approved  (marketer types this in Google Sheets)
                    └─► Published   (Phase 4 publishes to social platforms)

Draft ──► Revision_Requested  (admin taps Request Revision in Telegram)
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Workflow automation | [n8n](https://n8n.io) (self-hosted) |
| Master database | [Supabase](https://supabase.com) (PostgreSQL) |
| Operator dashboard | Google Sheets |
| Asset storage | Google Drive |
| Client onboarding | Google Forms |
| Command interface | Telegram Bot API |
| AI — copy generation | Google Gemini 2.5 Flash / Gemini 3 Flash Preview |
| AI — image generation | Gemini 2.5 Flash Image (Nano Banana) |
| Web scraping | [Firecrawl](https://firecrawl.dev) (self-hosted, Docker, port 3002) |
| Document generation | Google Docs API |

---

## ⚙️ Setup & Import

### Prerequisites
- n8n instance (self-hosted recommended)
- Supabase project with the schema above created
- Google Cloud project with Sheets, Drive, Docs APIs enabled
- Telegram bot created via [@BotFather](https://t.me/botfather)
- Firecrawl instance running (or use the hosted API)
- Gemini API key

### Credentials to configure in n8n
| Credential name in workflows | Type |
|------------------------------|------|
| `content management api` | Google Gemini (PaLM) API |
| `Supabase account` | Supabase API |
| `Google Sheets account` | Google Sheets OAuth2 |
| `Telegram account` | Telegram Bot API |
| Google Drive account | Google Drive OAuth2 |
| Google Docs account | Google Docs OAuth2 |

### Import order
Import workflows in this order to avoid broken sub-workflow references:

```
1. Error_Handler.json
2. PHASE_2C_-_REVISION_REPLY_CAPTURE.json
3. PHASE_2B__Telegram_Interactive_Approval_Gateway.json
4. PHASE_2A_-_Content_Calendar_Parser.json
5. PHASE_3B___Manager_Approval_Sync_Workflow.json
6. PHASE_3A___AUTOMATED_PRODUCTION_ENGINE.json
7. PHASE_1_-_Client_Onboarding___Brand_Intelligence.json
8. Telegram_Master_Router_Workflow.json
```

### After import
1. Update all credential references in each workflow to your own credentials
2. In Phase 3A and Phase 3B settings, set `errorWorkflow` to the ID of your imported `Error_Handler` workflow
3. In `Telegram_Master_Router_Workflow`, update the workflow IDs for Phase 2B and Phase 2C to match the IDs assigned after import
4. In Phase 1, update the Google Form Sheet ID and the trigger sheet name to match your form
5. Activate workflows in the same order as the import order above — activate `Telegram_Master_Router_Workflow` last

---

## 📁 Repository Structure

```
/
├── README.md
├── PHASE_1_-_Client_Onboarding___Brand_Intelligence.json
├── PHASE_2A_-_Content_Calendar_Parser.json
├── PHASE_2B__Telegram_Interactive_Approval_Gateway.json
├── PHASE_2C_-_REVISION_REPLY_CAPTURE.json
├── PHASE_3A___AUTOMATED_PRODUCTION_ENGINE.json
├── __Phase_3B___Manager_Approval_Sync_Workflow.json
├── Telegram_Master_Router_Workflow.json
└── Error_Handler.json
```

---

## 🔐 Security Notes

- The Telegram Master Router is the **only** workflow with an active Telegram trigger. All other workflows are called via `Execute Workflow` nodes. This prevents webhook conflicts.
- Every approval action in Phase 2B checks `Authorized_Users` in Supabase before proceeding. Unauthorized `telegram_id` values are blocked and logged.
- Firecrawl scraping is non-blocking (`Continue on Fail` enabled). A failed scrape will not stop the onboarding pipeline.
- Never commit credentials, API keys, or Supabase service role keys to this repository. Use n8n's credential store exclusively.

---

## 📄 License

Proprietary — Afsa FZE. All rights reserved.
