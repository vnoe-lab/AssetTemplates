## Flows: Robert Half Service Console Agent

---

### RH_Console_Define_Object_Type

- **Type**: Autolaunched Flow (no trigger)
- **Trigger**: Called by the RH Console Agent at session start, before any action is taken, to validate the CSM is viewing a Case record
- **Purpose**: Resolves a Salesforce record ID prefix to an object type label ("Case" or "Other"), acting as a guard rail so the agent does not attempt case operations on non-Case pages.
- **Inputs**:
  - `varRecordId` (String) — the current record ID injected from the agent variable `currentRecordId`
- **Outputs**:
  - `varObjectType` (String) — "Case" if the ID begins with "500", otherwise "Other"
- **Key Logic**:
  1. Decision: does `varRecordId` start with "500"?
  2. If yes → assign `varObjectType = "Case"`
  3. If no → assign `varObjectType = "Other"`
- **Connected To**: Agent routing logic (likely the first topic invoked after the agent initializes); populates the `object_type` agent variable used by downstream topic conditions
- **Parametrization Notes**:
  - The "500" prefix is the universal Salesforce Case ID prefix — no change needed for any org
  - If the template extends to other objects (e.g., Contact "003", Lead "00Q"), add additional decision branches here
  - The output variable name `varObjectType` must match the agent variable binding (`object_type`) — verify naming convention in the target org's agent variable mapping

---

### RH_Console_Derive_Customer_Type

- **Type**: Autolaunched Flow (no trigger)
- **Trigger**: Called by the Console Agent after object type is confirmed as Case; populates the `customer_type` agent variable used to pre-filter Data Cloud retrievers
- **Purpose**: Reads `RH_Customer_Type__c` on the Case record and maps its picklist value to a normalized string ("Candidate", "Client", or "Unknown") for use as a retriever pre-filter.
- **Inputs**:
  - `varCaseId` (String) — the confirmed Case record ID
- **Outputs**:
  - `varCustomerType` (String) — one of: "Candidate" / "Client" / "Unknown"
- **Key Logic**:
  1. SOQL Get on Case: retrieves `RH_Customer_Type__c`
  2. Decision: is `RH_Customer_Type__c` populated?
  3. If populated → maps picklist value to normalized output; if blank or unrecognized → defaults to "Unknown"
- **Connected To**: Feeds the `customer_type` agent variable consumed by the `RH_Console_Draft_Response` and `RH_Console_Similar_Cases` prompt templates as the retriever pre-filter value
- **Parametrization Notes**:
  - `RH_Customer_Type__c` is a Robert Half custom field — **replace** with the target org's equivalent customer classification field (e.g., `Contact_Type__c`, `Account_Segment__c`)
  - The picklist values "Candidate" / "Client" / "Third Party" / "Unknown" are RH-specific — remap to the target org's taxonomy
  - If the target org stores this on the Contact or Account (not the Case), redirect the SOQL Get accordingly and add a relationship lookup step
  - The retriever pre-filter key in the prompt template must match whatever value this flow returns — keep the two in sync

---

### RH_Console_Get_Requestor_Cases

- **Type**: Autolaunched Flow (no trigger)
- **Trigger**: Called by the Console Agent to populate the `requestor_cases` variable before invoking the `RH_Console_Case_Summary` prompt template
- **Purpose**: Queries the 10 most recent prior cases for the same Contact as the current case (excluding the current case) and returns them as a formatted HTML string ready for injection into the summary prompt.
- **Inputs**:
  - `varCaseId` (String) — the current Case record ID
- **Outputs**:
  - `varCasesFormatted` (String) — HTML-formatted list of prior cases, or a fallback message ("No contact linked" / "No prior cases found")
- **Key Logic**:
  1. SOQL Get on current Case: retrieves `ContactId`
  2. Decision: is `ContactId` populated? If not → output "No contact linked" and exit
  3. SOQL Get on Cases: WHERE ContactId = [value] AND Id != [current case], ORDER BY CreatedDate DESC, LIMIT 10
  4. Loop: for each case record, builds an HTML card (blue left-border inline style) with CaseNumber, Subject, Status, Type, CreatedDate, Description, and a clickable case link
  5. If no prior cases returned → output "No prior cases found"
