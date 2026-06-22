# Data Cloud Setup Guide: [COMPANY] Service Console Agent

Parametrized from the Robert Half Console Agent implementation (rhDemo org, June 2026).
Replace all `[COMPANY]` placeholders with the customer's namespace prefix before executing.

**Time estimate:** 3–5 hours for a clean org with Data Cloud provisioned and Knowledge Articles already populated.

**Prerequisites before starting:**
- Data Cloud provisioned and activated in the org
- Knowledge Articles published (minimum 20–30 recommended)
- Custom Case fields created: `[COMPANY]_Case_Country__c`, `[COMPANY]_Customer_Type__c`, `[COMPANY]_Case_Sensitivity__c`
- Closed Cases in the org with the `[COMPANY]_Customer_Type__c` field populated

---

## Step 1 — Verify Data Streams are Active

The four Data Streams this solution depends on are Salesforce CRM streams — they are auto-created when Data Cloud connects to the org. Verify they are ACTIVE before proceeding.

**Navigate to:** App Launcher → Data Cloud → Data Streams

Check for ACTIVE status on all four:
- `Knowledge_kav_Home` (Knowledge Articles)
- `Case_Home` (Cases)
- `Contact_Home` (Contacts)
- `Knowledge_DataCategorySelection_Home` (Knowledge data category assignments)

**If any are missing:** Click New → Salesforce Objects → search for the object → enable it. Allow 10–30 minutes for initial data sync.

**If `Knowledge_DataCategorySelection_Home` is absent:** This stream is required only if using the country-based data category pre-filter. Skip this stream if using a simpler retriever with no pre-filter.

**Parametrization note:** Stream names are Salesforce-generated and should match exactly. Do not rename them — the DLO names are derived from the stream names.

---

## Step 2 — Verify Data Lake Objects (DLOs)

DLOs are automatically created from the Data Streams. Once streams are ACTIVE, DLOs appear in the Data Lake Objects section.

**Navigate to:** Data Cloud → Data Lake Objects

Confirm these DLOs exist:
- `Knowledge_kav_Home__dll`
- `Case_Home__dll`
- `Contact_Home__dll`
- `Knowledge_DataCategorySelection_Home__dll` (if using category pre-filter)

**Critical check on Case DLO:** Confirm that `[COMPANY]_Customer_Type__c` and `[COMPANY]_Case_Country__c` are present as fields in `Case_Home__dll`. If the custom fields were added after the stream was first synced, the DLO may not include them. Fix: in the Data Stream, click Refresh Schema → sync.

**Critical check on the chunking field:** Confirm `SDO_Service_Close_Summary__c` is present in `Case_Home__dll`. If absent, identify which field holds the case resolution summary in this org and note it — you'll use it in Step 3.

---

## Step 3 — Configure Data Category Group (Knowledge pre-filter only)

Skip this step if the customer does not need country-scoped Knowledge Article retrieval.

The RH implementation uses Salesforce Data Categories to scope Knowledge Articles by country (US, UK, CA). The retriever pre-filter maps `DataCategoryName` to the value in `[COMPANY]_Case_Country__c`.

**Setup → Data Categories:**
1. Create a Data Category Group named `[COMPANY]_Country` (or equivalent)
2. Add categories matching the values in `[COMPANY]_Case_Country__c`:
   - If `RH_Case_Country__c` uses `United States`, `United Kingdom`, `Canada` — create those exact values
   - Values must match exactly — the pre-filter is case-sensitive
3. Assign the Data Category Group to the Knowledge object
4. Assign articles to the appropriate categories

**Verify:** A published Knowledge Article should show a data category assignment. If the `Knowledge_DataCategorySelection_Home` stream is active, the category assignments will appear in Data Cloud within the next sync window.

**If the customer doesn't use Data Categories:** Remove the pre-filter from the Knowledge retriever in Step 5. The retriever will return all Knowledge Articles regardless of country, which is acceptable for single-market deployments.

---

## Step 4 — Create Search Indexes

Search Indexes must be created manually in the Data Cloud UI. They are not deployable via Metadata API and must be recreated in every target environment.

**Navigate to:** Data Cloud → Search Indexes → New

### Knowledge Article Index

| Field | Value |
|---|---|
| Name | `[COMPANY]_KA_Semantic_Index` |
| Source | `Knowledge_kav_Home__dll` |
| Chunking Field | `ArticleBody` (or the equivalent rich-text body field — verify API name in org) |
| Chunk Size | ~512 tokens (recommended for KA content) |
| Index Type | Semantic |

**After creating:** Click Index Now to trigger the initial indexing run. Allow 15–60 minutes depending on article count. Check status until `Indexing Complete`.

