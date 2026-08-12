# CapitalAssist — HighLevel Build Spec

Companion document to `index.html`. This is what needs to be built **inside HighLevel itself** — the connected LeadConnector MCP can read CRM data (contacts, opportunities, pipelines, custom fields) but cannot create Funnels, native Forms, Workflows, Pipelines, or Custom Fields via API. Everything in this document has to be built by hand in the GHL dashboard (or by a GHL admin), one time.

## 0. Reality check — what's connected today

- `index.html` is a **fully standalone, self-contained front-end**. It has no backend. Form submissions currently render a local "thank you" panel and never leave the browser (see §5 for how to change that).
- The connected HighLevel MCP is authorized against **Acorn Brokers (Pty) Ltd** (location ID `948mmDDPoLgSeFGraJHy`), Pretoria — an existing, unrelated firearms-insurance sub-account. It already has:
  - One custom field: `How often do you normally workout?` (contact, RADIO) — **leave this alone**, it's unrelated to CapitalAssist.
  - One pipeline: `Marketing Pipeline` (New Lead → Hot Lead → New Booking → Visit Attended → Sale → Left a Review) — **leave this alone too**.
- Everything below is **new and additive** — a separate pipeline and separate custom fields, namespaced `ca_*`, so nothing collides with Acorn Brokers' existing setup.

## 1. New Pipeline

**Name:** `CapitalAssist — Funding Applications`

| Stage | Meaning |
|---|---|
| New Application | Form fully submitted, all consents given |
| Pre-Qualified | Passed the Funding Concierge Score hard gates (age, credit standing, R30k floor) |
| Documents Requested | Consultant has reached out, docs outstanding |
| Documents Received | Full document set collected |
| Submitted to Funding Partner | Handed to a funding partner for assessment |
| Partner Reviewing | Awaiting partner decision |
| Outcome — Approved | Funding partner approved |
| Outcome — Declined / Referred Elsewhere | Not approved this route; candidate for nurture |

**Important:** leads who self-disqualify in the Funding Concierge Score (age/credit/amount gates) do **not** enter this pipeline. They only get a tag (see §4) and go into a nurture list — they were never a real application.

## 2. New Custom Fields

All prefixed `ca_` to avoid any collision with existing fields.

| Field | Model | Type | Notes |
|---|---|---|---|
| `ca_intent` | Contact | Dropdown | vehicle_refinance / cash_against_vehicle / balloon_payment / debt_consolidation / commercial_vehicle / property_backed |
| `ca_loan_amount` | Opportunity | Number | Minimum 30000 enforced client-side; re-validate server-side |
| `ca_asset_type` | Contact | Dropdown | Motor vehicle / Motorcycle / Commercial motor vehicle / Property / Spouse-partner-relative's vehicle |
| `ca_province` | Contact | Dropdown | 9 SA provinces |
| `ca_employment_status` | Contact | Dropdown | Employed full-time / Self-employed / Contract or freelance / Not currently employed |
| `ca_income_band` | Contact | Dropdown | Under R8,000 / R8,000–R15,000 / R15,000–R30,000 / R30,000+ |
| `ca_concierge_score_tier` | Contact | Dropdown | excellent / possible / not_eligible |
| `ca_lead_source_detail` | Contact | Text | Free-text "Other" value when lead_source = Other |
| `ca_doc_method` | Opportunity | Dropdown | upload / email |
| `ca_consent_credit_enquiry`, `ca_consent_popia`, `ca_consent_credit_check`, `ca_consent_terms` | Contact | Checkbox | All four required at final step; capture timestamp + IP alongside (see §6) |

Standard GHL fields (`first_name`, `last_name`, `email`, `phone`, `city`) are reused as-is — no custom field needed for those.

## 3. Multi-Step Form — Field Mapping

`index.html`'s Application section (`#application`) is a 6-step custom form. If you embed a **native GHL Form** in place of it, mirror this exactly:

| Step | Fields | HL field type |
|---|---|---|
| 1. Your Details | first_name*, last_name*, email*, phone* | Text / Text / Email / Phone (standard) |
| 2. Location | city*, ca_province* | Text / Dropdown |
| 3. Confirm Your Funding | ca_intent*, ca_loan_amount* (min 30000), ca_asset_type* | Dropdown / Number / Dropdown |
| 4. How You Heard About Us | lead_source* (Google/Facebook/Magazine/Radio/Referred by a friend/Other), ca_lead_source_detail (conditional), hidden utm_source/medium/campaign/content/term, gclid, fbclid | Dropdown / Text / Hidden |
| 5. Documents | ca_doc_method*, then 8 file uploads: ID, driver's licence, proof of address, payslips, bank statements, settlement letter (optional), vehicle reg (optional), ITC report (optional) | Radio / File upload |
| 6. Consent | ca_consent_credit_enquiry*, ca_consent_popia*, ca_consent_credit_check*, ca_consent_terms* | Checkbox (all required) |

