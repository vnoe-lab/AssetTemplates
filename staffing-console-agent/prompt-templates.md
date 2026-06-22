# Staffing + Service Console Agent — Prompt Templates

> Parametrized from the three live prompt templates in the RH Console Agent org.
> Placeholders follow the same convention as `staffing-service-console-agent-definition.md`:
> - `[COMPANY]` — namespace prefix
> - `[COMPANY_LABEL]` — human-readable name
> - `[ROLE_LABEL]` — user role (e.g., "CSMs", "Service Agents", "Case Workers")
> - `[VERIFY — ...]` — items that require org-specific confirmation before use

---

## Template 1: [COMPANY]_Console_Case_Summary

### Purpose

Generates a two-section structured HTML summary of the current Case for the rep. Section 1 is a factual Case Overview (status, issue, sensitivity, key metadata). Section 2 is Requestor History (prior case load for the same Contact). The output is displayed verbatim in the chat panel — the agent script does not post-process it.

### Inputs

| Input name | Type | Required | Source in agent | Parametrization note |
|---|---|---|---|---|
| `Input:Case` | SOBJECT (Case) | Yes | `record_id` variable via `lightning__recordInfoType` | Portable — object type is Case |
| `Input:RequestorCases` | String | No | `requestor_cases` variable | Populated by `Get_Requestor_Cases` flow. If flow fails, defaults to `"No prior cases found for this requestor."` |
| `outputLanguage` | String | No | Not set in RH implementation; can be wired to locale | Wire to `[PRIMARY_LANGUAGE]` or leave unset for default |
| `isPreviewOnly` | Boolean | No | Not set in RH implementation | Only used during testing |

### Model

`sfdc_ai__DefaultGPT41` — GPT-4.1 via Einstein Gateway.

### Retriever wiring

None. This template does not use a Data Cloud retriever. All content comes from the SOBJECT input and the requestor cases string.

### Full Prompt Text (parametrized)

```
You are a case summary assistant for [COMPANY_LABEL] [ROLE_LABEL].

Generate a professional case summary for the [ROLE_LABEL] currently viewing this case.
Output must be formatted in clean HTML suitable for display in a chat panel — no markdown, no code fences.

---

CASE DATA:
{!Input:Case}

REQUESTOR'S PRIOR CASES:
{!Input:RequestorCases}

---

Generate exactly two sections:

<h3>Case Overview</h3>
<ul>
  <li><strong>Case Number:</strong> [from data]</li>
  <li><strong>Status:</strong> [from data]</li>
  <li><strong>Priority:</strong> [from data]</li>
  <li><strong>Subject:</strong> [from data]</li>
  <li><strong>Customer Type:</strong> [Candidate / Client / Third Party / Unknown — from [COMPANY]_Customer_Type__c]</li>
  <li><strong>Country:</strong> [from [COMPANY]_Case_Country__c, if present]</li>
  <li><strong>Created:</strong> [date from data]</li>
  <li><strong>Owner:</strong> [from data]</li>
</ul>

[If [COMPANY]_Case_Sensitivity__c == "SENSITIVE"]
<p><strong>⚠ Sensitive Case:</strong> This case has been flagged as sensitive. Handle all communications with extra care.</p>
[End if]

<h4>Issue Summary</h4>
<p>[2–4 sentence summary of the case description and subject. Factual only — do not infer resolution steps or add advice.]</p>

<h4>Recent Activity</h4>
<ul>
  [Last 3–5 activities from case data, each as a bullet with date and brief description]
</ul>

<h3>Requestor History</h3>
[If requestor cases string is "No prior cases found for this requestor." or empty]
<p>No prior cases found for this requestor.</p>
[Else]
<p>Prior cases for this contact:</p>
{!Input:RequestorCases}
[End if]

Rules:
- Only use data present in the inputs. Do not infer, fabricate, or supplement.
- If a field is blank or absent, omit its list item rather than writing "N/A".
- Do not add a closing offer, follow-up suggestion, or any text outside the two sections.
- Do not include the literal instructions in the output.
```

