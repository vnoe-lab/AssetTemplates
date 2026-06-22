## Permissions Required: [COMPANY] Service Console Agent

---

### 1. Custom Object / Field Permissions

All three custom Case fields were retrieved directly from metadata. The Knowledge__kav object has no RH-prefixed custom fields — the retriever pre-filter operates on Data Category assignment, not a custom field.

#### Case Object — Custom Fields

| API Name | Field Type | Purpose | CSM (Human User) | Agent Bot User |
|---|---|---|---|---|
| `RH_Case_Country__c` | Text(100) | Stores mapped country (US / UK / Canada / All Regions) used as retriever pre-filter in Draft Response and Sensitivity Classifier. Stamped by `RH_Case_Sensitivity_Classifier_and_Router` flow. | Read + Write | Read + Write |
| `RH_Case_Sensitivity__c` | Picklist (Unclassified / Standard / Sensitive) | Classification flag stamped by the Sensitivity Classifier flow. Drives routing: Sensitive cases skip the agent and go to human review queue. | Read | Read + Write |
| `RH_Customer_Type__c` | Picklist (Unknown / [INDUSTRY_TERM_CANDIDATE] / [INDUSTRY_TERM_CLIENT]) | Persona classification. Used as retriever pre-filter in Similar Cases and Draft Response prompt templates. Derived via `RH_Console_Derive_Customer_Type` flow. | Read | Read |

**Source:** Verified from `RH_Demo_Fields` permission set metadata and direct field-meta.xml retrieval.

**Notes:**
- The agent bot user requires Write on `RH_Case_Country__c` and `RH_Case_Sensitivity__c` because those are stamped by flows that run in the agent's execution context (`RH_Case_Sensitivity_Classifier_and_Router` performs `recordUpdates` on both fields).
- `RH_Customer_Type__c` is read-only for the bot user because `RH_Console_Derive_Customer_Type` only reads it and returns a mapped string — it never writes back.
- The `RH_Service_Case` record type visibility must be granted in any permission set assigned to CSMs. Verified from `RH_Demo_Fields`.

#### Case Object — Standard Fields Required by Flows

The following standard Case fields are queried by retrieved flows. These are standard fields on the Case object and typically accessible by default, but should be confirmed for any restricted profile:

| Field | Accessed By | Bot CRUD |
|---|---|---|
| `ContactId` | `RH_Console_Get_Requestor_Cases`, `RH_Sensitive_Redirect_Wrapper` | Read |
| `CaseNumber` | `RH_Console_Get_Requestor_Cases`, `RH_Sensitive_Redirect_Wrapper` | Read |
| `Subject` | Prompt templates (searchText), `RH_Console_Get_Requestor_Cases` | Read |
| `Description` | Prompt templates (searchText), `RH_Console_Get_Requestor_Cases` | Read |
| `Status` | `RH_Case_Sensitivity_Classifier_and_Router` (sets to "Waiting on Customer") | Read + Write |
| `OwnerId` | `RH_Case_Sensitivity_Classifier_and_Router` (routes to queue) | Read + Write |
| `Type` | `RH_Console_Get_Requestor_Cases` (displayed in formatted output) | Read |
| `CreatedDate` | `RH_Console_Get_Requestor_Cases` (sort order) | Read |
| `SDO_Service_Close_Summary__c` | `RH_Console_Similar_Cases` prompt template (resolution pattern extraction) | Read |

**Note on `SDO_Service_Close_Summary__c`:** This field appears in the Similar Cases prompt template content. It is not an RH-prefixed field and was not in the retrieved `RH_Demo_Fields` permission set. It must be confirmed as either a standard SDO field already accessible or added explicitly. **Inferred — not verified from a dedicated field-meta.xml retrieve.**

#### Contact Object — Standard Fields Required by Flows

| Field | Accessed By | Bot CRUD |
|---|---|---|
| `FirstName` | `RH_Sensitive_Redirect_Wrapper`, `RH_Case_Sensitivity_Classifier_and_Router` | Read |
| `LastName` | `RH_Sensitive_Redirect_Wrapper`, `RH_Case_Sensitivity_Classifier_and_Router` | Read |

#### Knowledge__kav — Custom Fields

No RH-prefixed custom fields exist on Knowledge__kav. Verified by direct retrieve — the 17 custom fields on that object are all generic SDO fields. The agent's knowledge grounding operates entirely through Data Cloud retriever pre-filters on Data Category Groups (Country, Audience, Inquiry_Type), not custom field filters on the KAV object itself.

