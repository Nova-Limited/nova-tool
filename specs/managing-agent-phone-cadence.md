# Managing-Agent Track + Phone Cadence Spec

**For Tuesday 26 May 2026 Apps Script work on iMac.** Captures decisions made Mon 25 May 2026. Save to `~/nova-tool/specs/managing-agent-phone-cadence.md`.

---

## 1. Decisions locked in

| Decision | Choice |
|---|---|
| Managing-Agent unsubscribe model | **Both** - Email Finder.Unsubscribed checkbox AND per-contact unsubscribe tokens |
| Track Override strictness | **Lenient** - Company.Track=Managing-Agent auto-includes all linked contacts |
| Phone calls in sequence | **Day 3 + Day 10** added to all tracks (Standard, Multi-Site, Managing-Agent) |
| Call framing | Direct MD lead - no "competition" fiction |

---

## 2. Airtable schema changes

### 2.1 Email Finder table (`tblDOx6ry8Od0GW4R`) - add 5 fields

| Field name | Type | Notes |
|---|---|---|
| Stage | singleSelect | Same options as Companies.Stage plus new "Not Interested - Closed". Only populated for Managing-Agent contacts. |
| Last Email Date | date | Only populated for Managing-Agent contacts. |
| Unsubscribed | checkbox | Per-contact unsubscribe checkbox (Option A from earlier). |
| Unsubscribe Token | singleLineText | UUID generated when first email drafted. Powers per-contact unsubscribe URLs (Option B). |
| Last Call Date | date | When the most recent call attempt was logged. Used by reminder logic. |

### 2.2 Companies table (`tblH0ph6uPuV72Ww9`) - add 1 field, modify 1

**Add:**
| Field name | Type | Notes |
|---|---|---|
| Last Call Date | date | Standard/Multi-Site call tracking. Per-Company because those tracks fire per-Company. |

**Modify Stage singleSelect:** Add option **"Not Interested - Closed"** between "Lost" and "No Response". Colour: red (distinct from amber Lost).

### 2.3 Same Stage modification on Email Finder

When Stage field added (per 2.1), include "Not Interested - Closed" option in the choices.

---

## 3. Cadence

### 3.1 The new sequence (all three tracks)

| Day | Touch | Channel | Stage written |
|---|---|---|---|
| 0 | Email 1 (door opener) | Email | "Email 1 Sent" |
| 3 | **Call 1** (did it land?) | Phone | "Call 1 Attempted" |
| 7 | Email 2 (figures) | Email | "Email 2 Sent" |
| 10 | **Call 2** (discuss figures) | Phone | "Call 2 Attempted" |
| 14 | Email 3 (risk) | Email | "Email 3 Sent" |
| 21 | Email 4 (close + re-entry) | Email | "Email 4 Sent" |

### 3.2 New Stage options needed

Add to Companies.Stage AND Email Finder.Stage choices:
- "Call 1 Attempted"
- "Call 2 Attempted"
- "Not Interested - Closed"

### 3.3 Cadence rules

- If a call doesn't happen on Day 3 (you were busy, no answer, etc), Email 2 still fires on Day 7
- Calls are reminders only - the script doesn't dial, it just adds them to your daily call list
- "Call 1 Attempted" doesn't mean answered - means you tried. Outcome lives in Contact Log.

---

## 4. The daily call list

### 4.1 Mechanics

Apps Script runs daily 9-10am UK (existing trigger). Two things happen in order:

1. **Send drafts** (existing): builds email drafts for any prospect at the right stage and gap.
2. **Send call reminder email** (new): emails jamie@nova-limited.com with today's call list.

### 4.2 Call reminder email shape

```
Subject: Nova call list - Tue 26 May 2026 (3 calls)

You have 3 calls due today:

CALL 1 (Day 3 after Email 1):
- Darren House MD, Grant & Stone Limited
- Phone: 01296 393939
- Email 1 sent: Tue 19 May - "Multi-site portfolio approach"
- Email 1 body: [first 500 chars]
- Suggested opener: "Did my Tuesday email about your 23,000 sqm portfolio reach you?"
- Log outcome in Contact Log: [direct Airtable URL]

CALL 2 (Day 10 after Email 2):
- [...]

[etc.]
```

