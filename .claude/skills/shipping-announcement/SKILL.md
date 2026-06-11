---
name: shipping-announcement
description: "Drafts personalized, CSM-voiced customer announcements from TWO sources: (A) customer-facing feature ships posted in the #shipped Slack channel, and (B) closed Production-Support bug fixes in Linear. For each, it finds the customers who requested/reported the work via Linear customer_needs, looks up each customer's Primary CSM in Salesforce, drafts a warm copy-paste Slack message in that CSM's voice (a launch tone for ships, a reassuring 'we fixed it' tone for bugs), and DMs it to the CSM. Never posts to a customer channel. DRY-RUN by default. Use when someone says \"run the shipping announcement agent\", \"draft shipped announcements\", \"check #shipped\", \"announce the new feature to customers\", \"draft bug-fix announcements\", \"who reported this bug / closed prod-support ticket\", \"tell customers their bug is fixed\", or \"run the announcement agent\"."
---

# Shipping & Bug-Fix Announcement Agent

> **IMPORTANT: Execute all steps below immediately and autonomously. Do not ask the user clarifying questions before starting. All configuration, defaults, and decision rules are defined in this document — use them as-is.**
>
> **DRY-RUN is the default.** On a dry run you do everything EXCEPT send DMs — you print each draft + its target CSM + target customer + source so the user can validate targeting and voice. Only send real DMs when the user explicitly says **LIVE** / "send for real" / "go live".

**Objective:** Notify the right CSM whenever a customer's request or reported issue is resolved, with a ready-to-paste customer message in the CSM's voice. Two independent **sources** feed one shared delivery pipeline:

- **Source A — Feature ships.** A *customer-facing* post in the **#shipped** Slack channel → find the Linear ticket(s) → gather requesting customers → draft a **launch** message.
- **Source B — Bug fixes.** A **closed Production-Support ticket** in Linear → gather the customers who **reported** it → draft a **reassuring "we fixed it"** message.

Both sources normalize into the same *announcement item* and run through the same back half (resolve customer → Salesforce CSM → Slack user → draft → DM the CSM). **Never message a customer channel directly.**

---

## Configuration

```
# --- Shared ---
SLACK_TEAM_ID            = T02ASCQECCV        # Rillet workspace
SF_CONNECTION            = Salesforce1         # CData Connect, schema: Salesforce
SF_INSTANCE_URL          = https://rillet.my.salesforce.com
STATE_FILE               = shipping-announcement-state.json        # repo-root-relative; cloud routine CWD = repo root. Read at start, write+commit+push at end.
CHANNEL_MAP              = channel-mapping-export.json             # repo-root-relative; customer → external_channel_url for the paste-target link
MODE                     = DRY-RUN             # DRY-RUN (default) | LIVE  (the scheduled cloud routine overrides to LIVE)

# --- Source A: #shipped feature ships ---
SHIPPED_CHANNEL_ID       = C0AMY7FDY8P         # #shipped
DONE_WINDOW_DAYS         = 7                    # match Linear tickets completed within this many days BEFORE the post
DONE_WINDOW_AFTER_DAYS   = 2                    # ...and up to this many days AFTER the post (slack post can lag the ticket)

# --- Source B: Linear Production-Support bug fixes ---
PROD_SUPPORT_PROJECT_ID  = 92fe53cf-cde0-416c-bea4-c91899761bb7   # Linear project "Production Support"
PROD_SUPPORT_PROJECT_NAME= Production Support
URGENCY_LABELS           = Level 1, Level 2, Level 3, Level 4, Level 5   # team-scoped urgency labels (NOT "L1".."L5")
BUG_LOOKBACK_DAYS        = 7                    # on each run, look back this many days for newly-completed tickets (watermark dedupes overlap)
```

**Customer gate by source:**
- **Source A (ships):** process only if the #shipped post explicitly says customer-facing = yes (a line like `*Customer facing?* big yes`). If "no", absent, or ambiguous → skip the post (still mark processed).
- **Source B (bugs):** the gate is **whether the closed ticket has any `customer_needs`**. If it has none, nothing needs to happen → skip silently (mark processed). No "customer-facing" line exists on these tickets.

**Customer → Slack channel mapping:** look up each customer in `CHANNEL_MAP` (`channel-mapping-export.json`) by `customer_name` (clear correspondence; same name-match discipline) and include its `external_channel_url` as the paste-target link in the CSM DM. If the entry has no `external_channel_id` (blank), flag "⚠ no external channel on file — send via your usual channel".