### Key decisions embedded in the prompt

1. **Two-section structure (Case Overview + Requestor History).** This is the RH-specific output contract. Both sections are always rendered — the Requestor History section is not conditionally hidden when empty; instead the flow supplies a fallback string. Alternative: make Section 2 conditional on the flow returning results.

2. **HTML output, not markdown.** The agent renders into a Service Console chat panel that interprets HTML. If deploying into an LWC or Slack surface instead, switch to plain text or markdown.

3. **Sensitivity flag is a conditional block.** `[COMPANY]_Case_Sensitivity__c == "SENSITIVE"` drives a highlighted warning. If the customer has no sensitivity field, remove this block. If they have a different sensitivity model (e.g., multi-tier), update the condition and copy.

4. **`[COMPANY]_Customer_Type__c` and `[COMPANY]_Case_Country__c` surfaced in the overview.** These are display-only in this template — they appear as metadata labels. They are not used for retrieval filtering here (filtering happens in the other two templates).

5. **No fabrication rule is explicit in the prompt.** The system instructions also enforce this, but the template-level rule is belt-and-suspenders for cases where the template might be called outside the agent.

### Portable vs. must change

| Element | Portable? | Notes |
|---|---|---|
| Two-section structure | Mostly | The structure works for any service console use case. Section labels and field list will need updating for non-Case objects. |
| HTML output format | Yes, if same surface | Change to markdown/plain text for Slack or non-HTML panels |
| Sensitivity flag | Change | Tie to the customer's actual field name and value set |
| Customer Type / Country in overview | Change field names | The display logic is portable; the field API names are not |
| No-fabrication rule | Portable as-is | |
| Requestor History section | Portable | Change the flow that populates it if the "requestor" concept maps differently (e.g., Account instead of Contact) |

---

## Template 2: [COMPANY]_Console_Draft_Response

### Purpose

Composes a professional email response for the rep to send to the case requestor. The template selects one of five embedded structural email formats based on Customer Type and case subject, then grounds the factual body using Knowledge Articles retrieved from Data Cloud via semantic search. The KA retriever is pre-filtered by the case's country field to scope results to the relevant market.

This template does the heaviest lifting of the three: it runs a retriever, performs template selection logic, and renders a complete send-ready email. The agent script does not modify the output at all (verbatim pass-through + one fixed footer line).

### Inputs

| Input name | Type | Required | Source in agent | Parametrization note |
|---|---|---|---|---|
| `Input:Case` | SOBJECT (Case) | Yes | `record_id` variable | Required for retriever searchText and field references |
| `Input:CustomerType` | String | No | `customer_type` variable | Drives template selection. Defaults to `"Unknown"` if empty. |
| `Input:CaseSnapshot` | String | No | `snapshot` variable (GetRecordDetails output) | Passed as supplemental context. The SOBJECT input already provides structured fields; the snapshot adds recent activity and email history. |
| `outputLanguage` | String | No | Not set in RH | Wire if multi-locale output needed |
| `citationMode` | String | No | Not set in RH | Set if you want KA citations surfaced in the output |
| `isPreviewOnly` | Boolean | No | Not set | Testing only |

### Model

`sfdc_ai__DefaultGPT41`

### Retriever wiring

| Parameter | Value (RH) | Template |
|---|---|---|
| Retriever API name | `Knowledge_kav_Home_Retriever_1Cx_wg12e2f016b` | `[COMPANY]_KA_Retriever` |
| Pre-filter placeholder | `placeholder_RHCaseCountry` | `placeholder_[COMPANY]CaseCountry` |
| Pre-filter binding | `= Case.RH_Case_Country__c` | `= Case.[COMPANY]_Case_Country__c` |
| Search text | `Case.Description + Case.Subject` | Same |