### 4.3 Filter logic for the call list

Pull contacts where ALL true:
- Stage ∈ {"Email 1 Sent", "Email 2 Sent"} (the trigger stages for calls)
- Days since Last Email Date = 3 (for Call 1) or 3 (for Call 2 - i.e. 10 days from Email 1, 3 days from Email 2)
- Last Call Date is blank OR more than 7 days ago
- Stage ≠ "Not Interested - Closed", "Responded", "Won", "Lost"
- Unsubscribed = false (both Companies and Email Finder where applicable)

---

## 5. Managing-Agent two-pass logic

### 5.1 Pass A - Companies (Standard + Multi-Site)

- Filter: Track ∈ {"Standard", "Multi-Site"} OR Track blank
- Skip: Track = "Managing-Agent" (handled by Pass B)
- Read: Stage, Last Email Date, Last Call Date from Companies
- Send to: Companies.Contact Email
- Write back: Stage, Last Email Date to Companies

### 5.2 Pass B - Email Finder (Managing-Agent, lenient mode)

- Trigger: Either Companies.Track = "Managing-Agent" OR Email Finder.Track Override = "Managing-Agent"
- For each linked Email Finder contact at Managing-Agent Companies:
  - Read Stage, Last Email Date, Last Call Date from **Email Finder** (NOT Companies)
  - Read shared data (Company Name, Sector, Registered Office) via Company link
  - Send to: Email Finder.Email Found
  - Write back: Stage, Last Email Date to Email Finder
- Unsubscribe check: skip if EITHER Email Finder.Unsubscribed = true OR linked Companies.Unsubscribed = true

### 5.3 Token sources for Managing-Agent emails

| Token | Source |
|---|---|
| `{{ContactName}}` | Email Finder.Contact Name |
| `{{ContactEmail}}` | Email Finder.Email Found |
| `{{JobTitle}}` | Email Finder.Job Title |
| `{{UnsubscribeLink}}` | Built from Email Finder.Unsubscribe Token (new) |
| `{{CompanyName}}` | Companies.Company Name (via link) |
| `{{Sector}}` | Companies.Sector (via link) |
| All financial tokens | Companies-derived (via link) |

---

## 6. Per-contact unsubscribe (Option B work)

### 6.1 Token generation

When Pass B drafts first email for a contact:
- Check Email Finder.Unsubscribe Token field
- If blank: generate UUID via `Utilities.getUuid()`, write back to that field
- Build URL: `https://nova-limited.com/unsubscribe?t={token}`

### 6.2 Token lookup endpoint (future work)

A nova-proxy Worker endpoint (`/unsubscribe?t=`) that:
1. Looks up the token in Email Finder via Airtable proxy
2. If found, ticks Email Finder.Unsubscribed = true on that record
3. Returns a confirmation page

This is **separate work** - the Apps Script just generates tokens; the click-handling Worker endpoint can be built later. Until built, the per-contact tokens exist in URLs but clicking them lands on a 404. Companies-level unsubscribe still works via existing mechanism.

### 6.3 Suggested phasing

| Phase | Work | When |
|---|---|---|
| 1 | Tuesday spec - schema, Pass B, call reminders | Tue 26 May |
| 2 | Generate unsubscribe tokens (still landing on 404) | Tue 26 May (same script work) |
| 3 | Build /unsubscribe Worker endpoint | Future session |
| 4 | Replace 404 landing with confirmation page | Future session |

Until phase 3, Companies.Unsubscribed checkbox is the working kill-switch. Per-contact unsubscribe links exist but require manual action by Jamie if clicked (recipient emails you, you tick the Email Finder.Unsubscribed checkbox).

---

## 7. "Not Interested - Closed" handling

### 7.1 Where it gets set

