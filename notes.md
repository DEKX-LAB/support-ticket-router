# Build Notes — Support Ticket Router

## 1. Keyword List Per Category

**Billing:** invoice, refund, payment, receipt, charged, deposit, price paid

**Repair:** screen, battery, won't turn on, cracked, charging port, not charging, water damage, repair status, diagnosis, still not fixed

**Sales:** price, quote, buy, purchase, sell, selling, how much, availability, available, new phone, brand new, refurbished, in stock, do you have

## 2. When a Message Matches Two Categories

Categories are checked in a fixed order — Billing, then Repair, then Sales — and the first one whose keywords appear in the message wins; General only applies if none of the three match at all.

This order is a deliberate choice, not an accident. Repair is checked before Sales because a message like "how much to fix my cracked screen" contains both a Sales cue ("how much") and a Repair cue ("screen") — and that customer has a repair problem with a price question attached, not a sales lead. Routing it to Repair first keeps it with the team that can actually resolve it; a customer with billing language present is assumed to need billing attention ahead of anything else, since unresolved billing issues tend to escalate faster than sales enquiries. The order reflects urgency and correctness of resolution, not simply the order the categories happen to be listed in.

## 3. When a Message Matches Nothing

If a message contains none of the Billing, Repair, or Sales keywords, it falls through to a dedicated General category and is routed to the `#general` Slack channel rather than being dropped or forced into one of the three named categories.

This matters for two reasons: nothing is ever silently lost, and forcing an unrelated message (e.g. "What are your opening hours?") into an existing category would corrupt that team's channel and the reporting data drawn from the ticket log. General exists specifically so ambiguous or off-topic requests are still visible and logged, just without a false category label.

## 4. Pricing

This build — the full form-to-Slack-to-email-to-sheet automation — would be priced at $120, with an additional $50 for ongoing retention/maintenance if the client wants continued support after delivery.

## 5. Problems Encountered & How They Were Fixed

### 5.1 — Switch rule using "contains" with a comma-separated string
The first attempt put all keywords for a category into one "contains" condition as a comma-separated string. n8n checked for that entire literal string, not any individual word, so the rule almost never matched. Fixed by switching to "matches regex" with pipe-separated alternation (e.g. `/screen|battery|cracked/`).

### 5.2 — Regex still not matching after the fix
One rule had its operator updated to "matches regex" correctly, but the other two still had "contains" left over from before — so they compared the raw regex string as literal text. Each rule's operator had to be checked and changed individually; fixing one doesn't retroactively fix the others.

### 5.3 — Comparing the wrong field entirely
Even after fixing operator and regex, matching still failed. Root cause: each rule's comparison field ("Value 1") had been left as static placeholder text ("Billing"/"Repair"/"Sales") instead of the expression `{{ $json.search_text }}`. The regex was being tested against the literal word "Repair," which obviously never contains "screen." Corrected to reference `search_text` and routing worked immediately.

### 5.4 — Wrong field mapped in Normalise
An early test showed the `email` field holding the customer's name instead of their email address — a copy-paste error pointing at `$json.Name` instead of `$json.Email`. Caught before it reached Auto Reply, where it would have caused emails to silently fail.

### 5.5 — Keyword gap exposed by real test phrasing
"Do you sell refurbished iPhones?" fell through to General instead of Sales, because the original Sales list only covered "buy"/"price"-style phrasing, not "sell," "selling," "purchase," or "refurbished." A genuine coverage gap, caught because testing used varied real-world phrasing instead of one example per category. Fixed by expanding the Sales pattern.

### 5.6 — Duplicated classification logic (design decision)
Initially, the Switch node's regex rules were the only place category logic lived, meaning the same keyword logic would need to be repeated in extra "tag" nodes after each branch just to label Slack messages. Instead, a single `Determine Category` Code node was inserted before the Switch, classifying once and writing a real `category` field onto the item. The Switch was simplified to a plain equality check. This is a single-source-of-truth design: one place defines what counts as a Billing ticket, and everything downstream just reads that field.

### 5.7 — Slack nodes overwriting the original ticket data
After the Merge node, Auto Reply came out with a blank customer name. Root cause: Merge was wired directly to the four Slack "Send Message" nodes, and a Slack node's output is Slack's own API response (channel, timestamp, bot ID) — not the ticket data that was sent to it. The original fields effectively disappeared the moment they passed through a Slack node.

Fixed using n8n's cross-node reference syntax to reach back to the last node that still held the full ticket record:

```
{{ $('Determine Category').item.json.name }}
{{ $('Determine Category').item.json.email }}
{{ $('Determine Category').item.json.ticket_id }}
{{ $('Determine Category').item.json.category }}
```

**Known limitation:** this pattern assumes exactly one item is flowing through the workflow at a time — true for a single form submission, but it would need a different approach (explicitly carrying data forward through a combining Merge mode) if ever adapted for batch processing.

## 6. Key Design Decisions Summary

- Renamed the generic "Technical" category to "Repair" to match the business's actual core service, while keeping the Slack channel named `#tech-support`
- Fixed category evaluation order (Billing → Repair → Sales → General) to resolve keyword overlaps in favor of the more specific/urgent category
- Centralized classification logic into one Code node rather than duplicating keyword rules across the Switch and four downstream branches
- Used Fallback → Extra Output on the Switch node so unmatched tickets get their own real output (General) rather than being dropped or misfiled
- Set differentiated, honest response-time promises per category (Sales: same day, Billing: 24hrs, Repair: 24–48hrs) rather than one blanket SLA
- Used cross-node data references to recover original ticket data after it was overwritten by the Slack nodes' API responses

## 7. Known Limitations

- Keyword lists are tuned against the test cases run during this build; real customer messages will likely surface phrasing not yet covered
- Ticket IDs use a 4-digit random suffix, carrying a small but real collision risk at higher volumes; a production version would check for existing IDs or use a longer random component
- The cross-node reference pattern used to recover ticket data assumes single-item processing, not batch processing
- Classification is keyword-based rather than true language understanding, so sarcasm, typos, or unusual phrasing can still be misrouted

## 8. Testing Checklist (Definition of Done)

- [ ] Four distinct test tickets (Billing, Repair, Sales, unmatched) each land in their correct Slack channel
- [ ] Every ticket triggers an auto-reply email carrying its own correct ticket ID and category-specific response time
- [ ] Every ticket appears as a complete row in the Tickets Google Sheet with no blank columns
- [ ] No two generated ticket IDs are identical across test runs