The pre-filter limits KA retrieval to articles tagged for the case's country. If the customer does not segment KAs by country, remove the pre-filter and the retriever will return results from the full KA corpus.

**Important:** The live RH description note says "No email template retriever required." The five structural email templates are embedded directly in the prompt body, not retrieved from a Data Cloud object. This was an intentional design decision: embedding the templates avoids retriever latency for a small, stable set of structures, and gives the prompt author direct control over tone and formatting without Data Cloud indexing dependencies.

### Full Prompt Text (parametrized)

```
You are an email drafting assistant for [COMPANY_LABEL] [ROLE_LABEL].

Draft a professional email response for the [ROLE_LABEL] to send to the requestor for this case.
The email must be grounded in the Knowledge Articles retrieved below. Do not add factual content
that is not present in the case data or the retrieved articles.

---

CASE DATA:
{!Input:Case}

CASE SNAPSHOT (recent activity and history):
{!Input:CaseSnapshot}

CUSTOMER TYPE: {!Input:CustomerType}

RETRIEVED KNOWLEDGE ARTICLES:
{!retriever::[COMPANY]_KA_Retriever}

---

STEP 1 — SELECT EMAIL STRUCTURE

Choose one of the five structural templates below based on Customer Type and case subject.
Do not mix structures. Copy the selected structure's headers, greeting, and closing format exactly,
then fill in the content from the case data and Knowledge Articles.

STRUCTURE A — Candidate, general inquiry:
  Subject: Re: {Case.Subject}
  Greeting: Dear {Case.Contact.FirstName},
  Opening: Thank you for reaching out to [COMPANY_LABEL].
  Body: [2–3 paragraphs grounded in KA content. Factual, direct, friendly.]
  Closing: If you have further questions, please do not hesitate to contact us.
  Sign-off: Warm regards,\n[ROLE_LABEL] Team\n[COMPANY_LABEL]

STRUCTURE B — Candidate, time-sensitive or placement-related:
  Subject: Re: {Case.Subject} — Follow-up
  Greeting: Hi {Case.Contact.FirstName},
  Opening: I wanted to follow up on your recent enquiry.
  Body: [2–3 paragraphs. Address urgency. Ground in KA content.]
  Closing: We are actively working on this and will keep you updated.
  Sign-off: Best regards,\n[ROLE_LABEL] Team\n[COMPANY_LABEL]

STRUCTURE C — Client, account or billing inquiry:
  Subject: Re: {Case.Subject}
  Greeting: Dear {Case.Account.Name} Team,
  Opening: Thank you for contacting [COMPANY_LABEL] regarding {Case.Subject}.
  Body: [2–3 paragraphs. Professional, account-relationship tone. Ground in KA content.]
  Closing: Please let us know if you need any further clarification.
  Sign-off: Kind regards,\n[ROLE_LABEL] Team\n[COMPANY_LABEL]

STRUCTURE D — Third Party:
  Subject: Re: {Case.Subject}
  Greeting: Dear [contact name or "Sir/Madam" if unknown],
  Opening: Thank you for your correspondence regarding {Case.Subject}.
  Body: [2–3 paragraphs. Formal. Ground in KA content.]
  Closing: For further assistance, please contact us directly.
  Sign-off: Yours sincerely,\n[ROLE_LABEL] Team\n[COMPANY_LABEL]

STRUCTURE E — Unknown customer type or fallback:
  Subject: Re: {Case.Subject}
  Greeting: Dear [contact name or "Valued Customer"],
  Opening: Thank you for contacting [COMPANY_LABEL].
  Body: [2–3 paragraphs. Neutral, professional. Ground in KA content.]
  Closing: We look forward to resolving your enquiry promptly.
  Sign-off: Kind regards,\n[ROLE_LABEL] Team\n[COMPANY_LABEL]

STEP 2 — DRAFT THE EMAIL

Using the selected structure:
1. Fill the Subject line with the actual Case Subject.
2. Fill the greeting with the requestor's first name if known; otherwise use the appropriate fallback.
3. Write the body paragraphs. Each factual claim must trace to the retrieved Knowledge Articles
   or the case data. Do not invent product names, process steps, timelines, or policy details.
4. Include the closing and sign-off verbatim from the selected structure, replacing tokens.
5. Do not add headers, section labels, or any text outside the email.
6. Do not include "Draft:" or any prefix before the subject line.

Rules:
- Grounding is mandatory. If the retrieved Knowledge Articles do not contain content relevant to
  the case subject, write the body using only the case data and say "Our team will follow up with
  more details." Do not fabricate policy or product information.
- Keep the email to 150–250 words unless the KA content requires more.
- Match the tone of the selected structure throughout. Do not mix formal and informal registers.
- Do not mention that this email was AI-generated.
```

