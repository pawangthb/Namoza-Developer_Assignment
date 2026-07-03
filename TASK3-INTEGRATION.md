# Task 03 — Integration Design

## Architecture

Form → a small serverless function (Node on AWS Lambda or a Vercel function) → HubSpot → Karix → Google Ads. Not a native HubSpot form embed, and not Zapier/Make.

Native embed is out — it submits straight into HubSpot with no room to check for an existing contact by phone first, and HubSpot's own dedup runs on email, which this form doesn't collect. Zapier/Make would work for a v1 but adds its own polling delay, and with a 2-minute WhatsApp SLA I don't want a black-box layer sitting in that timing path. Direct API call gives full control over sequencing and retries.

Order: the function first calls HubSpot's Contacts API and searches by phone (a custom indexed property, since phone isn't a native unique key in HubSpot) — updates if a match exists, creates if not. Once that write succeeds, it calls Karix to send the WhatsApp confirmation. In parallel, it fires the Google Ads conversion server-side via the Conversion API rather than relying only on the browser-side dataLayer, since the user's likely already left the page by then and a browser-side hit alone isn't reliable.

## Biggest failure point

Phone-based deduplication. HubSpot dedupes on email by default, and this form never collects one, so without explicit handling every submission creates a new contact instead of updating the existing patient record.

Fallback: the function always searches by phone before writing. If two patients submit with the same phone number but different names — a real scenario, shared family phones aren't rare — I'd update the contact with the latest name and enquiry details, but log the older name into a separate "alternate names" property instead of just overwriting it. Keeps one canonical contact per phone number for the sales team to work from, without silently losing the earlier name.

## SLA risk

What could blow the 2-minute WhatsApp window: Karix rate-limiting, Meta throttling the template message, a HubSpot latency spike delaying the WhatsApp trigger, or the function cold-starting during a campaign traffic spike.

I'd timestamp submit, HubSpot-write-confirmed, and WhatsApp-send-confirmed separately, and alert on Slack if the submit-to-WhatsApp gap crosses 90 seconds — a buffer before the SLA actually breaks. A daily p50/p95 view catches latency creeping up before it becomes a real miss.
