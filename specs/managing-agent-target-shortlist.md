# Managing-Agent Cohort 1 - Target Shortlist

**Drafted Mon 25 May 2026 evening.** 10 UK managing agent firms split into two tiers based on strategic approach. To be enriched via nova-tool tomorrow morning, then loaded into Airtable for the Tuesday Apps Script run.

---

## Tier A: Big 4 / National (3 firms) - PHONE ONLY, NO EMAIL SEQUENCE

These firms have PSL gates and procurement processes that defeat cold email. The play is direct phone contact to start a relationship that may lead to PSL nomination over months/years. Per Jamie's instinct: getting a phone conversation may surface mutual connections (ex-Royal Marines network, ex-IoR contacts, ex-construction law course peers) that unlock the relationship.

**Airtable handling:**
- Track = Managing-Agent
- Stage = New
- Unsubscribed = **ticked** (semantic workaround to prevent Apps Script auto-emailing)
- Notes = "Tier A - phone-only approach. Unsubscribed flag is gate, not opt-out. Do not remove without changing Stage."

### 1. JLL (Jones Lang LaSalle Limited)
- **CH number:** 01188567 (likely - verify via nova-tool)
- **Why:** Largest commercial PM firm operating in UK, $26.2bn global revenue 2025. Manages industrial and logistics estates at scale. If we get to PSL the order pipeline is enormous.
- **Strategic angle:** Birmingham office runs Global Occupier Services with industrial/logistics focus. Easier entry than London for a non-London supplier.
- **Phone research priority:** Head of UK Property Management, plus regional industrial team leads. The estates director title typically authorises £200k+ roof spend.

### 2. Workman LLP
- **CH number:** verify via nova-tool (Workman LLP is LLP, not Limited)
- **Why:** £3bn+ annual rent under management. Pure commercial property management - this is their entire business, not a Big 4 division. More approachable than JLL/CBRE because PM is the core service not a sideline.
- **Strategic angle:** Strong industrial portfolio. Less procurement bureaucracy than the Big 4 because LLP structure. Most realistic Tier A prospect for actual contract work in 12-18 months.
- **Phone research priority:** Head of Industrial Asset Services or equivalent regional roles.

### 3. Lambert Smith Hampton (LSH)
- **CH number:** verify via nova-tool
- **Why:** 32 UK offices, 1,000+ staff, dedicated industrial advisory team. Mid-tier between Big 4 and regional.
- **Strategic angle:** SW England office presence makes Plymouth-based engagement plausible. Industrial focus is genuine, not bolt-on.
- **Phone research priority:** Industrial advisory directors at Bristol or Birmingham offices.

---

## Tier B: Mid-Tier National (7 firms) - FULL EMAIL + PHONE SEQUENCE

These firms are scaled enough to have multi-site portfolios that benefit Nova's offer, small enough to respond to cold email, and don't have prohibitive procurement gates. They enter the Day 0/3/7/10/14/21 sequence Tuesday after Apps Script work is done.

**Airtable handling:**
- Track = Managing-Agent
- Stage = New (will progress to Email 1 Sent when sequence fires)
- Unsubscribed = unticked
- Each linked Email Finder contact gets Track Override = Managing-Agent (or relies on lenient mode - Company.Track is enough)

### 4. Vail Williams LLP
- **Why:** UK regional commercial property consultancy with industrial focus. Multi-office (Birmingham, Bristol, Southampton, Reading). Mid-tier scale - probably under 200 staff, manageable response rate likely.
- **Strategic angle:** Southampton office covers SW gateway - natural for Plymouth-based engagement.
- **Verify:** SIC 68320 registration, decision-maker name via CH officers endpoint.

### 5. Fisher German
- **Why:** Mentioned in Avison Young hiring news as a feeder firm. Rural and industrial property management focus. Multi-office UK.
- **Strategic angle:** Industrial/rural mix means commercial buildings on out-of-town industrial estates - exactly the asbestos cement / box profile metal stock Nova specialises in.
- **Verify:** SIC code, key contacts.