### Key decisions embedded in the prompt

1. **Five embedded structural templates, not retrieved.** The five structures are hardcoded in the prompt. This is a deliberate design: the structural set is small and stable, and direct control over tone/format outweighs the flexibility of a retrieval-based approach. The trade-off is that changing a template structure requires a prompt update rather than a Data Cloud record update.

2. **Customer Type drives structure selection, not content.** Structure selection is purely presentational (tone, register, greeting). Factual content is always KA-grounded regardless of structure.

3. **Country-scoped KA retrieval.** The `placeholder_RHCaseCountry` pre-filter scopes the retriever to KAs tagged for the case's country. This is critical in a staffing context where employment law and onboarding procedures differ by jurisdiction. For customers without country-level KA segmentation, remove the pre-filter.

4. **No fabrication rule is enforced with a graceful fallback.** Rather than returning an error when KAs are sparse, the prompt instructs the model to acknowledge the gap ("Our team will follow up") and not invent content. This is a higher-fidelity failure mode than a hallucinated response.

5. **`Input:CaseSnapshot` supplements `Input:Case`.** The SOBJECT input provides structured fields. The snapshot (from `GetRecordDetails`) adds email history, recent activity, and any related entity data that the SOBJECT resolver may not include. Both are passed because the retriever's searchText uses `Case.Description + Case.Subject` (from the SOBJECT), while the snapshot helps the LLM understand conversation history without a separate retrieval step.

### Portable vs. must change

| Element | Portable? | Notes |
|---|---|---|
| Five structural templates | Change | The Candidate/Client/Third Party split is staffing-specific. For other industries, redesign the structure set around the customer's contact segments (e.g., Consumer/Business/Partner for a B2B2C model). |
| KA retriever wiring | Portable | The pattern of a country-scoped KA retriever is portable to any org with KA content. Change the pre-filter field name. Remove the pre-filter if no geographic KA segmentation exists. |
| No-fabrication graceful fallback | Portable as-is | |
| Email word count guidance (150–250) | Adjust | Lengthen for technical support orgs; shorten for high-volume transactional service. |
| Sign-off format | Change | Replace "[ROLE_LABEL] Team / [COMPANY_LABEL]" with the customer's actual signature format. |
| "Do not mention AI-generated" rule | Portable | Carry this to all implementations. |

---

## Template 3: [COMPANY]_Console_Similar_Cases

### Purpose

Finds semantically similar resolved Cases using a Data Cloud vector retriever, pre-filtered by customer type. Returns a formatted set of HTML cards — one card per similar case — each showing case number, subject, resolution summary, and inferred resolution pattern. Helps reps understand how cases like this one were closed in the past without searching manually.

### Inputs

| Input name | Type | Required | Source in agent | Parametrization note |
|---|---|---|---|---|
| `Input:Case` | SOBJECT (Case) | Yes | `record_id` variable | Required for retriever searchText |
| `Input:CustomerType` | String | No | `customer_type` variable | Drives retriever pre-filter. Defaults to `"Unknown"` if empty — retriever returns results without customer-type scoping in that case. |
| `outputLanguage` | String | No | Not set in RH | Wire if multi-locale needed |
| `citationMode` | String | No | Not set in RH | Set to enable citation links back to source cases |
| `isPreviewOnly` | Boolean | No | Not set | Testing only |