- **Connected To**: Populates `requestor_cases` agent variable; consumed by `RH_Console_Case_Summary` prompt template as the `RequestorCases` input
- **Parametrization Notes**:
  - The SOQL query field list (CaseNumber, Subject, Status, Type, CreatedDate, Description) is portable — no RH-specific fields required in the query itself
  - The HTML card template with inline styles is hardcoded — acceptable for most Service Console deployments; update brand colors (blue border) to match the target org's style guide
  - The case link URL pattern in the HTML assumes a standard Salesforce record URL — verify the target org's Experience Cloud or console URL structure if cases are linked from a community
  - LIMIT 10 is a business rule choice — make this configurable via a flow variable or custom metadata if the target org has different volume expectations
  - If the target org does not use ContactId on Case (e.g., uses AccountId or a custom lookup), redirect the relationship traversal accordingly

---

### RH_Case_Sensitivity_Classifier_and_Router

- **Type**: Autolaunched Flow (no trigger)
- **Trigger**: Called by the Console Agent (or directly from a topic action) when a new inbound email case is received and before any Knowledge retrieval or draft response is generated
- **Purpose**: Classifies a Case as STANDARD or SENSITIVE using an LLM prompt, stamps the result on the Case record, and — if sensitive — routes the case to a human review queue and generates a holding response; if standard, returns a `STANDARD_PROCEED` signal for the agent to continue.
- **Inputs**:
  - `caseId` (String) — the Case record ID
  - `customerQuestion` (String) — the inbound case description or subject
  - `mappedCountry` (String, also output) — country scope for classifier context
- **Outputs**:
  - `agentResponse` (String) — either "STANDARD_PROCEED" or the LLM-generated redirect response text
  - `mappedCountry` (String) — the resolved country value after mapping
  - `sensitivityResult` (String) — "STANDARD" or "SENSITIVE"
- **Key Logic**:
  1. SOQL Gets: retrieves Case (`RH_Customer_Type__c`, `RH_Case_Country__c`) and associated Contact
  2. Country mapping decision tree: maps Case country field value to one of "US" / "UK" / "Canada" / "All" and stamps `RH_Case_Country__c` on the Case record
  3. LLM action: invokes `RH_Sensitive_Request_Classifier` prompt template; captures result into `sensitivityResult` and stamps `RH_Case_Sensitivity__c` on the Case record
  4. Sensitivity decision: if `sensitivityResult = "SENSITIVE"` → invoke `RH_Sensitive_Request_Redirect` prompt template, reassign Case to queue `00Gg700000RYsOHEA1`, set Case Status to "Waiting on Customer", return redirect text as `agentResponse`; if "STANDARD" → set `agentResponse = "STANDARD_PROCEED"`
- **Connected To**: Called before `RH_Console_Draft_Response` topic; the `STANDARD_PROCEED` return value acts as a gate — the agent proceeds to Knowledge retrieval only when this value is returned. Also feeds `mappedCountry` into the `RH_Case_Country__c` pre-filter used by the Draft Response retriever
- **Parametrization Notes**:
  - `RH_Case_Country__c` — **replace** with target org's country field on Case; if none exists, derive from Contact.MailingCountry or Account.BillingCountry
  - `RH_Customer_Type__c` — **replace** as noted above
  - `RH_Case_Sensitivity__c` — **replace** with target org's sensitivity or escalation flag field; if the target org does not have this classification requirement, this entire flow may be omitted
  - The hardcoded queue ID `00Gg700000RYsOHEA1` is org-specific — **must be replaced** with the target org's human review / escalation queue ID; consider using a Custom Metadata record to store this so it is environment-portable
  - The country mapping logic (US/UK/Canada/All) is RH-specific — remap to the target org's supported locale list, or replace with a dynamic lookup from Custom Metadata
  - The prompt templates `RH_Sensitive_Request_Classifier` and `RH_Sensitive_Request_Redirect` are not part of the Console Agent's three main prompts — they must be built or adapted separately; the classifier prompt needs org-specific sensitivity criteria
  - If the target org does not have a sensitivity classification requirement, this flow (and its wrapper) can be dropped entirely from the template

---

### RH_Sensitive_Redirect_Wrapper

