# Baron AI — Lead Finder + Visibility Audit

An n8n workflow for finding local businesses, assessing their websites, preparing outreach drafts and running an AI visibility audit on selected leads.

Built for **Baron AI Solutions LTD**. Google Sheets holds the lead records and drafts; the audit produces a separate HTML report delivered by email.

## Why I built it

I built this to connect prospect research with the visibility audit. Before deciding whether a business is worth approaching, I need to understand what it does, how established it is online and whether there is a specific problem I can help with.

Doing that manually means searching for businesses, opening websites, checking reviews, taking notes and writing an individual message for each prospect. The audit then requires another round of research.

This workflow brings those steps together. It creates a shortlist with supporting observations and a draft approach, then lets me run a deeper audit on the leads worth investigating. The purpose is to make research more consistent and give outreach a concrete starting point.

## What it fixes

| Problem | How the workflow addresses it |
| --- | --- |
| Repeated searches for local prospects | Queries Google Places by business category and location. |
| Rechecking businesses already recorded | Compares business name and website URL against existing sheet rows. |
| Inconsistent website assessments | Applies the same HTML checks and scoring rules to each website. |
| Contact details and research scattered across tools | Saves business details, observations, scores and drafts in one sheet. |
| Generic outreach with little context | Asks Claude to draft messages around an observed weakness or opportunity. |
| A separate handover from lead research to auditing | Maps the selected lead into the same input format used by the audit form. |

It automates research and preparation. It does not establish a prospect’s budget, confirm buying intent or improve their website automatically. No conversion results or measured time savings are included in the export.

## How it works

### 1. Find and prepare leads

A manual trigger starts the lead finder. A weekly Monday 08:00 schedule is also configured but disabled in the supplied file; no explicit workflow timezone is set.

The configuration node defines the category, location, limits and destinations. Google Places returns business names, addresses, websites, phone numbers, ratings and review counts. The workflow reads existing sheet rows, removes matching records, optionally excludes businesses without websites and sorts the remaining leads by review count and rating.

### 2. Inspect websites and calculate a score

For each website, an HTTP request fetches the HTML with a 10-second timeout. JavaScript checks for service and contact wording, the target location, FAQ or pricing content, structured markup, booking signals, directory references and marketing tools. It also attempts to extract an email address.

These are checks against the fetched page’s HTML. The workflow does not crawl the whole site or render client-side JavaScript. A “contact page found” result means contact-related wording was detected, not that a contact page was opened and verified.

### 3. Qualify leads and draft outreach

Claude receives the business data and extracted signals. It is asked to return a summary, main weakness, suggested service, priority and reason.

The requested priorities are **Hot Lead**, **Warm Lead**, **Low Priority** and **Not Suitable**. Hot and Warm leads receive five drafts:

- A cold email.
- A LinkedIn message.
- A short phone script.
- A day-three follow-up.
- A day-seven follow-up.

The prompt prohibits guaranteed AI rankings and asks for claims tied to the supplied observations. These rules guide generation; they still need human review.

### 4. Save and summarise

The workflow appends the results to Google Sheets and prepares an email and Telegram run summary with counts, the top five leads by score and a link to the sheet.

**There is no prospect outreach sender in this workflow.** Messages and follow-ups are saved as drafts. Their day-three and day-seven labels do not schedule delivery.

### 5. Run the visibility audit

The audit has two entry points:

- **Manual:** submit a business through the audit form.
- **Automatic:** enable `AUTO_AUDIT_HOT_LEADS` outside testing mode to audit the highest-scoring Hot Lead from the run.

Both feed the shared **Audit Input** node. The audit generates questions, queries Claude, Perplexity and OpenAI, analyses business mentions and competitors, then generates an HTML report. Automatic audit reports go to `notify_email`; form-based reports go to the address entered in the form.

The automatic path selects one business per run. With the single category passed by that path, it normally generates five questions. The form supports up to four services, producing a maximum of 15 questions with the current templates.

## Scoring and interpretation

The lead score is calculated in JavaScript before the model assigns a priority.

| Signal | Points |
| --- | ---: |
| Website URL present | 20 |
| Fetched HTML passes the load check | 10 |
| Service-related wording | 10 |
| Target location mentioned | 10 |
| FAQ or pricing wording | 10 |
| Structured markup detected | 10 |
| Rating and review count meet configured minimums | 15 |
| Directory or social-platform references | 15 |
| SEO or marketing signals detected | 20 |

The possible total is 120, capped at **100**. Businesses without websites can receive up to 15 points for strong reviews when that branch is allowed.

This score measures detected online-presence signals. It is not a probability of conversion. The model’s Hot/Warm classification is a separate judgement that also considers weaknesses.

The audit uses a different measure:

`mention rate = questions marked as mentioning the business / total analysed questions × 100`

That percentage combines service-discovery questions with questions that explicitly name the business. A mention can be neutral or negative; it is not necessarily a recommendation.

## Design decisions