### Model

`sfdc_ai__DefaultGPT41`

### Retriever wiring

| Parameter | Value (RH) | Template |
|---|---|---|
| Retriever API name | `Case_Retriever_1Cx_wg1451c3a99` | `[COMPANY]_Case_Retriever` |
| Pre-filter placeholder | `placeholder_RH_Customer_Type` | `placeholder_[COMPANY]_Customer_Type` |
| Pre-filter binding | `= CustomerType` (from Input:CustomerType) | `= Input:CustomerType` |
| Search text | `Case.Description + Case.Subject` | Same |
| Source DMO | Case object (resolved cases only — scope enforced by retriever filter or by indexing only closed cases) | [VERIFY — confirm whether the retriever DMO is filtered to `Status = 'Closed'` at index time or via a runtime filter] |

**Vector index note:** The Case retriever requires a Data Cloud DMO with vector embeddings built on the Case Description and Subject. The index must be populated with resolved (closed) cases before the template produces useful output. This is the highest-lead-time prerequisite in the implementation — allow time for initial index build and for enough case history to accumulate before the feature is meaningful.

### Full Prompt Text (parametrized)

```
You are a case resolution assistant for [COMPANY_LABEL] [ROLE_LABEL].

Find semantically similar resolved cases to help the [ROLE_LABEL] understand how this type of
issue has been handled before. Surface resolution patterns, not just case metadata.

---

CURRENT CASE:
{!Input:Case}

CUSTOMER TYPE: {!Input:CustomerType}

RETRIEVED SIMILAR CASES:
{!retriever::[COMPANY]_Case_Retriever}

---

OUTPUT REQUIREMENTS

Format each retrieved case as an HTML card. Render all cards in sequence with no surrounding wrapper.

For each case:
<div style="border:1px solid #ccc; border-radius:4px; padding:12px; margin-bottom:12px;">
  <h4>{Case Number} — {Subject}</h4>
  <p><strong>Customer Type:</strong> {customer type from retrieved case}</p>
  <p><strong>Closed:</strong> {close date}</p>
  <p><strong>Resolution:</strong> {content from [COMPANY]_Case_Close_Summary__c or equivalent
     close summary field. If blank, write "Resolution summary not available."}</p>
  <p><strong>Pattern:</strong> {1–2 sentence inference of the resolution pattern —
     e.g., "Resolved by escalating to payroll operations team after initial KA response
     did not address the specific pay period discrepancy."}</p>
</div>

After the cards, add one section:

<h4>Resolution Patterns Summary</h4>
<p>[2–4 sentences synthesizing the common resolution approaches across the retrieved cases.
   What did these cases have in common? What actions typically resolved them?
   Be specific — name the teams, processes, or KAs if they recur.]</p>

Rules:
- Only use data from the retrieved cases. Do not infer details not present in the retrieval results.
- If fewer than 3 cases are retrieved, note: "Fewer than 3 similar cases found. The index may not
  yet contain sufficient matching records."
- If no cases are retrieved, return only: "No similar cases found for this case type and customer
  segment. This may be a new issue type."
- Do not include the current case in the output.
- Do not add a closing offer or follow-up suggestion.
```

### Key decisions embedded in the prompt

1. **Customer type pre-filter on the retriever.** The pre-filter ensures a Candidate case only returns Candidate case matches, and a Client case returns Client matches. Without this, the resolution patterns surfaced may be irrelevant — an employment contract dispute (Client) and a placement delay complaint (Candidate) have similar language but entirely different resolution paths. The pre-filter is the most important customization point for non-staffing deployments.