- **Type**: Autolaunched Flow (no trigger)
- **Trigger**: Called directly by an agent action (standalone invocation path, separate from the Classifier flow) when a Case has already been identified as sensitive and a redirect response needs to be generated on demand
- **Purpose**: Standalone wrapper that queries Case and Contact data and invokes the `RH_Sensitive_Request_Redirect` prompt template, returning the generated response text for direct agent delivery — without re-running classification.
- **Inputs**:
  - `CaseId` (String) — the Case record ID
- **Outputs**:
  - `redirectRaw` (String) — raw LLM output from the prompt template
  - `redirectResponse` (String) — same value; second output variable for downstream flexibility
- **Key Logic**:
  1. SOQL Get on Case: retrieves Id, ContactId, CaseNumber, `RH_Case_Country__c`, `RH_Customer_Type__c`
  2. SOQL Get on Contact: retrieves FirstName, LastName
  3. Invokes `RH_Sensitive_Request_Redirect` prompt template with Case + Contact data
  4. Assigns output to both `redirectRaw` and `redirectResponse`
- **Connected To**: Used when the agent needs to surface a sensitive-redirect response without re-running the full classifier; likely called from an explicit "explain why this case was escalated" or "resend redirect message" agent action
- **Parametrization Notes**:
  - Same field replacements as the Classifier flow apply: `RH_Case_Country__c`, `RH_Customer_Type__c`
  - The dual output variables (`redirectRaw` / `redirectResponse`) are redundant — consolidate to one in the template unless the target org has a specific downstream reason to keep both (e.g., one goes to agent variable, one to a record update)
  - If the target org does not implement sensitivity routing, this flow can be omitted entirely

---

## Inferred Flows (Not Directly Retrieved but Architecturally Required)

The following flow behaviors are confirmed by agent variable bindings and prompt template inputs, but their implementation detail was not directly observed. They are included as template stubs.

---

### [INFERRED] Snapshot / Get Record Details Action

- **Type**: Standard Agent Action (GetRecordDetails) or Autolaunched Flow
- **Trigger**: Called by the agent when the `snapshot` variable is empty and a case action is requested
- **Purpose**: Produces a full plain-text or structured summary of the current Case record — including all relevant fields — as a single string for injection into prompt templates that require a case snapshot rather than individual field mappings.
- **Inputs**:
  - `record_id` (String) — current Case ID
- **Outputs**:
  - `snapshot` (String) — full case text snapshot (field labels + values, formatted)
- **Key Logic**: Likely the standard Salesforce "Get Record Details" agent action configured for the Case object; may be a flow if additional field enrichment or formatting is needed
- **Connected To**: Consumed by `RH_Console_Draft_Response` as the `CaseSnapshot` input; also potentially used by the Similar Cases prompt
- **Parametrization Notes**:
  - If using the standard GetRecordDetails action, configure the field list to match the target org's Case layout
  - If using a custom flow, ensure the output format matches what the prompt template expects (plain text vs. HTML vs. JSON)
  - Fields likely included: Subject, Description, Status, Priority, Type, Origin, CreatedDate, all custom classification fields

---

### [INFERRED] Session Initialization / Variable Bootstrap Sequence

- **Type**: Implicit agent topic or entry dialog
- **Trigger**: Agent session start (user opens console, agent initializes)
- **Purpose**: Orchestrates the sequence: inject `currentRecordId` → call Define Object Type → call Derive Customer Type → set `record_id`, `object_type`, `customer_type` agent variables — so all downstream topics have clean inputs before any user interaction.
- **Inputs**: `currentRecordId`, `currentObjectApiName` (injected by the External Copilot framework)
- **Outputs**: `record_id`, `object_type`, `customer_type` populated in agent variable scope
- **Key Logic**: This is likely implemented as an agent dialog or topic that fires automatically on context change (record navigation), not a standalone flow
- **Connected To**: All three main prompt-backed topics depend on these variables being set; this initialization must complete before any topic fires
- **Parametrization Notes**:
  - The `currentRecordId` injection is standard ExternalCopilot behavior — no changes needed
  - If the target org adds additional classification dimensions (e.g., account tier, region), add the corresponding classification flows here and bind their outputs to new agent variables
  - The variable names (`record_id`, `object_type`, `customer_type`) must be consistent across the bot definition, all flows, and all prompt template input bindings
