# Solution Launchpad — Manifest Schema Reference

This document describes the schema for `index.yaml`. Read this before adding a new template entry.

---

## Top-Level Structure

```
manifest_version    string     Schema version. Increment on breaking changes.
last_updated        date       ISO 8601. Update whenever a template entry changes.
vocabulary          object     Canonical tag lists. Add new tags here before using them.
global_anti_tags    list       Rules that apply across all templates by default.
templates           list       One entry per solution template.
```

---

## Template Entry Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique slug. Matches the directory name under solution-launchpad/. |
| `label` | string | yes | Human-readable name shown in skill output. |
| `slug` | string | yes | Same as id. Used for URL construction in GitHub raw fetch. |
| `path` | string | yes | Relative path from this file to the template directory. |
| `validation_status` | enum | yes | `validated` / `in-progress` / `draft`. Only `validated` templates are returned as primary recommendations. `in-progress` templates are returned with a warning. `draft` templates are not returned. |
| `source_org` | string | yes | The org the template was extracted from. |
| `extracted` | date | yes | ISO 8601 extraction date. |
| `shaheen_pattern` | string | no | The Shaheen framework pattern this template implements (e.g., SOMA, MOMA). |
| `summary` | string | yes | 3–5 sentence plain-language description. This is what the SE reads first. |
| `tags` | object | yes | Positive tags by dimension. See Tag Dimensions below. |
| `requires` | object | no | Named prerequisites — used to generate the Prerequisites Gap Report. |
| `anti_tags` | list | no | Template-level anti-tag overrides. Add and extend; do not remove global rules here. |
| `files` | list | yes | List of files in this template directory. |
| `qualification_signals` | list | no | Discovery signals that confirm this pattern fits. Used in the qual brief. |
| `disqualification_signals` | list | no | Discovery signals that rule this pattern out. |
| `compensation_shape` | object | no | SKU notes for AE coordination. Never quote specific rates. |

---

## Tag Dimensions

All tag values must exist in the `vocabulary` section of `index.yaml`. Do not use undeclared tag values — the skill will flag them as unrecognized.

| Dimension | Purpose | Matching weight |
|---|---|---|
| `channel` | How the agent is delivered | Medium |
| `industry` | Customer vertical | Low (narrows, doesn't pick winner) |
| `capability` | What the agent does | **High** (primary matching dimension) |
| `data_sources` | What data the agent requires | High |
| `surface` | Where the agent is deployed | Medium |
| `persona` | Who the agent serves | Medium |
| `org_stack` | Required Salesforce licenses | High (feeds anti-tag evaluation) |
| `build_constraints` | Technical build constraints | High (feeds anti-tag evaluation) |

**Capability tags are the primary matching dimension.** When two templates have similar industry and persona tags, capability match count is the tiebreaker.

---

## Anti-Tag Schema

```yaml
- condition: <string>        # The SE-provided input that triggers this rule
  blocker: <bool>            # true = Tier 3 (disqualifies). false = Tier 2 (warns).
  missing: <bool>            # Optional. true = fires when template does NOT have the trigger tag.
  message: <string>          # What the SE sees in the gap report. Be specific about the fix.
  triggers_on_template_tag: <string>        # Fires if template has this tag
  triggers_on_template_capability: <string> # Fires if template has this capability
  triggers_on_template_data_source: <string># Fires if template uses this data source
```

**Tier 3 (blocker: true):** Template is disqualified. SE sees a clear "BLOCKED" message with the reason. The skill will suggest the next-best non-blocked template if one exists.

**Tier 2 (blocker: false):** Template is returned with a gap section. SE sees "PREREQUISITE GAP: [message]." The gap report is meant to feed the AE conversation, not stop the SE from proceeding.

---

## Validation Status Rules

| Status | Meaning | Skill behavior |
|---|---|---|
| `draft` | Written but not tested | Not returned in skill output |
| `in-progress` | Passes parametrization test; second-customer test pending | Returned with warning: "Pattern not yet validated on a second engagement" |
| `validated` | Passes all three tests (parametrization, second-customer, new-SE) | Returned as primary recommendation |

Do not set `validated` without documenting which accounts satisfied the second-customer and new-SE tests.

---

## Adding a New Template — Checklist

- [ ] Create directory: `solution-launchpad/<slug>/`
- [ ] Write required files: `README.md`, `agent-definition.md`, `prompt-templates.md`, `flows.md`, `permissions.md`
- [ ] Write optional files if applicable: `data-kit.yaml`, `data-cloud-setup-guide.md`
- [ ] Add entry to `index.yaml` under `templates:`
- [ ] Tag against existing vocabulary. Add new tags to `vocabulary` first if needed.
- [ ] Set `validation_status: draft` until parametrization test passes
- [ ] Promote to `in-progress` after parametrization test
- [ ] Promote to `validated` after second-customer and new-SE tests
- [ ] Update `last_updated` in `index.yaml`