**Keep scoring explicit.** JavaScript handles weights, counts and sorting. The model handles interpretation and drafting. This makes the numeric rules inspectable without relying on the model to perform the arithmetic.

**Use one audit input contract.** Form submissions and selected leads are mapped to the same fields, so both use the same downstream audit logic.

**Separate drafting from sending.** The sheet is the review point for prospect outreach. Research findings can be checked before a message is used.

**Limit expensive work.** The search requests at most 20 results and has no pagination. The default live cap is 15 leads; the optional full audit selects only one Hot Lead.

**Keep recoverable failures visible.** Several integration nodes continue on error. Invalid lead-analysis JSON falls back to a conservative classification with empty drafts. This is partial failure handling, not a guarantee that every execution will complete.

## Configuration

Most settings sit in **EDIT THIS SECTION FIRST**.

| Setting | Supplied value or behaviour |
| --- | --- |
| Location and category | Birmingham, United Kingdom; electricians |
| `max_leads` | 15 |
| `min_rating` / `min_reviews` | 4.0 / 10; used for scoring, not hard exclusion |
| `must_have_website` | `yes` |
| `TESTING_MODE` | `true`; processes up to three leads and marks ordinary rows `TEST` |
| `AUTO_AUDIT_HOT_LEADS` | `false` |
| `REVIEW_MODE` | `true`, but not used by the routing logic |
| `prioritise_marketing_signals` | Defined, but not used by the scoring or sorting logic |
| Sheet, company and notification details | Placeholders requiring replacement |

Testing mode is **not an offline dry run**. It still attempts API requests, website fetches, sheet writes and configured notifications. Dummy businesses are used only when no Places leads are available. The default website filter removes one of the three dummy businesses, so the fallback need not produce three saved rows. Testing mode blocks automatic audits, not the separate audit form.

## Setup

1. Import the JSON into n8n.
2. Configure Google Places Header Auth (`X-Goog-Api-Key`), Anthropic Header Auth (`x-api-key`), Perplexity and OpenAI Bearer credentials, Google Sheets OAuth2 and Gmail OAuth2. Configure Telegram if using that notification channel.
3. Create the leads sheet with the headers below and assign its ID, URL and tab name in the configuration node.
4. Replace the company and notification placeholders. Update the separate contact placeholder in **Generate HTML Report** as well.
5. Reassign credentials in the imported nodes. Keep testing mode enabled and automatic auditing disabled for the first run.
6. Inspect the sheet rows, extracted observations and drafts. Test the audit separately using your own recipient email.
7. After validating both paths, turn testing mode off. Enable scheduling or automatic auditing only when required.

Required sheet headers, in order:

```text
Date Found, Business Name, Business Type, Location, Website URL,
Phone Number, Email, Address, Google Rating, Number of Reviews,
Website Status, Contact Page Found, Service Page Found,
SEO Signals Found, Marketing Signals Found, Lead Quality Score,
Lead Priority, Main Weakness, Suggested Service,
Suggested Outreach Message, LinkedIn Message, Phone Script,
Follow Up Day 3, Follow Up Day 7, Source, Status, Notes
```

The stack is n8n, JavaScript, Google Places, Google Sheets, Anthropic, Perplexity, OpenAI, Gmail and optional Telegram. SerpAPI, Apify/Outscraper and CSV nodes are disabled placeholders, not working alternative integrations. API costs and model compatibility need checking in the target environment.

## Known limitations and next work

- **Marketing signals are weak evidence of spending.** WordPress, analytics tags or an agency credit do not prove an active budget. The current “HIGH INTENT” label and “already spending” note overstate what was observed.
- **HTML checks can misclassify sites.** A keyword does not establish page quality, a blog link does not establish recent activity, and a missing signal does not prove the feature is absent. Extracted email addresses also need verification.
- **Duplicate matching is basic.** It lowercases the combined name and URL but does not canonicalise domains. URL variations can create duplicates; failed sheet reads can also bypass the check.
- **Empty batches need an explicit completion path.** If every lead is filtered out or already recorded, the preparation node returns no items, so the downstream summary may not run. Empty-sheet behaviour also needs validation in the target n8n instance.
- **Save counts are not reconciled per row.** The summary counts prepared rows and checks only the first sheet output for an error. It can overstate successful saves. Automatic audit selection also does not require a confirmed successful save.
- **Model output needs stronger validation.** Parsing JSON is not schema validation. Field types, priority values, draft lengths and factual claims are not fully enforced.
- **Audit scores need clearer boundaries.** Separate brand and discovery questions, distinguish mentions from endorsements and exclude or retry failed analyses rather than counting them as non-mentions.

My next priorities would be verified write results, explicit empty-run handling, evidence-backed website findings and validated model output. Those changes would make the shortlist easier to trust and the workflow easier to operate repeatedly.

## Project status

This README describes the supplied JSON configuration. No live execution, external messages or production validation were performed during this review. The file does not provide evidence of leads converted or client visibility improvements.