---

### 2. Permission Set Requirements

The following permission sets were retrieved directly from the org. All are verified unless noted.

#### RH-Specific Permission Sets

| Permission Set | Purpose | Assign To |
|---|---|---|
| `RH_Demo_Fields` | Grants Read+Write on `RH_Case_Country__c`, `RH_Case_Sensitivity__c`, `RH_Customer_Type__c` and visibility on `RH_Service_Case` record type. The primary custom-field access grant for this build. | CSM users (human agents), Admin users, Agent bot user |
| `RH_Service_Agent1337078425_Permissions` | Auto-generated Einstein Agent PSL for bot user `rh_service_agent@00daj00000vewg1610647096.ext`. License-only, no explicit object/field permissions. | Bot user (auto-assigned on agent activation) |
| `RH_Service_Agent409558732_Permissions` | Auto-generated Einstein Agent PSL for bot user `rh_service_agent@00daj00000vewg1225569082.ext`. Same pattern. | Bot user (auto-assigned) |
| `RH_Service_Agent984188567_Permissions` | Auto-generated Einstein Agent PSL for bot user `rh_service_agent@00daj00000vewg12118367366.ext`. Same pattern. | Bot user (auto-assigned) |
| `Robert_Half_Service_Agent1225406921_Permissions` | Auto-generated Einstein Agent PSL for bot user `robert_half_service_agent@...`. Corresponds to an earlier bot user named Robert_Half_Service_Agent rather than RH_Service_Agent. | Bot user (auto-assigned) |

**Note on agent PSLs:** These four permission sets carry only `license: Einstein Agent`. The actual object/field access for the bot user comes from whatever additional permission sets are stacked on top. In a new deployment, only one of these will be active; the multiple entries reflect re-activations or sandbox copies in the RH org. A new deployment will generate a new auto-named PSL on first activation.

#### SDO Platform Permission Sets Relevant to This Build

These were retrieved and are relevant to replicating the agent in another org. They are not RH-authored but are required dependencies.

| Permission Set | Purpose | Assign To |
|---|---|---|
| `SDO_Service_AI_Agents_Demo` | Grants CRUD on `AI_Agent__c`, `AI_Agent_Conversation__c`, `AI_Agent_Conversation_Entry__c`; grants access to `AIAgentConversationComparator`, `AIAgentMonitorController`, `AgentListController` Apex classes; grants tab visibility on SemanticModel. Required if the Agent monitoring/comparison UI is used. | Demo admins, SEs running the demo |
| `SDO_Service_Agent_Helper_PSL` | CRUD on `SDO_Service_Einstein_Agent_Helper__c` — a helper object used by the SDO Einstein agent framework. | Bot user (if SDO helper flows are triggered), Admin |
| `SDO_Service_Base_Access` | Baseline Service Cloud object permissions for CSM users. Contains Case, Contact, Account CRUD. Standard dependency for any service console demo. **(Inferred as required — content not read in full during this session.)** | CSM users |

---

### 3. Agent (Bot) User Permissions

The bot user is an auto-provisioned Salesforce user created when the Agentforce agent is activated. It runs all flow actions, prompt template calls, and Data Cloud retriever queries in the agent's execution context.

The following is derived from analysis of prompt template inputs, flow actions, and retriever configurations. Items marked **[Inferred]** were not found in an explicit permission set retrieve but are required for the described operations to succeed.

#### Object-Level Permissions

| Object | Create | Read | Edit | Delete | Notes |
|---|---|---|---|---|---|
| `Case` | No | Yes | Yes | No | Edit required for stamping `RH_Case_Country__c`, `RH_Case_Sensitivity__c`, and `Status` / `OwnerId` (sensitivity routing). Verified from flow actions. |
| `Contact` | No | Yes | No | No | Read only — used for name lookup in sensitivity redirect and requestor case history. Verified from flow actions. |
| `Knowledge__kav` | No | Yes | No | No | Read required for retriever grounding. No write needed. **[Inferred — retriever access is controlled at the Data Cloud/Einstein Search layer, but the underlying KAV object must be readable.]** |
| `Account` | No | Yes | No | No | Standard lookup on Case. **[Inferred — required by GetRecordDetails action used in case_summary and similar_cases topics.]** |

#### Field-Level Permissions (Bot User Minimum Set)

In addition to all standard readable Case fields listed in Section 1:

