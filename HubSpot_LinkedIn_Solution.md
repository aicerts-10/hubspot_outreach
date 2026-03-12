# HubSpot + LinkedIn Outreach Agent — Solution Document

---
### What's Being Asked

> With Team Sales Navigator integrated into HubSpot, can each sales rep pick LinkedIn contacts they want to reach out to, have an AI agent (like the existing Lead Gen Agent) automatically create personalized outreach — draft emails and LinkedIn connection notes — and then let the rep review and approve before sending?

**This implicitly also asks for two things that are not possible:**
- **Auto-pull leads from Sales Navigator into the agent** — Sales Navigator has no API; contacts must enter HubSpot via native sync, CSV export, or manual entry.
- **One-click send for LinkedIn connection requests** — LinkedIn has no API for sending personal messages from a CRM; the rep must copy-paste the note manually (~10 sec).


## TL;DR — Solution Summary

We build a custom Agent API that plugs into HubSpot via webhooks. When a rep's LinkedIn contacts land in HubSpot — via Sales Nav native sync (auto), CSV export from Sales Nav, or manual URL entry — our agent automatically enriches the profile (via Apify scraper), generates personalized outreach (LinkedIn connection note + email) using AI, and writes the drafts back to the HubSpot contact record. The rep reviews, edits if needed, and sends — emails go directly from HubSpot (fully automated); LinkedIn connection notes are copied and pasted manually by the rep (~10 sec) since LinkedIn provides no official API for sending personal messages from a CRM. Each rep gets customized outreach based on their tone, service focus, and signature stored in HubSpot.

**Two hard limitations:** (1) No Sales Navigator API exists — contacts enter HubSpot via the native Sales Nav sync (auto, but limited), CSV export from Sales Nav, or manual URL entry. (2) No LinkedIn messaging API for personal accounts — LinkedIn connection notes require a quick manual copy-paste send by the rep.

---

## 1. Solution


### Our Solution

A **custom Agent API** (FastAPI) that plugs into HubSpot via webhooks. HubSpot handles the CRM, task management, and email sending. Our agent handles LinkedIn enrichment and AI outreach generation.

### ✅ What CAN Be Achieved