---

## State file

`shipping-announcement-state.json` in this skill's folder:
```json
{
  "last_ts": "0",
  "processed_ids": ["<#shipped post ts values, capped at last 300>"],
  "last_bug_completed_at": "0",
  "processed_issue_ids": ["<Linear issue identifiers e.g. APE-872, capped at last 300>"],
  "csm_cache": { "<csm name, lowercased>": { "slack_id": "U…", "matched": true } },
  "customer_account_cache": { "<linear customer name, lowercased>": { "sf_account_id": "001…", "csm": "Jane Doe" } }
}
```
- If the file doesn't exist or a field is missing: `last_ts="0"`, `processed_ids=[]`, `last_bug_completed_at="0"`, `processed_issue_ids=[]`, both caches `{}`.
- `processed_ids` dedupes Source A (Slack ts). `processed_issue_ids` dedupes Source B (Linear identifier). Keep them separate — a ticket can appear in both a #shipped post AND as a prod-support close; dedupe is per-source by design, and the back half de-duplicates the actual (customer, item) drafts so the same customer isn't double-DM'd in one run.
- Caches are reuse hints, not sources of truth. A cache entry with `"matched": false` means "tried, couldn't resolve" — don't keep re-querying every run, but DO surface it in the summary.

---

## Voice guides

Every draft must read like a CSM wrote it by hand. **Pick the guide by the item's source.**

### A. Feature-launch voice (Source A) — anchor on these real Shannon Hendricks examples

**Example A — short form (Niche):**
> Hi team 👋 hope you had a great weekend! Just wanted to update you on one of your feature requests –
> 🚢 *Date filter on drill-downs*
> *What*: you can now filter by date on the income statement & balance sheet drill-downs. This lets you more easily see exactly what went into a specific number

**Example B — rich form (Nisos):**
> Hi Nisos team 👋 We just shipped a feature we know you've been waiting for: *Custom Rows & Formulas* on the income statement.
> You can now add formula rows directly to your P&L — including EBITDA, subscription gross margin, or any metric your team tracks. … Your default P&L stays untouched.
> A few things you can build today:
> • *EBITDA* (Net income + taxes + depreciation + interest expense)
> • *Opex as % of revenue* or any ratio using account values
> We'd love your honest feedback as you try it — what works, what's missing, what you'd want next.
> Full instructions: <docs link>

**Feature voice rules (the shared DNA):**
- Warm, personal greeting addressed to the customer ("Hi {Customer} team 👋"). Light, natural emoji — not corporate.
- Frame it as *their* request: "one of your feature requests", "a feature we know you've been waiting for".
- Plain-language **benefit in second person** ("you can now…"), tied to what the *What* line says.
- Optional short bullets only if the feature has a few concrete uses (mirror Example B). Otherwise keep it to Example A length.
- Include the docs link if the #shipped post has one.
- **3–5 sentences max.** No formal sign-off, no "Best regards", no automated/templated feel. Don't invent capabilities not in the *What* line.

### B. Bug-fix voice (Source B) — casual "give it another go" check-in

Anchor on this real Kayla Cameron message to a customer contact after a fix shipped:

**Example (date-selection fix):**
> hey @adam — do you mind giving the date selection another go in the revenue section of the contract builder? team was able to push a fix and it looks to be toggling (selecting?) better on my side! 🤞

**Bug-fix voice rules (the DNA of the example):**
- **Casual and very short.** Usually ONE sentence, occasionally two. Lowercase "hey" is fine. Light emoji (🤞, 🙂) — at most one.
- **Open with "hey {Customer} team —"** (e.g. "hey Niche team —"), matching the feature-side greeting. (The real example addressed one contact by name — "@adam" — but we default to the team greeting; the CSM can swap in an individual name before sending.)
- **Frame it as "give X another go" / "mind retrying X?"** — invite the customer to *re-test*, rather than declaring "it's fixed." This is the signature move of the example.
- **Reference the exact thing they reported, in plain words** (drawn from the ticket title/description) — e.g. "the date selection in the revenue section of the contract builder", not "the bug."
- **Humble and hedged. Do NOT over-claim.** "team was able to push a fix", "looks to be working on my side", "should be sorted now." Never assert it's definitely fixed for them — that's what the re-test is for. Credit "the team," not yourself.
- **No celebratory/launch framing** (no 🚢, no "excited to announce"), no bullets, no formal sign-off, no root-cause detail beyond what the ticket says.
- This is a draft the CSM edits and sends; the CSM owns the relationship and the exact contact.