| Object.Field | Read | Edit | Verified |
|---|---|---|---|
| `Case.RH_Case_Country__c` | Yes | Yes | Verified (flow stamping) |
| `Case.RH_Case_Sensitivity__c` | Yes | Yes | Verified (flow stamping) |
| `Case.RH_Customer_Type__c` | Yes | No | Verified (read-only derivation) |
| `Case.SDO_Service_Close_Summary__c` | Yes | No | Inferred (prompt template input) |
| `Case.Status` | Yes | Yes | Verified (sensitivity routing sets "Waiting on Customer") |
| `Case.OwnerId` | Yes | Yes | Verified (sensitivity routing assigns to queue) |
| `Contact.FirstName` | Yes | No | Verified (flow lookup) |
| `Contact.LastName` | Yes | No | Verified (flow lookup) |

#### Queue Access

The bot user's flow (`RH_Case_Sensitivity_Classifier_and_Router`) routes sensitive cases to queue ID `00Gg700000RYsOHEA1`. The bot user must be able to assign `OwnerId` to a queue record. This is controlled by the `Edit` permission on Case plus the queue being configured to accept Case records. **[Queue ID is org-specific; a new deployment requires a new queue to be created and the queue ID updated in the flow. Verified as hardcoded in the flow — this is a required customization step for any new deployment.]**

---

### 4. Einstein / Agentforce Platform Permissions

#### License Requirements

| License | Purpose | Scope | Notes |
|---|---|---|---|
| Einstein Agent (PSL) | Required for each Agentforce bot user. Auto-assigned via the auto-generated permission sets described in Section 2. | Per-bot-user | Provisioned automatically on agent activation. No manual step required beyond having the license available in the org. |
| Data Cloud (base) | Required for all Data Cloud retriever functionality — the Case retriever and Knowledge retriever both query Data Cloud data lake objects. | Org-wide (Data Cloud must be provisioned) | Verified: Data Cloud is active in the RH org with 54 data streams. A new org must have Data Cloud provisioned before retrievers can be configured. |
| Agentforce for Service | Required to create and activate the `ExternalCopilot` type bot with `Atlas__ConcurrentMultiAgentOrchestration` orchestration. | Per-user (CSMs who interact with the agent panel) | **[Inferred — standard licensing requirement for Agentforce for Service. The RH org uses this license class; the specific SKU was not retrieved as a license object during this session.]** |
| Prompt Builder | Required for admins who author or edit the three prompt templates (`RH_Console_Case_Summary`, `RH_Console_Draft_Response`, `RH_Console_Similar_Cases`). | Per-admin-user | **[Inferred — standard Prompt Builder access requirement. Not retrieved as a distinct permission set in this org.]** |

#### Einstein Retriever / Data Cloud Retriever Permissions

The two retrievers configured in this build (`Knowledge_kav_Home_Retriever_1Cx_wg12e2f016b` for Draft Response, `Case_Retriever_1Cx_wg1451c3a99` for Similar Cases) are Data Cloud Einstein Retrievers. These are configured entirely within Data Cloud setup and are not controlled by standard Salesforce permission sets.

| Requirement | Scope | Notes |
|---|---|---|
| Data Cloud Search Index on `Knowledge__kav` | Org-wide | The index underlying the knowledge retriever must be built in Data Cloud before the prompt template can invoke it. The RH org has the retriever configured; a new deployment requires creating the index from scratch. **[Verified: the retriever API names are hardcoded in the prompt template metadata.]** |
| Data Cloud Search Index on `Case` | Org-wide | Same requirement for the case similarity retriever. The Case index must include `Description`, `Subject`, and `SDO_Service_Close_Summary__c` fields to support the semantic search used in Similar Cases. **[Verified: retriever name hardcoded in prompt template; field coverage inferred from prompt template content.]** |
| Data Cloud Data Space access | Per-user (Einstein Retriever queries run in `default` data space) | **[Inferred — standard Data Cloud permission model. The `SDO_Data_Cloud_Default_Data_Space` permission set was observed in the org and may be required for users who administer retrievers.]** |

---

### 5. Setup Requirements (Admin-Only)

These are org-level toggles and configuration steps that must be enabled in Setup before the agent can function. Items are ordered by dependency.