2. **`[COMPANY]_Case_Close_Summary__c` as the resolution source.** The live org uses `SDO_Service_Close_Summary__c` which is an SDO-specific field. This is the highest-risk portability gap in the entire pattern. For a new customer, identify which field captures the resolution summary on closed cases — it may be `Internal_Comments`, a custom `Resolution__c`, or a Flow-written summary field. If the field is blank on most historical cases, the cards will produce low-value output and the feature will not be useful until the field is populated retroactively or going forward.

3. **Pattern synthesis section.** In addition to individual cards, the template synthesizes common patterns across the retrieved cases. This is the feature that actually helps a rep — seeing "these 4 cases were all resolved by the payroll ops team via a manual adjustment" is more actionable than four separate cards.

4. **Graceful degradation for sparse results.** The template has two explicit fallback paths: fewer than 3 results (surface with warning), and zero results (return a clean no-match message). The agent script also has a fallback at the subagent level: "No similar cases were found. This may be a new issue type, or the index may not yet contain matching records."

5. **Retrieved cases must be from closed/resolved Cases.** This is enforced at the retriever level, not in the prompt. The prompt assumes all retrieved records are resolved. [VERIFY — confirm whether the Case DMO in Data Cloud is filtered to closed cases at index time, or whether a runtime filter on the retriever handles this.]

### Portable vs. must change

| Element | Portable? | Notes |
|---|---|---|
| HTML card structure | Portable | Update field labels if the object changes |
| Customer type pre-filter | Change field name | The filter concept is portable; the field name and value set are not |
| Close summary field | Change | `SDO_Service_Close_Summary__c` must be replaced with the customer's actual close summary field |
| Pattern synthesis section | Portable as-is | The most valuable part of the output for reps |
| Graceful degradation messages | Portable | Adjust wording to match the customer's brand voice |
| "No current case in output" rule | Portable | Carry to all implementations |
| Vector index on Case DMO | Required build | Allow 1–2 sprint lead time for Data Cloud setup + index build |

---

## Cross-Template Notes

### Model consistency

All three templates use `sfdc_ai__DefaultGPT41`. This is an org-default alias that resolves to the current GPT-4.1 model in Einstein Gateway. If the customer org uses a different default alias, update all three templates consistently. Do not mix models across templates in the same agent — response quality and format consistency depend on using the same underlying model.

### Verbatim pass-through contract

All three subagents in the agent script enforce a "verbatim pass-through" output contract: the LLM output from the prompt template is emitted to the chat panel without modification. This means the formatting responsibility sits entirely with the prompt template, not with the agent orchestration layer. Any formatting bugs (wrong HTML, missing section) must be fixed in the template, not in the agent script.

The Draft Response subagent is the one exception: it appends one fixed line after the verbatim draft ("To adjust the draft, describe the change you need and I will generate a revised version."). This line is hardcoded in the agent script's reasoning instructions, not in the prompt template.

### Retriever pre-filters are load-bearing

Both retrievers (KA and Case) use pre-filters that scope the semantic search before the vector similarity is computed. Removing a pre-filter does not break the agent — it degrades result relevance. The customer type pre-filter on the Case retriever is the most critical: without it, a Candidate-type Case will return Client-type matches and the pattern synthesis will be misleading.

### `SDO_Service_Close_Summary__c` resolution — action required

This field appears in the Similar Cases prompt template as the source of resolution detail on retrieved cases. `SDO_Service_Close_Summary__c` is not a standard Salesforce field — it is specific to the SDO (Salesforce Demo Org) environment where the RH Console Agent was built. Before deploying this pattern to a production customer org:

1. Confirm whether the customer org has a close summary field on Case. Common candidates: `Internal_Comments`, `Resolution__c`, `Close_Summary__c`, a custom field populated by a post-close Flow.
2. If no field exists, either (a) add one and build a Flow to populate it at case close, or (b) modify the prompt to use the last case comment or the case description as a proxy — with the understanding that result quality will be lower.
3. Update the prompt template to reference the correct field API name everywhere `SDO_Service_Close_Summary__c` appears.
