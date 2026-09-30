# Support Ticket Router

An n8n workflow that automatically categorizes, routes, acknowledges, and logs incoming customer support requests for a device repair business.

## The Problem

Support requests came in through a website form and a shared email inbox watched loosely by four staff. This caused:

- Billing questions sitting unanswered for days (technical staff assumed accounts would handle them)
- Sales enquiries — the highest-value requests — treated like ordinary complaints
- No visibility into request volume or type, so staffing decisions were guesses
- No acknowledgement to customers, causing duplicate submissions across channels

## What This Workflow Does

1. **Captures** every request via a public web form
2. **Classifies** it as Billing, Repair, Sales, or General based on message content
3. **Routes** it to the matching Slack channel for the right team
4. **Replies** to the customer immediately with a ticket reference and an honest response-time estimate
5. **Logs** every ticket to a Google Sheet for reporting

## Architecture

```
Support Form (Form Trigger)
      │
      ▼
Normalise (Set) ── cleans fields, builds search_text
      │
      ▼
Ticket ID (Code) ── generates TCK-YYYYMMDD-#### reference
      │
      ▼
Determine Category (Code) ── single source of truth for classification
      │
      ▼
Category (Switch) ── routes on the category field
      │
   ┌──┼──┬──────┐
   ▼  ▼  ▼      ▼
Billing Repair Sales General   (four Slack nodes)
   │  │  │      │
   └──┴──┴──────┘
      ▼
All Tickets (Merge, Append)
      │
      ▼
Auto Reply (Gmail)
      │
      ▼
Log Ticket (Google Sheets)
```

## Category Keywords

**Billing:** invoice, refund, payment, receipt, charged, deposit, price paid

**Repair:** screen, battery, won't turn on, cracked, charging port, not charging, water damage, repair status, diagnosis, still not fixed

**Sales:** price, quote, buy, purchase, sell, selling, how much, availability, available, new phone, brand new, refurbished, in stock, do you have

Anything matching none of the above routes to **General**, so nothing is silently dropped.

Categories are evaluated in the order **Billing → Repair → Sales → General**. This resolves overlapping messages (e.g. "how much to fix my cracked screen") in favor of the more specific/urgent category rather than the first keyword typed.

## Setup

1. Import `workflow/support-ticket-router.json` into your n8n instance
2. Set the workflow timezone to your local timezone (this build uses `Africa/Lagos`)
3. Create your own credentials in n8n and connect them to the nodes:
   - Slack OAuth2 (four Slack "Send Message" nodes)
   - Gmail OAuth2 (Auto Reply node)
   - Google Sheets OAuth2 (Log Ticket node)
4. Update the following placeholders with your own values:
   - `YOUR_SLACK_CHANNEL_ID` in each Slack node — select your channel from the node's UI
   - `YOUR_GOOGLE_SHEET_ID` in the Log Ticket node — select your sheet from the node's UI
5. Create a Google Sheet with these exact column headers: `Ticket ID, Received At, Name, Email, Subject, Message, Category, Routed To, Status`
6. Create four Slack channels matching your routing: `#billing`, `#tech-support`, `#sales`, `#general`

> Credential IDs, channel IDs, and the spreadsheet ID have been redacted from the exported JSON in this repo. n8n will prompt you to reselect them from your own account when you import the workflow.

## Design Decisions & Build Notes

See [`notes.md`](./notes.md) for the full write-up: every bug encountered during the build, why each fix works, and the reasoning behind key decisions (category ordering, single-source-of-truth classification, response-time commitments, and known limitations).

## Known Limitations

- Keyword lists are tuned against the test cases run during this build; real-world phrasing will likely surface gaps needing ongoing refinement
- Ticket IDs use a 4-digit random suffix — a small collision risk exists at high volume
- Classification is keyword-based, not true language understanding, so atypical phrasing can be misrouted

---

Built by [Dekx](https://github.com/DEKX-LAB) — no-code automation for African SME markets.