> Both guides: this is a **draft for the CSM**, not the final word — the CSM edits and sends. Address the customer, never internal Rillet teammates. The user may add more examples (esp. other CSMs) later.

---

## Procedure

### 1. Load state + get today
- Read `STATE_FILE` (Read tool). Initialize any missing field to its default.
- Get today's date/time (PowerShell `Get-Date`; epoch via `[int][double]::Parse((Get-Date -UFormat %s))`). Needed for both the Linear completed-window math (Source A) and the bug lookback watermark (Source B).

### 2. Collect announcement items (run BOTH sources)

Each source produces zero or more **normalized announcement items** with this shape:
```
{
  source:    "ship" | "bugfix",
  ref:       "<#shipped post ts>"  |  "<Linear identifier, e.g. APE-872>",   # for dedupe + summary
  title:     "<feature title>"     |  "<short 'what was fixed' summary>",
  what_text: "<authoritative description of what shipped / what was fixed>",
  docs_link: "<url or null>",
  date:      "<post date / completedAt>",
  tickets:   [ <matched Linear ticket idents> ],
  customers: [ { name, id } … ]    # unique
}
```

#### 2A. Source A — #shipped feature ships
1. `slack_read_channel` on `SHIPPED_CHANNEL_ID`, `oldest`=`last_ts`, `limit` 50, `response_format:"detailed"`.
2. Keep only **parent posts** (not thread replies) that look like a ship announcement (contains `:ship:`/🚢 or a bold feature title + a `*What*`/`*What:*` line). Drop any in `processed_ids`. Process oldest-first.
3. **Customer-facing gate:** read the body for the customer-facing field. If it does NOT clearly say customer-facing = yes → skip (add to `processed_ids`, note "skipped: not customer-facing"). Else continue.
4. **Extract the feature:** `title` = bold text after 🚢/`:ship:`; `what_text` = text on/after the `*What*` line (authoritative — use for matching AND drafting); `docs_link` = any `docs.rillet.com`-style URL; `ref`/`date` = the parent message ts/date.
5. **Find the Linear ticket(s)** — one ship can map to multiple tickets. **FIRST STOP (authoritative): the tagged Linear ticket link(s).** The team now tags the Linear ticket(s) for a ship directly in the `#shipped` **post body and/or its thread**, so a tagged link is the intended, highest-confidence source — always look there before anything else.
   - **5a. Tagged-link seeds (authoritative — do this first):** scan BOTH the **parent post body** AND the **thread** (`slack_read_thread` on `SHIPPED_CHANNEL_ID`/post ts) for every `linear.app/rillet/issue/<IDENT>` link (dropped by humans or the "Rillet Rex" bot). **If one or more tickets are tagged, treat them as THE match** — trust them even if a tagged ticket's `completedAt` falls slightly outside the window (a human explicitly tied it to this ship). Use 5b only to *supplement* (catch sibling tickets the same ship covers), not to second-guess a tag.
   - **5b. Keyword search (FALLBACK — only if NO ticket is tagged):** if 5a found no tagged link, `list_issues` with `query` = key terms from `title`/`what_text`, `limit` 50 (do NOT restrict by team), then apply 5c+5d to the candidates.
   - **5c. Filter to completed-in-window** (applies to 5b keyword candidates; tagged 5a seeds are kept regardless of window): keep a candidate only if `statusType="completed"` AND `completedAt` ∈ `[date − DONE_WINDOW_DAYS, date + DONE_WINDOW_AFTER_DAYS]`.
   - **5d. Semantic confirm:** confirm each surviving ticket's title/description actually matches `what_text`. Drop clear mismatches. Keep borderline matches but mark them low-confidence for the "review these matches" summary line. If none survive (no tag AND no keyword match), record "no matching Linear ticket found" and move on (still mark processed).