### 6. BNP Paribas Real Estate UK
- **Why:** Mentioned as competitor in Avison Young hires. UK arm of European bank's PM division. Mid-tier scale (smaller than Big 4 but proper national reach).
- **Strategic angle:** Already engaged with industrial sector. Birmingham + London + regionals.
- **Caveat:** May lean to Tier A if PSL process exists. Worth phone-screening before emailing.

### 7. Matthews & Goodman (now Fisher German subsidiary or rebrand - verify)
- **Why:** Listed as origin firm in Avison Young hires - so still has staff identity even if rebranded. Industrial focus.
- **Verify:** Whether still independent or fully absorbed - if absorbed, drop and replace with substitute below.

### 8. Vibe & Co / LCP Properties
- **Why:** LCP Properties mentioned in Avison Young news as retail-focused asset management. Different from pure PM but worth scoping.
- **Verify:** Whether they sub-contract for roofing/M&E or handle in-house. If sub-contract, prime target.

### 9. Sanderson Weatherall LLP
- **Why:** UK-focused regional commercial property firm. LLP structure. Generally responsive to direct outreach.
- **Strategic angle:** Industrial assets across northern UK industrial heartlands.
- **Verify:** Multi-site portfolio scale via CH accounts.

### 10. GVA / Avison Young UK (post-merger)
- **Why:** GVA merged into Avison Young UK in 2018. Post-merger entity is mid-tier national. Strong industrial team based on news intel.
- **Strategic angle:** Multi-office UK including Bristol (SW). Industrial team mentioned in 2025 reporting as growing.
- **Caveat:** May behave like Tier A given parent group scale. Phone-screen first.

---

## Substitution shortlist (if any of #7-#10 turn out to be wrong/merged/too big)

- **Knight Frank Commercial UK** - if Tier A scope changes
- **Carter Jonas** - rural/commercial UK mid-tier
- **Bidwells** - East England industrial focus  
- **Innes England** - East Midlands regional, strong industrial portfolio
- **Allsop LLP** - commercial sales + management hybrid

---

## Tomorrow's enrichment workflow (via nova-tool)

For each of the 10 firms above:

1. Open nova-tool live URL
2. Search Company by name → CH match
3. Click "Use this match" → /places fires → captures phone, website, registered office
4. Click "Find Email" with a known decision-maker name from research
5. Save Prospect → Companies record created with full data
6. For Tier A (3 firms): manually tick Unsubscribed = true on Companies record, add Notes per template above
7. For Tier B (7 firms): leave defaults, set Track = Managing-Agent on Companies record

Time estimate: 10 firms × ~3-4 minutes each = 30-45 minutes total tomorrow morning.

---

## Outstanding work flagged for Tuesday Apps Script session

After this cohort is loaded, the Tuesday Apps Script work needs to:
- Read Track field on Companies (already exists, fldv8VjQJwEmxzO4A)
- Branch to Pass B logic for Managing-Agent
- Use Email Finder.Track Override OR Companies.Track per lenient mode decision
- Respect Unsubscribed flag (already done)

All covered in `~/nova-tool/specs/managing-agent-phone-cadence.md`.

---

## Important caveats on this shortlist

**1. Company names may be incorrect or outdated.** Verify each via nova-tool CH search before saving. Use the CH-search-then-Use-this-match pattern that already handles dedup.

**2. CH numbers above are best-guess.** Multiple group entities exist for most of these firms (JLL has at least 5 UK companies). The right CH entity to target is the one that actually contracts with clients - usually the UK trading entity, not the holding company.

**3. Decision-maker names not pre-populated.** The /officers endpoint on the Worker pulls live directors and PSC data. Use that during nova-tool enrichment rather than relying on stale research names.

**4. Industries not always SIC 68320 specifically.** Big firms have multiple SIC codes (often 68209, 68310, 68320 combined). Don't filter strictly - judgement call per firm.

**5. This is a starter cohort, not a finished list.** Expectation is 1-2 of these will surface as best fit after first contact, and 3-4 will go silent. That's healthy outcome for cold outreach at this volume.

---

*Drafted by Claude in chat 25 May 2026. To be enriched via nova-tool tomorrow morning.*
