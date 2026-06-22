# Solution Launchpad — Template Library

Maintained by Victor Noé (L7 AFSE, AMER_ACC_CBS_AFDC).

This library holds parametrized solution templates for Agentforce engagements. It is the backend of the Solution Launchpad skill — when an SE runs the skill with discovery variables, the skill fetches `index.yaml` from this repo, matches against template tags, and returns a parametrized artifact kit recommendation.

**Do not use these files directly as customer deliverables.** They are parametrized patterns. Replace all `[COMPANY]` placeholders before use.

---

## How the Library Works

```
index.yaml              ← Skill fetches this first. Contains all template tags and matching metadata.
schema.md               ← Tag vocabulary reference and contribution rules.
<template-slug>/        ← One directory per solution template.
  README.md             ← Pattern overview, input variables, validation status.
  agent-definition.md   ← Parametrized agent definition.
  prompt-templates.md   ← Parametrized prompt text.
  flows.md              ← Flows with purpose, inputs, outputs.
  permissions.md        ← Permission sets, field-level security, setup toggles.
  data-kit.yaml         ← Data Cloud component manifest (if pattern requires Data Cloud).
  data-cloud-setup-guide.md ← Step-by-step Data Cloud setup (if applicable).
```

---

## Available Templates

| Template | Industry | Core Capability | Status |
|---|---|---|---|
| [staffing-console-agent](staffing-console-agent/) | Staffing | Case summary, similar cases, email draft | In Progress |

---

## Using a Template

1. Run the Solution Launchpad skill with your discovery variables
2. The skill will match your inputs to the closest template and return a gap report
3. Review the README.md in the matched template directory
4. Replace all `[COMPANY]` placeholders with the customer's values
5. Follow the setup guides in sequence (Data Cloud setup first if applicable)
6. Use the flows.md and permissions.md to complete the Salesforce org setup

---

## Contributing a New Template

After extracting a pattern from a customer engagement:

1. Read `schema.md` for the full contribution checklist
2. Create a new directory using the pattern slug
3. Write the required files (agent-definition, flows, permissions, prompt-templates)
4. Add the entry to `index.yaml` with `validation_status: draft`
5. Promote to `in-progress` after parametrization test passes
6. Post the write-back in the team Slack canvas when the template is `in-progress` or `validated`

**The validation gate:** A template is only `validated` after it has been used on a second customer engagement (second-customer test) and an SE unfamiliar with the pattern reached a working first demo in under two days using only these files (new-SE test).

---

## Tag Vocabulary

The full vocabulary of valid tags is in `index.yaml` under the `vocabulary:` key. Do not use tags outside this vocabulary — the skill will reject unrecognized tags. If a new tag is needed, add it to the vocabulary first, then use it in the template entry.

---

## Anti-Tags

Anti-tags represent conditions that disqualify or warn against a template based on customer constraints:

- **Tier 3 (blocker):** Template is disqualified. Skill returns next-best alternative.
- **Tier 2 (warning):** Template is returned with a gap report section. Feeds the AE/prerequisites conversation.

Global anti-tags (applying to all templates) are defined in `index.yaml` under `global_anti_tags:`. Template-specific anti-tags are in each template's entry under `anti_tags:`.