| Capability | How |
|-----------|-----|
| **Rep saves a LinkedIn contact → it lands in HubSpot** | HubSpot-Sales Nav native sync (basic fields) **OR** rep manually adds LinkedIn URL to a HubSpot contact |
| **Full LinkedIn profile enrichment** (skills, experience, headline, education) | Our Agent API calls **LinkedIn Profile Scraper**  |
| **AI-generated personalized outreach** (connection note + email) | Our Agent API uses **AI** with custom prompts per prospect |
| **Drafts stored on the contact record in HubSpot** | Agent writes back to HubSpot custom properties via API |
| **Rep reviews, edits, approves before sending (HITL)** | HubSpot Tasks + custom properties — rep sees drafts, edits them, then sends |
| **Email sent from rep's own mailbox with tracking** | HubSpot native email sending (tracked, logged on timeline) |
| **Per-rep customization** (tone, service focus, signature) | Rep preferences stored in HubSpot owner properties → passed to agent in webhook (see [Per-Rep Customization](#per-rep-customization) below) |
| **Batch processing** — multiple contacts at once | Agent API accepts batch payloads; processes asynchronously |

### ❌ What CANNOT Be Achieved (and Why)

| Capability | Why Not |
|-----------|---------|
| **Auto-send LinkedIn connection requests** | LinkedIn's official APIs for personal member accounts **do not support** sending 1:1 connection requests or DMs from a CRM. While LinkedIn has expanded APIs for Message Ads, Page messaging, and some enterprise Sales Nav features — none of these cover personal member-to-member outreach. Third-party tools that offer this use non-compliant browser automation. Our HITL approach (rep copies + pastes in ~10 sec) is the safe path. See [Issue #2](#-issue-2-sending-linkedin-messages-programmatically--the-full-picture) for full details. |
| **Pull lead lists directly from Sales Navigator via API** | LinkedIn **does not expose a Sales Navigator API**. You cannot programmatically fetch a rep's saved leads or search results from Sales Nav. The only bridge is the limited native HubSpot-Sales Nav sync (see Issue #1 below). |
| **Auto-send InMail** | Same as connection requests — no API exists. InMail can only be sent through LinkedIn/Sales Nav UI. |

---

## 2. Proposed Architecture

```
 REP'S WORKFLOW
 ═══════════════

 ┌─────────────────────┐
 │  Rep finds contact   │  (Sales Nav / LinkedIn / Manual)
 │  Adds LinkedIn URL   │
 │  to HubSpot contact  │
 └─────────┬───────────┘
           │
           ▼
 ┌─────────────────────┐
 │   HUBSPOT CRM        │
 │                      │
 │  Workflow triggers:  │
 │  "linkedin_url was   │
 │   just populated"    │
 │                      │
 │  Sends webhook ──────────────────────┐
 └─────────────────────┘                │
                                        ▼
                          ┌──────────────────────────┐
                          │   OUR CUSTOM AGENT API    │
                          │   (FastAPI on Cloud Run)  │
                          │                           │
                          │  1. Receive payload       │
                          │  2. Apify: scrape profile │
                          │
                          │  3. GPT: generate outreach│
                          │  4. HubSpot API: write    │
                          │     drafts back to contact│
                          │  5. Create review task    │
                          └──────────┬───────────────┘
                                     │
                                     ▼
 ┌──────────────────────────────────────────────────┐
 │   HUBSPOT CRM — Contact Record Updated           │
 │                                                   │
 │  Custom Properties:                               │
 │   • ai_linkedin_note  = "Hi [Name], I'm a..."   │
 │   • ai_email_subject  = "Quick question about.." │
 │   • ai_email_body     = "Hi [Name],\n\nI noticed│
 │   • ai_outreach_status = "pending_review"        │
 │                                                   │
 │  Task Created:                                    │
 │   "Review AI outreach for [Contact Name]"        │
 │   Assigned to: [Rep who owns the contact]        │
 └──────────────────┬───────────────────────────────┘
                    │
                    ▼
 ┌──────────────────────────────────────────────────┐
 │   REP REVIEWS & SENDS (HITL)                      │
 │                                                   │
 │  📧 Email: Edit draft → Send via HubSpot email   │
 │  🔗 LinkedIn: Copy note → Paste in LinkedIn UI   │
 │                                                   │
 │  ✅ Mark task complete → status = "sent"          │
 └──────────────────────────────────────────────────┘
```

---

## 3. HubSpot Configuration & Webhook Details

### A. Custom Properties to Create in HubSpot

Create these on the **Contact** object:

| Property Internal Name | Type | Description |
|----------------------|------|-------------|
| `linkedin_profile_url` | Single-line text | The LinkedIn profile URL (e.g., `https://www.linkedin.com/in/john-doe/`) |
| `ai_linkedin_note` | Multi-line text | AI-generated LinkedIn connection note |
| `ai_email_subject` | Single-line text | AI-generated email subject line |
| `ai_email_body` | Multi-line text | AI-generated email body |
| `ai_matched_services` | Single-line text | Services matched to this prospect |
| `ai_outreach_status` | Dropdown | Options: `pending_enrichment`, `pending_review`, `approved`, `sent` |

### B. HubSpot Workflow Setup

**Trigger:** Contact property `linkedin_profile_url` is known (just got populated)  
**AND** `ai_outreach_status` is unknown or `pending_enrichment`

**Action:** Send a Webhook (POST) to our Agent API endpoint

### C. Webhook Payload — What HubSpot Sends to Our Agent

HubSpot webhook sends the **contact ID**. Our agent then fetches the contact details via HubSpot API using that ID.

**Webhook POST from HubSpot:**
```json
[
  {
    "objectId": 12345678,
    "propertyName": "linkedin_profile_url",
    "propertyValue": "https://www.linkedin.com/in/john-doe/",
    "changeSource": "CRM_UI",
    "eventId": 100,
    "subscriptionId": 200,
    "portalId": 9876543,
    "appId": 1234,
    "occurredAt": 1741789200000,
    "eventType": "contact.propertyChange",
    "attemptNumber": 0
  }
]
```

**What our agent extracts from this:**
- `objectId` → HubSpot Contact ID (used to read full contact & write back)
- `propertyValue` → LinkedIn Profile URL (used for Apify enrichment)

### D. What Our Agent API Does (Internal Flow)

```
INPUT: HubSpot webhook payload (contact ID + LinkedIn URL)
  │
  ├─► Step 1: Call Apify LinkedIn Profile Scraper
  │            → Returns: fullName, headline, skills, experience, company, etc.
  │
  ├─► Step 2: Map profile to our services
  │            → Returns: ["cloud", "azure", "cybersecurity", ...]
  │
  ├─► Step 3: Call OpenAI GPT with custom prompt
  │            → Returns: linkedin_note, email_subject, email_body
  │
  └─► Step 4: Write back to HubSpot via Contacts API
               → PATCH /crm/v3/objects/contacts/{contactId}
```

### E. Agent API Response — Written Back to HubSpot

Our agent calls HubSpot API to update the contact:

**PATCH** `https://api.hubapi.com/crm/v3/objects/contacts/12345678`

```json
{
  "properties": {
    "ai_linkedin_note": "Hi John, I'm a Learning Advisor at NetCom Learning. Noticed your work in cloud architecture — we've helped Fortune 1000 teams upskill in Azure & AWS. Would love to connect and share some insights. – [Rep Name]",
    "ai_email_subject": "Upskilling your cloud team?",
    "ai_email_body": "Hi John,\n\nI came across your profile and was impressed by your work leading cloud infrastructure at Acme Corp.\n\nMany teams in similar roles are looking to accelerate Azure and AWS certifications — NetCom Learning (a Microsoft Partner of the Year) has helped 80% of Fortune 1000 companies do exactly that.\n\nWould you be open to a quick 15-min chat to see if we could help your team? Here's a relevant upcoming webinar: [link]\n\nBest,\n[Rep Name]\nLearning Advisor, NetCom Learning",
    "ai_matched_services": "azure, aws, cloud",
    "ai_outreach_status": "pending_review"
  }
}
```

Our agent also **creates a HubSpot Task** via:

**POST** `https://api.hubapi.com/crm/v3/objects/tasks`

```json
{
  "properties": {
    "hs_task_subject": "Review AI outreach for John Doe",
    "hs_task_body": "AI has generated a LinkedIn connection note and email for John Doe. Review the drafts on the contact record, edit if needed, then send.",
    "hs_task_status": "NOT_STARTED",
    "hs_task_priority": "HIGH",
    "hubspot_owner_id": "87654321"
  },
  "associations": [
    {
      "to": { "id": "12345678" },
      "types": [{ "associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 204 }]
    }
  ]
}
```

---

## Per-Rep Customization

Each rep gets a **personalized agent experience** without needing separate deployments. The agent reads rep-specific settings from HubSpot and adjusts its AI output accordingly.

| What's Customized | Where It's Stored | How the Agent Uses It |
|---|---|---|
| **Tone & writing style** | HubSpot Owner custom property: `rep_outreach_tone` (e.g., `"formal"`, `"conversational"`, `"consultative"`) | Injected into the GPT prompt: *"Write in a {tone} style"* |
| **Service/product focus** | HubSpot Owner custom property: `rep_service_focus` (e.g., `"azure, cybersecurity"`) | Agent filters the service-mapping step to only match the rep's focus areas |
| **Email signature** | HubSpot Owner custom property: `rep_email_signature` | Appended to the generated email body |
| **Rep's name & title** | HubSpot Owner standard fields (`firstname`, `lastname`, `jobtitle`) | Used in the outreach: *"Hi John, I'm {rep_name}, {rep_title} at..."* |
| **Company pitch / value prop** | Agent config or HubSpot custom property: `rep_value_prop` | Injected into GPT prompt to tailor the pitch per rep's vertical |

### Where Does the Owner ID Come From?

The `owner_id` (which identifies the rep) comes from the **Contact's `hubspot_owner_id` property** — this is a built-in HubSpot field automatically set when a contact is assigned to a rep.

**How our agent gets it:**

After receiving the webhook with the `objectId` (contact ID), the agent makes one API call to fetch the contact's details:

**GET** `https://api.hubapi.com/crm/v3/objects/contacts/12345678?properties=hubspot_owner_id,linkedin_profile_url`

```json
// Response
{
  "id": "12345678",
  "properties": {
    "hubspot_owner_id": "87654321",       // ← This is the rep's Owner ID
    "linkedin_profile_url": "https://www.linkedin.com/in/john-doe/"
  }
}
```

Then the agent fetches the rep's details using that Owner ID:

**GET** `https://api.hubapi.com/crm/v3/owners/87654321`

```json
// Response
{
  "id": "87654321",
  "email": "sarah@company.com",
  "firstName": "Sarah",
  "lastName": "Chen",
  "userId": 11111,
  "teams": [{ "id": "5001", "name": "Sales Team East" }]
  // Custom owner properties (rep_outreach_tone, rep_service_focus, etc.)
  // are also accessible here
}
```

> **Where to find Owner IDs in HubSpot UI:** Go to **Settings → Users & Teams**. Each user listed has an Owner ID. You can also see it in the URL when viewing a user's profile in HubSpot settings, or via the Owners API: `GET /crm/v3/owners`.

**How it flows:**

```
Webhook fires → Agent receives contact_id + owner_id (from contact record)
                          │
                          ├─► Agent calls HubSpot API: GET /owners/{owner_id}
                          │   → Returns: rep name, title, tone, service focus, signature
                          │
                          ├─► Agent builds a rep-specific GPT prompt:
                          │   "You are writing on behalf of {rep_name}, {rep_title}.
                          │    Tone: {tone}. Focus services: {services}..."
                          │
                          └─► GPT output is personalized to that rep's style
```

**Example:** Two reps processing the same contact get different outreach:

| | Rep A (Formal, Cloud focus) | Rep B (Casual, Cybersecurity focus) |
|---|---|---|
| **LinkedIn Note** | "Dear John, I'm Sarah, a Cloud Solutions Advisor at NetCom Learning. Your Azure architecture work caught my attention..." | "Hey John! I'm Mike from NetCom Learning — saw your security background and thought we should connect. We help teams like yours level up on cybersecurity certs 🚀" |
| **Email Subject** | "Azure Certification Paths for Your Team" | "Quick thought on your security team's growth" |

---

## 4. Biggest Issues

### 🔴 Issue #1: LinkedIn Sales Navigator Has No API

**The Problem:**  
LinkedIn does **not** provide any public API for Sales Navigator. You cannot programmatically:
- Fetch a rep's saved leads
- Pull search results from Sales Nav
- Access lead lists or account lists via code

**The HubSpot-Sales Nav native integration** only syncs **basic data** (name, title, company, LinkedIn URL) and it's controlled by LinkedIn — you can't customize what gets synced.

**What This Means for Us:**  
We **cannot** build a "click a button and import my Sales Nav leads" feature. The contacts need to get into HubSpot through one of these workarounds:

| Workaround | How It Works | Effort |
|-----------|-------------|--------|
| **Native Sales Nav sync** | Rep saves a lead in Sales Nav → it auto-syncs to HubSpot with basic fields + LinkedIn URL. Our agent triggers from there. | Low (but limited data, depends on LinkedIn's sync behavior) |
| **Manual entry** | Rep pastes the LinkedIn profile URL into a HubSpot contact's `linkedin_profile_url` field. Workflow triggers our agent. | Low (simple but manual) |
| **CSV export from Sales Nav** | Rep exports leads from Sales Nav as CSV → uploads to HubSpot. Workflow triggers for each new contact. | Medium (Sales Nav allows limited CSV exports) |
| **Browser extension** (future) | A Chrome extension that captures the LinkedIn profile URL from the page the rep is viewing and pushes it to HubSpot via API. | High (custom build) |

**Recommended starting point:** Use **Manual entry** or **Native sync** for now. Add the browser extension later if the workflow proves valuable.

---

### 🔴 Issue #2: Sending LinkedIn Messages Programmatically — The Full Picture

**The Nuanced Reality (2026):**

LinkedIn's API landscape for messaging has evolved, but **not in the way CRM-to-LinkedIn outreach needs**. Here's the accurate breakdown:

| What Exists | What It Actually Does | Useful for Us? |
|---|---|---|
| **LinkedIn Message Ads API** | Lets companies send sponsored InMail-style ads to targeted audiences (paid, goes to "Sponsored" tab) | ❌ No — this is advertising, not personal 1:1 outreach from a rep |
| **LinkedIn Pages Messaging API** | Lets company pages respond to conversations initiated by users | ❌ No — only works for company pages, not personal member accounts |
| **Sales Navigator Enterprise APIs** | Some enterprise-tier Sales Nav plans offer limited API access for CRM sync, notes, and lead management | ⚠️ Partially — may allow reading data, but **still does not allow sending connection requests or DMs on behalf of a personal account** |
| **LinkedIn Marketing Partner Programs** | Vetted partners get expanded API access for ads, content, and analytics | ❌ No — none of these include member-to-member messaging |
| **Third-party "LinkedIn automation" tools** | Services that programmatically send connection requests/DMs by controlling a logged-in browser session (Selenium, Puppeteer, headless browsers) | ⚠️ Technically works, but **uses non-compliant automation** — LinkedIn actively detects and bans accounts doing this |

**The Bottom Line:**  
LinkedIn's official APIs for **individual member accounts** (i.e., a sales rep's personal LinkedIn) **still do not provide a way** for a third-party CRM like HubSpot to programmatically send 1:1 connection requests or DMs "as that rep." The HubSpot-LinkedIn integrations (Marketing, Ads, Sales Navigator widget) are focused on ads, page content, and embedded Sales Nav views — **none offer a "send connection request with note from HubSpot" capability.**

**Our Recommended Approach — HITL (Human-in-the-Loop):**

This is still the **widely recommended, compliance-safe approach** in 2026:

```
┌─────────────────────────────────────────────────────┐
│  HITL CHECKPOINT                                     │
│                                                      │
│  📧 Email outreach:                                  │
│     ✅ Rep clicks "Send" in HubSpot → FULLY AUTO     │
│     (sent from rep's mailbox, tracked, logged)       │
│                                                      │
│  🔗 LinkedIn connection note:                        │
│     ⚠️  Rep copies note from HubSpot contact record  │
│     ⚠️  Rep opens LinkedIn profile (link provided)    │
│     ⚠️  Rep pastes note into "Add a note" → MANUAL   │
│     (~10 seconds per contact)                        │
└─────────────────────────────────────────────────────┘
```

**How We Make the Manual Step Near-Instant:**
- AI-generated note is **ready to copy** directly from the contact record (one click)
- Contact's LinkedIn profile URL is a **clickable link** → opens LinkedIn in one click
- Note is already **<300 characters** (LinkedIn's connection note limit) — no trimming needed
- Rep flow: **Copy → Click link → Connect → Paste → Send** (~10 seconds per contact)

> **Note on third-party automation tools:** Several services now offer programmatic LinkedIn DMs and connection requests by controlling a logged-in browser session. While technically functional, these operate outside LinkedIn's official API program and carry real risk of account restrictions. We intentionally do **not** recommend this path — the HITL approach keeps rep accounts safe while still delivering 90% of the value (AI does the thinking, rep just clicks send).

---

## Quick Reference Summary

| Question | Answer |
|----------|--------|
| Can we build a per-rep agent in HubSpot? | **Yes** — HubSpot as CRM + our custom Agent API for enrichment & AI outreach |
| Can we parse LinkedIn contacts per rep? | **Yes** — via Apify scraper triggered by our agent. Contacts enter HubSpot via Sales Nav sync or manual URL entry |
| Can the agent generate outreach? | **Yes** — AI-generated connection notes + emails, written back to HubSpot |
| Can reps review before sending (HITL)? | **Yes** — drafts on contact record + review task. Rep edits then approves |
| Can emails be sent from HubSpot? | **Yes** — fully native, tracked, from rep's own mailbox |
| Can LinkedIn messages be auto-sent? | **No** — LinkedIn's official APIs do not support personal member-to-member connection requests from a CRM. Expanded APIs (Message Ads, Page messaging, enterprise Sales Nav) don't cover this use case. Rep copies the AI-generated note and sends manually (~10 sec per contact). |
| Can we pull leads from Sales Nav programmatically? | **No** — no Sales Nav API. Use native sync, manual entry, or CSV export instead |