| # | Requirement | Setup Location | Verified / Inferred |
|---|---|---|---|
| 1 | **Einstein generative features enabled** | Setup > Einstein Setup (or Generative AI settings page) — must be turned on for the org before any prompt templates or agents are created. | Inferred — standard prerequisite. |
| 2 | **Agentforce enabled** | Setup > Agents — the Agents menu item must be present, which requires Agentforce licensing. The bot type `ExternalCopilot` will not appear until this is active. | Inferred. |
| 3 | **Data Cloud provisioned and connected** | Setup > Data Cloud Setup — Data Cloud must be provisioned and the org-to-org connector to the CRM org established. Required before any retriever can be created. | Verified — Data Cloud is active in RH org with confirmed data streams. |
| 4 | **Data Category Groups created** (`Country`, `Audience`, `Inquiry_Type`) | Setup > Data Category Setup — the four Data Category Groups retrieved (`Country`, `Audience`, `Inquiry_Type`, `Product`) must exist before Knowledge Articles can be categorized for retriever pre-filtering. The `Country` group (All / United States / United Kingdom / Canada / All Regions) is the one actively used by the retriever pre-filter. | Verified — all four groups retrieved. |
| 5 | **Data Category visibility rules configured for Knowledge** | Setup > Data Category Visibility / Customer Portal Settings — roles and profiles must have visibility into the relevant Country data categories for the retriever to surface articles. | Inferred — required by standard Knowledge + Data Category behavior. |
| 6 | **Case Data Cloud data stream active** | Data Cloud > Data Streams — a CRM data stream ingesting Case records must be running and synced before the Case retriever index can be built. | Verified — `AiAgentInteraction__dll` and related Agentforce telemetry streams are active; CRM Case stream existence inferred from retriever configuration. |
| 7 | **Einstein Retriever indexes built in Data Cloud** — one for `Knowledge__kav`, one for `Case` | Data Cloud > Einstein Retrieval — create Search Indexes, build vector embeddings. The retriever developer names (`Knowledge_kav_Home_Retriever_1Cx_wg12e2f016b`, `Case_Retriever_1Cx_wg1451c3a99`) are org-specific; a new deployment will generate new names that must be updated in the prompt templates. | Verified (retriever names hardcoded in prompt templates); index build step inferred. |
| 8 | **Prompt templates activated** | Setup > Prompt Builder — all three prompt templates (`RH_Console_Case_Summary`, `RH_Console_Draft_Response`, `RH_Console_Similar_Cases`) must be in Active status before the agent can invoke them. Templates in Draft status will cause runtime failures. | Inferred — standard Prompt Builder behavior. |
| 9 | **Sensitivity routing queue created** | Setup > Queues — a Case queue must exist and the queue ID must be updated in `RH_Case_Sensitivity_Classifier_and_Router` (currently hardcoded as `00Gg700000RYsOHEA1`). This queue receives cases classified as Sensitive. | Verified — queue ID is hardcoded in the flow. This is a required customization step for every new deployment. |
| 10 | **Custom fields deployed to target org** | Deployment — `RH_Case_Country__c`, `RH_Case_Sensitivity__c`, `RH_Customer_Type__c` must be deployed to the target org via change set, package, or `sf project deploy start` before any flow, prompt template, or agent can reference them. | Verified — field-meta.xml files retrieved; deployment is a prerequisite. |
| 11 | **`RH_Service_Case` record type created on Case** | Setup > Object Manager > Case > Record Types — the record type must exist in the target org and be included in the `RH_Demo_Fields` permission set assignment. | Verified — record type visibility is in `RH_Demo_Fields`. |
| 12 | **Agent script model override** (v2 only) | Agent Builder — the v2 agent script specifies `model: "model://sfdc_ai__DefaultEinsteinLlmGatewayGPT4Omni"` on the `agent_router` node. This model alias must be available in the target org. If it is not, the agent will fall back to the org default model (v1 behavior). | Verified — model config difference between v1 and v2 confirmed from decoded agent script. |
| 13 | **Service Console app configured with agent panel** | App Builder — the Service Console Lightning app must have the Agentforce agent panel component added and the `RH_Console_Agent_v2` bot selected as the active agent. The agent surface variables (`currentRecordId`, `currentObjectApiName`, etc.) are injected by this component. | Inferred — standard Agentforce for Service console configuration. |

---

**Legend:**
- **Verified** — requirement confirmed directly from retrieved metadata (field-meta.xml, flow-meta.xml, permission set files, agent script, prompt template content, or Tooling API query results).
- **Inferred** — requirement derived from platform behavior, naming conventions, or functional analysis of retrieved artifacts. Should be validated against Salesforce documentation or a test deployment before treating as authoritative.