6. **Gather customers (ticket-level + project-level):** for each matched ticket:
   - **Ticket-level:** `get_issue(id, includeCustomerNeeds:true)` → collect each `customerNeeds[].customer.name`/`.id`.
   - **Project-level (SAME-FEATURE filter — important):** read the ticket's `projectId`; `list_issues(project=<projectId>, limit 250)` and for each issue `get_issue(..., includeCustomerNeeds:true)`. **Do NOT blindly union the whole project.** Include an issue's customers ONLY if that issue's own request **semantically matches the ship's `what_text`/`feature_title`** (i.e. it's the same feature, just a different customer's request for it). **Drop issues that are about a *different* feature**, even in the same project. Many projects are grab-bag buckets (e.g. "Reporting Improvements" mixes custom rows, trial balance, PDF export, cash-flow drill-down…) — unioning the whole thing would announce this ship to customers who asked for something else, sometimes for features that haven't even shipped. When in doubt about a borderline issue, EXCLUDE it (or list it under "review these" rather than auto-including).
   - Dedupe customers across all matched tickets + the same-feature project issues (case-insensitive) → the item's `customers`.
   - *Rationale:* a feature serves everyone who requested **that feature** — so we union same-feature requests across the project (one EBITDA ship legitimately covers ~11 customers who asked for EBITDA), but we do NOT spam customers whose request was a different line item that happens to share a project. *(Known limitation: a customer request attached to the project itself, not to any issue, won't surface — note in summary.)* The Primary CSM is still the final human review gate.
7. Emit one `ship` item per qualifying post. Mark the post ts processed.

#### 2B. Source B — Linear Production-Support bug fixes
1. **Find newly-completed prod-support tickets.** Union two queries (then dedupe by identifier):
   - **By project:** `list_issues(project=PROD_SUPPORT_PROJECT_ID, state="completed", orderBy="updatedAt", limit 100)`.
   - **By urgency label:** for each label in `URGENCY_LABELS`, `list_issues(label="<Level N>", state="completed", orderBy="updatedAt", limit 50)`. (These catch prod-support tickets that live on a team board but outside the project — e.g. APE-872 was `Level 3` with no project.)
2. **Recency + dedupe filter.** Keep a ticket only if `statusType="completed"` AND `completedAt` is within the last `BUG_LOOKBACK_DAYS` days AND its identifier is NOT in `processed_issue_ids`. (The lookback + watermark `last_bug_completed_at` make overlap safe; `processed_issue_ids` is the hard dedupe.)
3. **Gather candidate customers (two sources, unioned).** `get_issue(id, includeCustomerNeeds:true)`, then collect from BOTH:
   - **(a) Ticket-level `customer_needs`** — each `customerNeeds[].customer.name`/`.id`. **Ticket-level ONLY — do NOT do the project union here** (unlike features): only customers who actually reported/hit the bug should be told. (e.g. APE-872 → Neuralink + Traba; RAT-2823 → Niche + ChartHop — both get a draft.)
   - **(b) The `Org Name:` line** parsed from the description (also `Org:` / a `## Summary` `Org Name:`). This catches bugs where the customer was named only in free text and never tagged as a customer_need (e.g. CAT-2055 titled "…— Bolt" had zero customer_needs). Mark any customer sourced this way as **"(via Org Name)"**.
   - **Union (a)+(b), dedupe by name** (case-insensitive). A name appearing in both is one customer; prefer the `customer_need` id.
4. **Customer gate + Org-Name hygiene.** If the union is empty → **skip silently** (add to `processed_issue_ids`, note "skipped: no customer"). When parsing `Org Name:`:
   - **Ignore non-customer values:** `All orgs`, `All?`, blank, "All Orgs", or anything that isn't a single identifiable company → don't treat as a customer.
   - **Strip trailing qualifiers** before matching: `(sandbox)`, `(New)`, `(All Orgs)`, etc. The stripped name must still map to exactly ONE real Salesforce Account under the strict name-match discipline (step 3 of the back half) — if it's ambiguous or zero, treat as **unresolved** (needs-attention), never a guess.
   - An `Org Name:`-sourced customer is lower-confidence than a `customer_need` — still draft it, but tag it "(via Org Name)" in the DRY-RUN output and summary so it's reviewable.
5. **Build `what_text` (what was fixed).** The ticket describes the *problem*; summarize it as a *resolution* for drafting. Use the ticket `title` + `description` (strip internal noise: Org ID/Sub ID, trace IDs, Honeycomb links, screenshots). If a Pylon attachment or a completion comment clarifies the fix, use it for accuracy. Do NOT overstate beyond what the ticket says. `title` = a short customer-readable phrasing of the resolved issue.
6. `docs_link` = a customer-facing docs URL only if the ticket plainly has one (rare for bugs — usually null). `ref` = the Linear identifier; `date` = `completedAt`.
7. Emit one `bugfix` item per qualifying ticket. Mark the identifier processed.

> After 2A + 2B you have a combined list of items. If the SAME customer appears on both a ship item and a bug item this run, that's fine — they're distinct announcements; the CSM gets both, grouped in one DM (see step 6).

### 3. (shared) Resolve each customer → Salesforce Account → Primary CSM
For each unique customer name across all items:
- Check `customer_account_cache` first (case-insensitive). If present, reuse `sf_account_id` + `csm`.
- Else query Salesforce (CData `queryData`, fully-qualified table):
  ```sql
  SELECT [Id], [Name], [Type], [New_Primary_CSM__c], [New_Secondary_CSM__c]
  FROM [Salesforce1].[Salesforce].[Account]
  WHERE [Name] LIKE '<customer_name>%'
  ORDER BY [Name] LIMIT 5
  ```
  - **Name-match discipline (hard rule):** do NOT assume two differently-named companies are the same customer. Accept a match only when the account name clearly corresponds to the Linear customer name. (CData compiles `LIKE` to regex; if a name has regex-special characters, match in code from a broader result set rather than trusting `LIKE`.)
  - **Use `[Type]` to disambiguate and filter:** prefer the account whose `Type` = `Customer - Active`. A `Prospect`-type account is NOT an active customer → never announce to it. When the name returns **multiple** accounts, pick the single `Customer - Active` one (e.g. two "Posh" accounts — one `Prospect`, one `Customer - Active`; the active one is correct). If multiple remain genuinely ambiguous after the Type filter, or none is `Customer - Active`, treat as **unresolved / needs-attention** — do not guess.
  - **CSM rule — use the `New_*` fields ONLY:** use `New_Primary_CSM__c`; if blank, fall back to `New_Secondary_CSM__c` (note in the DM it's going to them as Secondary CSM). **⚠️ Do NOT use the deprecated `Primary_CSM__c` / `Secondary_CSM__c` fields** — they are stale and store a User Id, not a name; the `New_*` fields are the source of truth and store the CSM's **full name** directly (e.g. `Shannon Hendricks`). **If BOTH `New_*` fields are blank, the account is almost certainly NOT an active customer** (prospect/churned/test) → **skip silently**, no DM, note "skipped: no CSM (likely not an active customer)". Not an error.
- Cache the result (including `"matched": false` for unresolved).

### 4. (shared) Resolve the CSM (a name) → Slack user
The `New_Primary_CSM__c` / `New_Secondary_CSM__c` fields give a CSM **full name** (e.g. `Shannon Hendricks`). Resolve name → Slack in two hops (don't DM by raw name — too ambiguous):
- **Name → email** (batch all distinct CSM names in one call):
  ```sql
  SELECT [Id],[Name],[Email],[IsActive] FROM [Salesforce1].[Salesforce].[User]
  WHERE [Name] IN ('<csm name 1>','<csm name 2>',…)
  ```
  (CSM users may show `IsActive=false` even when active in Slack — informational, not a blocker. If a name returns multiple User rows, mark unresolved/needs-attention — don't guess.)
- **Email → Slack user:** check `csm_cache` first; else `slack_search_users` with the CSM's **email** (most reliable). Accept only an **unambiguous single match** on the Rillet workspace. Zero or multiple → mark unresolved, do NOT guess. Cache `{ slack_id, name, email }`.

### 5. (shared) Draft the message (per customer, per item)
For each (customer, item) where the customer resolved to a CSM, write ONE customer-facing draft using the **voice guide for that item's `source`**:
- **`ship` → Feature-launch voice (A).** Greet the customer, announce `title` as their request, 1–2 sentence second-person benefit from `what_text`, optional bullets, docs link if present. 3–5 sentences.
- **`bugfix` → Bug-fix voice (B).** Greet the customer, acknowledge the specific issue they raised (from `what_text`), state plainly it's resolved and what now works, optional "let us know it's working for you". 2–4 sentences, reassuring not celebratory.
- Never invent capabilities/fixes beyond `what_text`.

### 6. (shared) Deliver to the Primary CSM
Group all items' drafts **by CSM** (one CSM may cover several customers and/or both a ship and a bug). Assemble one DM per CSM. **Each item carries a `ref:` line** linking its source so the CSM can verify context — for ships: the matched Linear ticket(s) **and** the `#shipped` post permalink; for bugs: the Linear ticket. **These reference links go in the CSM-facing header only — NEVER inside the blockquote** (the blockquote is the verbatim text the customer sees; it must contain no internal Linear/#shipped links).
```
Here are draft announcements you can send to your customers:

🚢 *New feature notification: <ship title>* — for <Customer(s)>            (ship blocks first)
ref: <Linear ticket link(s), e.g. CAT-435> · <#shipped post permalink>
*Customer:* <Customer Name>  →  <customer channel link, or ⚠ note if none>
*Paste into that customer's channel:*
> <the drafted launch message — NO internal links>

🛠️ *Bug fixed: <bug title>* — for <Customer(s)>       (bug blocks)
ref: <Linear ticket link, e.g. RAT-2823>
*Customer:* <Customer Name>  →  <customer channel link, or ⚠ note if none>
*Paste into that customer's channel:*
> <the drafted fix message — NO internal links>

_(Edit to taste, then paste into the right customer channel — I won't post it for you.)_
```
- Build the `#shipped` permalink from the post channel + ts: `https://team-rillet.slack.com/archives/<SHIPPED_CHANNEL_ID>/p<ts without the dot>`. Linear ticket links come straight from each issue's `url`.
- **DRY-RUN (default):** print the full payload(s) to the user, grouped by CSM, each block labeled with its source, resolved Slack handle, and customer name(s). **Send nothing.**
- **LIVE:** `slack_send_message` with `channel` = the CSM's **Slack user ID** (DMs accept a user ID as channel). One DM per CSM. Never send to a customer channel under any circumstances.
- **Skipped (no CSM):** accounts with neither Secondary nor Primary CSM → skip silently (summary note only), never "needs attention".
- **Genuinely unresolved** (name matched a real account WITH a CSM but the CSM can't be resolved to Slack, or the name matched multiple/zero accounts): never DM a customer; collect into "needs attention" with the reason.

### 7. Update state
Write `STATE_FILE`:
- `last_ts` = ts of the most recent #shipped post processed (Source A).
- `processed_ids` = union of prior + newly processed post ts (cap last 300).
- `last_bug_completed_at` = the max `completedAt` among bug tickets processed this run (Source B watermark).
- `processed_issue_ids` = union of prior + newly processed Linear identifiers (cap last 300).
- Refresh `csm_cache` and `customer_account_cache`.

### 8. Run summary
Print:
- **Source A:** posts checked / processed / skipped-not-customer-facing; per ship: title, matched ticket(s) (flag low-confidence), # unique customers, project-level limitation note if relevant.
- **Source B:** prod-support tickets checked / processed / skipped-no-customer-tagged; per bug: identifier + title, # customers tagged.
- **Drafts produced,** grouped by CSM (Primary, or Secondary-as-fallback), each tagged ship/bug, and whether DM'd (LIVE) or withheld (DRY-RUN).
- **Skipped as non-customers:** accounts with no CSM at all (name them — expected).
- **Needs attention:** customers whose name matched zero/multiple SF accounts, accounts whose CSM couldn't be resolved to Slack, and ship tickets needing match review.

---

## Notes & limitations
- **Membership:** the running Slack account must be a member of #shipped (Source A). DMs go to the CSM's user, not a channel.
- **Two gates, two shapes:** Source A gates on the post's "customer facing? yes"; Source B gates on the presence of `customer_needs`. Both skip-and-mark-processed when the gate fails.
- **Bug customers = ticket-level `customer_needs` UNION the parsed `Org Name:`** (the people who actually hit it), never the project-level union. Feature customers include a **same-feature** project-level union: customers from other project issues are added ONLY if their request semantically matches the shipped feature — NOT the whole grab-bag project (see Source A step 6). This asymmetry is intentional — don't "fix-announce" to customers who never hit the bug, and don't "feature-announce" to customers who asked for a different line item that happens to share a project. `Org Name:`-sourced customers are tagged "(via Org Name)" and are lower-confidence.
- **No volume throttle (by design):** every fix that resolves to a customer gets a draft, regardless of severity/age. The Primary CSM is the human gate who decides whether to actually send. Revisit if CSMs find it noisy.
- **No-CSM customers are silently skipped** even for inbound bugs (a tagged customer with both CSM fields blank in SF is treated as not-currently-assigned and dropped without noise) — same rule as the feature side.
- **Matching is conservative** — borderline ship-ticket matches and ambiguous customer/CSM resolutions are surfaced, never silently resolved. Two different company names are never assumed to be the same customer.
- **Voice:** feature launch is anchored on two real Shannon examples; the **bug-fix voice is pending the user's real example** (Section B is stubbed — fill it before going LIVE on Source B). More examples (esp. other CSMs) can be added later.
- **Going LIVE:** flip `MODE` to LIVE only after a dry run looks right for the relevant source(s). Consider scheduling via `/schedule` once validated.
