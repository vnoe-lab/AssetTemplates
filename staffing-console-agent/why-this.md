# Why This: Staffing — Service Console Agent

**Pattern:** Agentforce Internal Copilot on the Service Console Case record page  
**Source engagement:** Robert Half (rhDemo org, June 2026)  
**Prepared by:** Victor Noé, L7 AFSE  

---

## 1. The Customer Pain

Staffing service reps handle a structurally different case load than most service orgs. Their cases are not product defects or billing disputes — they are human situations: a candidate whose paycheck is wrong, a client whose placement fell through, a worker asking about a contract they signed months ago. Each case has a person on the other end who expects the rep to know their history, understand their relationship to the company, and respond with the right tone for the right audience.

In practice, reps spend the first several minutes of every case doing three things manually: reading the case description to orient themselves, searching for similar closed cases to understand how this type of issue was handled before, and opening a blank email and writing a response from scratch (or hunting for the right template in a separate tab). None of this is judgment work. All of it is overhead.

The specific pain this pattern addresses is that overhead — not the complexity of the case, but the ramp time before a rep can start exercising judgment. In a high-volume staffing service org, that overhead compounds across hundreds of reps and thousands of cases per week.

---

## 2. The Recommendation, with Receipts

**Recommendation:** Agentforce Internal Copilot (Service Console Agent) with three declarative subagents — Case Summary with requestor history, Similar Resolved Cases via Data Cloud semantic retrieval, and AI-drafted email responses grounded on Knowledge Articles — deployed on the Case record page via Lightning App Builder.

**Why this and not something else:**

**The deployment surface choice is non-negotiable for rep adoption.** The agent is embedded directly in the Service Console Case record page as a Lightning App Builder component. The rep never leaves the record. Every agent interaction happens in context of the case they are already looking at. The platform injects four variables at session start — current record ID, object type, page type, app name — so the agent reads the case automatically without the rep entering a case number. (Source: agent-definition.md, Variables — Tier 1 External; confirmed in live org bot-meta.xml extraction, June 2026.)

If the agent lived anywhere else — a standalone portal, a separate app, a sidebar that required copy-pasting — rep adoption would be near zero. The friction of context-switching is the primary reason tools like this fail in production.

**Data Cloud retrievers are the right grounding architecture for this use case.** Two retrievers power the two highest-value subagents: a Knowledge Article retriever for the email draft (pre-filtered by case country so a UK case gets UK policy content, not US content), and a Case retriever for similar case search (pre-filtered by customer type so a Candidate case returns Candidate resolutions, not Client contract disputes). Both use Case.Description + Case.Subject as the search text — the content the rep is already looking at.