Manually, by Jamie, after a call where the prospect/contact says they're not interested. Two ways:

**For Standard/Multi-Site:** Set Companies.Stage = "Not Interested - Closed"
**For Managing-Agent:** Set Email Finder.Stage = "Not Interested - Closed" (per-contact)

### 7.2 What it does

- Stops all future email drafts for that prospect/contact
- Stops all future call reminders
- Hides from daily call list
- Does NOT delete the record (history preserved for due-diligence and to prevent re-prospecting the same contact)

### 7.3 Re-engagement override

If you want to re-engage a closed prospect later (12 months on, new context), manually change Stage to "New" and clear Last Email Date - the sequence starts fresh.

---

## 8. Call script opener - decided framing

**Use Option A (direct, no apology) as default:**

> "Jamie Gibbs from Nova. I sent you an email Tuesday about your 23,000 sqm portfolio. Two-minute question if you've got it - if not, when's better?"

**Variations by call number:**

**Call 1 (Day 3):** As above. Reference Email 1 content (portfolio shape, sector).

**Call 2 (Day 10):** "Jamie Gibbs from Nova. Sent you the figures last Tuesday - £4.8m total spend, 5.6 year payback on the solar. Wanted to walk through assumptions if you've got two minutes."

**Voicemail (if no answer):** "Jamie Gibbs from Nova, MD. Sent an email Tuesday about your portfolio - just trying to catch you. Call me back on 0203 479 2994 or I'll try again in a week. Cheers."

**No call competition fiction.** Reasons documented in chat: too risky, contradicts premium positioning, MD-personally-calling is genuinely better.

---

## 9. Contact Log integration

### 9.1 Existing fields (no changes needed)

Contact Log (`tblDO47IagkE4W6Lp`) already has: Prospect, Contact Name, Email, Method, Subject, Outcome, Follow Up Date, Completed, Notes, Date. This is sufficient.

### 9.2 Workflow

After each call, Jamie logs in Contact Log:
- **Method:** Phone (existing option, verify)
- **Outcome:** singleSelect - verify it has appropriate options (e.g. "Spoke - interested", "Spoke - not interested", "Voicemail left", "No answer", "Gatekeeper - try again")
- **Notes:** What was said
- **Date:** Today
- **Follow Up Date:** If applicable, when to retry

### 9.3 Apps Script integration with Contact Log (optional, future)

The script could read Contact Log to auto-update Last Call Date on Companies/Email Finder when a call is logged. **Out of scope for Tuesday** - manual Last Call Date update is fine to start.

---

## 10. Tuesday's build order

Recommended sequence (estimated 3-4 hrs total):

1. **Schema first** (Airtable MCP, 15 min): Add 5 fields to Email Finder, 1 field to Companies, modify Stage choices on both. Verify via MCP.
2. **Pass A refinement** (45 min): Add Track filter to existing Companies iteration. Skip Managing-Agent. Test with Grant & Stone draft.
3. **Pass B build** (90 min): New iteration for Email Finder where Companies.Track = Managing-Agent OR Email Finder.Track Override = Managing-Agent. Token sourcing per section 5.3. Lenient mode auto-include logic. Test with a placeholder Managing-Agent record.
4. **Token generation** (30 min): UUID write to Email Finder.Unsubscribe Token on first draft.
5. **Call reminder email** (45 min): Daily summary email shape per section 4.2. Pull filter logic per section 4.3.
6. **Outcome verification check** (15 min): Confirm Contact Log Outcome field has the options listed in 9.2; add any missing.

---

## 11. Out of scope (for Tuesday)

- `/unsubscribe` Worker endpoint (phase 3, future session)
- Contact Log → Last Call Date auto-sync (manual update is fine)
- Algorithm v4 (banded SOLAR_GBP_PER_KWP, modern W_M2)
- Pin Drop Prospecting spec (separate Tuesday task per memory entry 22)

---

*Spec written Mon 25 May 2026 evening. Source of truth for Tuesday Apps Script work. Update this doc if decisions change during build.*