**Known GHL limitation:** native file-upload fields are single-file with a size cap and no drag-drop multi-file UX like the one built into `index.html`. Two options:
- (a) Accept the native field's limits and use it directly (simplest, fully "native").
- (b) Keep the custom drag-and-drop UI from `index.html` and forward files to GHL via a webhook + document-storage step in a workflow (more work, better UX).

The Funding Concierge Score quiz (`#concierge-score`) is **not** a GHL form — it's a client-side pre-qualifier and should stay as custom code regardless of which form option you pick, since GHL has no interactive branching quiz element.

## 4. Automation / Workflow Logic (build in GHL Workflow Builder)

**Workflow A — Application Submitted**
1. Trigger: Form submitted (Step 6 complete, all consents checked)
2. Add tag `capitalassist-application-submitted`
3. Create/Update Opportunity in `CapitalAssist — Funding Applications` at stage `New Application`
4. Internal notification (SMS/email) to the consultant on duty
5. Applicant confirmation SMS/email: "We've received your application — a consultant will call you shortly."

**Workflow B — Concierge Score: Not Eligible Yet**
1. Trigger: lite-capture form submitted (name/email/phone only, from the quiz's disqualification panel)
2. Add tag `capitalassist-not-yet-eligible` + a note recording the disqualifying reason (age / credit standing / amount below R30k)
3. Add to a nurture list — **do not** create an Opportunity in the funding pipeline
4. Send the same honest "we'll be in touch when circumstances change" message as on-page, so online/offline experience matches

## 5. Bridging `index.html` to GHL Today (before native embedding)

`index.html` already has a wiring point ready to go: a single constant near the top of the `<script>` block —

```js
var GHL_WEBHOOK_URL = ''; // set this to your GHL "Inbound Webhook" workflow trigger URL
```

Steps to activate it:
1. In GHL, create a Workflow with an **Inbound Webhook** trigger (Workflow Builder → Add Trigger → Inbound Webhook). Copy the generated URL.
2. Paste that URL into `GHL_WEBHOOK_URL` in `index.html`.
3. On final submit, the page POSTs a JSON payload (see the exact shape in the script — mirrors the field table in §3) to that URL, then still shows the local confirmation panel.
4. In that same Workflow, add the actions from **Workflow A** above (create contact/opportunity, tag, notify, confirm).

This gets you a genuinely working connection without waiting on a fully native Form — the workflow's webhook trigger does the work Workflow A describes, driven by this page's own submit event. It's not more work later either: once you're ready for a fully native GHL Form, swap the custom form markup out at the `<!-- GHL-EMBED -->` comment in `index.html` and this webhook path becomes unnecessary.

## 6. Compliance data to capture alongside consent

For the four consent checkboxes in Step 6, capture and store **timestamp + IP address** at submission time (both the AutoFin Assist and RefiMyCar reference materials treat this as mandatory for the audit trail). If using the webhook bridge in §5, add `timestamp: new Date().toISOString()` to the payload (the client already has this) — IP address should be captured server-side (GHL's webhook trigger logs the request IP automatically; confirm it's accessible in the workflow's trigger data).

## 7. Tracking

- GHL's native page-view / form-submit tracking works automatically once this page is served from a GHL funnel.
- Insert Meta Pixel + Google Ads conversion tags at the `<!-- GHL-EMBED: insert Meta Pixel / Google Ads conversion / GHL tracking scripts here -->` comment right before `</body>`.
- Recommended conversion events to fire: page view → VSL played → Concierge Score started → Concierge Score completed (tier) → Application started → Application submitted. That's a 5-point funnel for CRO analysis, matching the sections in `index.html`.

## 8. Setup checklist

- [ ] Create `CapitalAssist — Funding Applications` pipeline (§1) — do not touch `Marketing Pipeline`
- [ ] Create the `ca_*` custom fields (§2) — do not touch the workout field
- [ ] Decide: native GHL Form (§3) vs. webhook bridge keeping the custom UI (§5) — or both, phased
- [ ] Build Workflow A (application submitted) and Workflow B (not-yet-eligible nurture) (§4)
- [ ] If using the webhook bridge, set `GHL_WEBHOOK_URL` in `index.html` (§5)
- [ ] Confirm IP capture on consent submission (§6)
- [ ] Add Meta Pixel / Google Ads tags (§7)
- [ ] Test the full funnel end-to-end, including the R30k and age/credit disqualification branches, before sending paid traffic
