# Solution Template: Staffing — Service Console Agent

## What This Is
A reusable solution template for deploying an Agentforce Internal Copilot (Service Console Agent) in a staffing industry context. Built from the Robert Half POC (June 2026). Covers rep-facing Case assistance: case summary with requestor history, similar resolved case retrieval, and AI-drafted email responses grounded on Knowledge Articles.

## Pattern Classification
- Shaheen Framework: Operator with Domain SME Workers (SOMA)
- Grounding Layers: Structured CRM Records (Cases, Contacts) + Governed Unstructured Content (Knowledge Articles)
- Deployment Surface: Service Console — Case record page (Lightning App Builder component)
- Engagement Stage: POC → Production

## Input Variables (9)
| Variable | RH Value | Your Value |
|---|---|---|
| Product | Agentforce for Service (Internal Copilot) | |
| Industry | Staffing | |
| Use Case Type | Case Summary, Similar Cases, Draft Email Response | |
| Deployment Surface | Service Console — Case record page | |
| Data Sources | Knowledge Articles (ADL), CRM Cases (Data Cloud) | |
| Persona | Service Console CSM (rep-facing) | |
| Engagement Stage | POC / Production-readiness | |
| Customer Prerequisites | Service Cloud, Data Cloud provisioned, Knowledge Articles populated | |
| Compensation Shape | EO subcloud (AF for Service, 100%) + FA (Data Cloud, 100%) | |

## Prerequisite Gap Notes
- **Data Cloud**: Required for both retrievers. If not provisioned, the Similar Cases and Draft Response subagents cannot function as built. Data kit manifest is in data-kit.yaml — present to AE for SKU conversation before scoping.
- **Knowledge Articles**: Must be populated before Knowledge retriever produces grounded results. Minimum viable: 20–30 articles covering the top case types. Zero articles = agent responds with fallback message.
- **Custom Case Fields**: 4 custom fields required on Case object (see agent-definition.md). These are cheap to create but must be populated by the email-to-case routing flow or manually.

## Contents
| File | Purpose |
|---|---|
| agent-definition.md | Parametrized agent definition: variables, subagents, system prompt |
| prompt-templates.md | Full parametrized prompt text for all 3 Console Agent templates |
| flows.md | All flows with explicit purpose, inputs, outputs, parametrization notes |
| permissions.md | Permission sets, field permissions, bot user requirements, setup toggles |
| data-kit.yaml | Machine-readable Data Cloud component manifest |
| data-cloud-setup-guide.md | Step-by-step Data Cloud setup (parametrized) |

## Validation Status
- [ ] Parametrization test: account-specific values replaced with placeholders
- [ ] Second-customer test: used on at least one additional staffing account
- [ ] New-SE test: SE unfamiliar with use case reached first demo in <2 days using only this template

## Source
Extracted from live org: trailsignup-b840dc69c3ac6b (rhDemo alias)
Extraction date: 2026-06-18
SF CLI retrieve: Bot, GenAiPromptTemplate, GenAiPlanner, Flow, PermissionSet, CustomObject:Case