**Known issue:** Incremental indexing has a gap — Knowledge Article updates may not sync immediately. Schedule periodic full re-indexes until Salesforce Engineering resolves. (Documented: Doc 08, Engineering known issue #2)

### Case Index

| Field | Value |
|---|---|
| Name | `[COMPANY]_Case_Semantic_Index` |
| Source | `Case_Home__dll` |
| Chunking Field | `SDO_Service_Close_Summary__c` (if present) OR `Description` |
| Chunk Size | ~512 tokens |
| Index Type | Semantic |

**IMPORTANT — chunking field selection:** Do NOT use `Id`, `CaseNumber`, or any ID-type field as the chunking field. Chunking on ID fields is syntactically valid but produces semantically meaningless vector embeddings — the retriever will return technically valid results that are useless for similarity matching. This is a known silent failure mode. (Documented: `architecture/data-cloud-operational-gotchas.md`)

**After creating:** Click Index Now. Only closed Cases will produce useful similarity results — confirm the Case stream filters to `Status = Closed` before indexing (or add a DLO filter on the index).

---

## Step 5 — Configure Retrievers in Prompt Builder

Retrievers are configured within each Prompt Template in Prompt Builder. The retriever API name is org-specific — it will be different in every org and must be updated in the Prompt Template after each deployment.

**Navigate to:** Setup → Prompt Builder → [COMPANY]_Console_Draft_Response

### Knowledge Article Retriever (for Draft Response template)

1. In the template, locate the `{!$EinsteinSearch:...}` merge field in the prompt body
2. Click the merge field → Configure Retriever
3. Select: `[COMPANY]_KA_Semantic_Index`
4. **Search text:** `{!$Input:Case.Description}{!$Input:Case.Subject}`
5. **Pre-filter (if using country scoping):**
   - Click Add Pre-filter
   - Filter field: `DataCategoryName`
   - Value source: `{!$Input:Case.[COMPANY]_Case_Country__c}`
   - Placeholder name: `placeholder_[COMPANY]CaseCountry`
6. **Output fields:** `Chunk`
7. Save and publish the template version

**Verify:** Use Preview in Prompt Builder with a sample Case record. Confirm that Knowledge Articles are returned and that the country filter is scoping correctly.

### Case Retriever (for Similar Cases template)

1. Open [COMPANY]_Console_Similar_Cases in Prompt Builder
2. Configure the retriever merge field
3. Select: `[COMPANY]_Case_Semantic_Index`
4. **Search text:** `{!$Input:Case.Description}{!$Input:Case.Subject}`
5. **Pre-filter:**
   - Filter field: `[COMPANY]_Customer_Type__c`
   - Value source: `{!$Input:CustomerType}` (passed from agent variable)
   - Placeholder name: `placeholder_[COMPANY]_Customer_Type`
6. **Output fields:** `Chunk`, `Id`, `CaseNumber`, `Subject`, `Type`, `SDO_Service_Close_Summary__c`
   - **REQUIRED:** Include `Id` — the template renders clickable case links using the record Id. If Id is not in the output fields, links will not render.
7. Save and publish

---

## Step 6 — Deploy Prompt Templates and Agent via Metadata API

After completing Steps 1–5, deploy the Salesforce metadata components:

```bash
# From the project directory
sf project deploy start --target-org [TARGET_ORG_ALIAS] \
  --metadata "Bot:[COMPANY]_Console_Agent" \
  --metadata "GenAiPromptTemplate:[COMPANY]_Console_Case_Summary" \
  --metadata "GenAiPromptTemplate:[COMPANY]_Console_Draft_Response" \
  --metadata "GenAiPromptTemplate:[COMPANY]_Console_Similar_Cases" \
  --metadata "GenAiPlanner:[COMPANY]_Console_Agent_v2"
```

**Post-deploy required steps (manual, cannot be deployed):**
1. Update retriever references in each Prompt Template — the retriever API name changes per org
2. Activate the agent in Agentforce Builder
3. Add the agent component to the Case record page in Lightning App Builder
4. Assign the Agentforce permission set to CSM users

---

## Step 7 — Test End-to-End

Minimum test set before going live:

| Test | Expected result |
|---|---|
| Open agent on a Case record | Welcome message appears; agent identifies Case and CSM name |
| "Summarize this case" | Two-section HTML summary with Case Overview and Requestor History |
| "Summarize this case" (same record, second turn) | Returns `last_generated_summary` immediately — no re-fetch |
| "Find similar cases" | HTML cards with resolved cases; links are clickable |
| "Draft a response" | Email appears with correct template (A–E) selected; body grounded on KAs |
| Navigate to different Case, then ask "summarize" | New summary generated (not the previous case's) |
| Navigate to a non-Case record | Agent states it can only assist on Case records |
| "Summarize this case" with no Knowledge Articles indexed | Draft Response fallback message appears; no hallucinated content |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Retriever returns no results | Index not complete or retriever not wired | Check Search Index status; re-run Index Now; verify retriever merge field in Prompt Builder |
| Similar cases show wrong customer type | Pre-filter placeholder not wired | Check `placeholder_[COMPANY]_Customer_Type` value expression in retriever config |
| Draft Response ignores Knowledge Articles | Wrong chunking field selected on index | Rebuild index with correct text/body field; verify field contains content |
| Country pre-filter returns empty | Data category values don't match `[COMPANY]_Case_Country__c` values | Check exact string match; values are case-sensitive |
| Agent shows wrong case after navigation | `last_record_id` sync logic missing from a subagent | Check all subagents have the record-change detection block in `before_reasoning` |
| `SDO_Service_Close_Summary__c` is blank in results | Field not present or not mapped in DLO | Verify field exists in org; update chunking field to `Description` as fallback |