The reason these pre-filters are load-bearing is staffing-specific: in a single org, you have fundamentally different customer segments (candidates who are employees, clients who are employers) and different legal jurisdictions (UK employment law ≠ US employment law) coexisting in the same Case object. A retriever without pre-filters would mix those populations and produce misleading resolution patterns. (Source: prompt-templates.md, Template 2 — Key decisions #3 and Template 3 — Key decisions #1.)

This is the same grounding architecture used in Project CLAIrity — ~350 Knowledge Articles, semantic vector retrieval, no hallucination on factual claims. The difference is the pre-filter layer, which CLAIrity did not need because it operated in a single-market, single-segment context. The pre-filter addition is the adaptation that makes the pattern work at the scale and complexity of a staffing org.

**The three-tier variable architecture is what makes the agent stateful without breaking across record navigation.** Service console reps jump between cases constantly. Without explicit record-change detection, the agent would carry state from Case A into Case B — showing the wrong summary, surfacing the wrong similar cases, drafting an email for the wrong requestor. The pattern uses a `last_record_id` cross-turn variable that every subagent checks in its `before_reasoning` block. If the record ID has changed, all cached state is cleared. (Source: agent-definition.md, SYNC RULE, Tier 2 Variables.) This is not a platform default — it is an explicit design decision baked into every subagent's script. An implementation that skips this sync rule will produce silent state corruption that only surfaces when a rep navigates away and back.

**The verbatim output contract eliminates post-processing complexity.** All three prompt templates produce final, formatted HTML output that the agent emits directly to the chat panel without modification. There is no orchestration-layer cleanup, no regex, no second LLM pass. This keeps the system deterministic: if the output is wrong, the fix is in the prompt template, not scattered across the agent script. (Source: prompt-templates.md, Cross-Template Notes — Verbatim pass-through contract.)

**The five embedded email structures replace a retrieval-based approach for structure selection.** The Draft Response template embeds five structural email formats (Candidate general, Candidate time-sensitive, Client, Third Party, Unknown) directly in the prompt body rather than retrieving them from a Data Cloud object. This was a deliberate trade: the structure set is small and stable, and direct control over tone and register outweighs the flexibility of making structures retrievable. The retriever is reserved for KA content — the factual body — where semantic similarity is genuinely needed. (Source: prompt-templates.md, Template 2 — Key decisions #1.)

---

## 3. Alternatives Considered

**Flow automation instead of an agent.** An auto-triggered Flow on Case open could generate a summary and pre-populate a template-driven email. This would be faster to build and require no Agentforce licenses. The reason it falls short: Flow cannot do semantic similarity matching across a corpus of closed cases, and it cannot dynamically select and ground an email response using retrieved Knowledge Article content. The case summary and template-prefill pieces of this pattern are technically possible with Flow. The similar-cases retrieval and KA-grounded drafting are not — those require vector search, which requires Data Cloud and a retriever. If the customer's primary need is just case summarization with no retrieval, a pure Flow approach should be evaluated first. The Agentforce pattern is justified specifically by the retrieval requirements.

**Einstein Copilot actions without a dedicated agent.** The three subagents could in principle be built as standalone Copilot actions and invoked via the standard Einstein Copilot sidebar. The reason this wasn't the approach: the standard Copilot sidebar is not embedded in the Case record page context the same way a dedicated Internal Copilot is. An Internal Copilot agent deployed via Lightning App Builder appears where the rep is working, reads the page context automatically, and can be locked to Case-record-only behavior. The sidebar requires more rep navigation and cannot enforce the object-type guard that prevents the agent from being used on non-Case records. (Source: agent-definition.md, Topic 1 — routing — object_type guard.)

**Apex-based retrieval instead of Data Cloud.** A custom Apex class could query closed cases via SOQL and return the most similar ones using LIKE matching or a custom similarity algorithm. The reason this was rejected: the declarative-only constraint makes it non-viable (no Apex), and even without that constraint, SOQL-based similarity is keyword matching — it returns cases with the same words, not cases with semantically similar situations. A candidate asking "I haven't received my paycheck" and a candidate asking "there's a discrepancy in my compensation" are the same problem described differently. Keyword SOQL returns one, misses the other. Semantic vector retrieval returns both. (Source: pattern constraint — declarative-only; grounding model — Governed Unstructured Content layer.)

---

## 4. Why This Works for Staffing Specifically

Three properties of the staffing service model make this architecture particularly well-suited to the vertical:

**Multi-segment, multi-jurisdiction in a single org.** Most service orgs deal with one customer type (consumers, or enterprise clients, or employees). Staffing orgs deal with all three simultaneously — job seekers, employer clients, and third parties — in the same Case queue, often in the same rep's workload. The customer type pre-filter on the Case retriever and the structural template selection in the Draft Response template both depend on this segmentation being explicit in the data model. The `Derive_Customer_Type` Flow reads `[COMPANY]_Customer_Type__c` from the Case to drive both behaviors. This is not a generic pattern with a staffing label — the segmentation logic is structural. (Source: agent-definition.md, Topic 3 and Topic 4; flows.md.)

**High case volume with high requestor recurrence.** In staffing, the same candidate or client contacts service multiple times over a placement lifecycle. The Requestor History section of the Case Summary (populated by the `Get_Requestor_Cases` Flow) surfaces the last 10 cases for the same Contact, excluding the current one. For a new rep picking up a case from a high-frequency caller, this section eliminates the manual lookup that would otherwise eat 3–5 minutes of handle time. (Source: prompt-templates.md, Template 1 — Key decisions #2.)

**Knowledge Article content varies meaningfully by jurisdiction.** Employment law, payroll procedures, and placement terms are not uniform across markets. A US Knowledge Article about PTO accrual does not apply to a UK case about holiday pay. The country-scoped KA retriever pre-filter ensures the agent does not surface cross-jurisdiction content in the draft. For a single-market staffing firm, the pre-filter is unnecessary and can be removed. For a multi-market firm, it is the feature that prevents the agent from being actively misleading. (Source: prompt-templates.md, Template 2 — Key decisions #3.)

---

## 5. What Could Go Wrong

**`SDO_Service_Close_Summary__c` is an SDO-specific field.** The Similar Cases prompt template surfaces case resolution detail from this field. In the Robert Half org it is populated. In a production customer org, this field almost certainly does not exist. The symptom is similar cases cards that show no resolution detail — technically correct output, but low value. The fix is to identify the customer's equivalent field (common candidates: `Internal_Comments`, a custom `Resolution__c`, or a Flow-written close summary) and update the prompt template and data-kit.yaml before building. This is the highest-risk portability gap in the pattern. (Source: prompt-templates.md, Cross-Template Notes — SDO_Service_Close_Summary__c resolution.)

**The vector index needs lead time and case history.** The Case retriever requires a Data Cloud DMO with vector embeddings built on closed cases. On a new org or a recently activated Data Cloud instance, there may not be enough closed case history to make the similar cases feature meaningful. The typical lead time for a usable index is one full sprint (2 weeks) for index build plus enough historical cases to return relevant results. Plan for this in the implementation timeline. The agent is deployable before the index is complete — the similar cases subagent will return the graceful no-results message until the index matures. (Source: prompt-templates.md, Template 3 — Vector index note.)

**The SYNC RULE breaks silently if a new cross-turn variable is added without updating all subagents.** Every subagent has a `before_reasoning` block that clears cross-turn variables when the record changes. If a new variable is added to one subagent but not propagated to the clear block in all other subagents, stale state will persist across case navigation for that variable — and the bug will only surface when a rep navigates to a different case and asks a question that touches the new variable. The SYNC RULE in agent-definition.md is explicit about this. Any modification to the variable schema must update the clear block in every subagent, not just the one that introduces the variable. (Source: agent-definition.md, SYNC RULE.)

**Customer type classification breaks if the Contact RecordType model differs from RH.** The `Derive_Customer_Type` Flow reads `[COMPANY]_Customer_Type__c` and returns one of four values (Candidate, Client, Third Party, Unknown). This enum is RH-specific. For a different staffing firm with a different segmentation model (e.g., Perm vs. Contract vs. Temp rather than Candidate vs. Client), the flow output, the prompt template structure selection, and the retriever pre-filter values all need to be updated consistently. A mismatch between the flow's output enum and the retriever's pre-filter values will produce silent filtering failures — the retriever will pre-filter on a value that never matches and return zero results without an error. (Source: agent-definition.md, Variables — Tier 3, `customer_type`.)
