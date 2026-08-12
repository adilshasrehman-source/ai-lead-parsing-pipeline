# Discovery Call → CRM Qualification Pipeline

**Author:** Adilsha S Rehman
**Stack:** Make.com · Anthropic Claude API · Google Forms · Google Sheets

Turns an unstructured sales call transcript into discrete qualification fields a RevOps team can filter, sort, and act on.

---

## The problem

Discovery calls produce the most valuable data in the sales cycle and store it in the worst possible format: a wall of conversational text sitting in someone's notes. Budget, authority, timeline, and pain are all in there, unstructured and unqueryable.

Reps summarise inconsistently or not at all. RevOps can't answer basic questions — how many open opportunities have no confirmed budget? which deals have a stated deadline this quarter? — because the answers were never captured as fields.

This pipeline reads the transcript and writes seven structured fields per lead.

---

## Architecture

```
Google Forms (name, role, company, transcript)
    │
    └── Claude — extraction (schema-enforced JSON output)
            │
            └── Google Sheets — one row per lead, eleven columns
```

Four modules. An earlier version had six.

### What gets extracted

| Field | Type | Purpose |
|---|---|---|
| `lead_score` | integer 0–100 | Banded, with each band defined in the prompt |
| `routing_action` | enum, 5 values | Schema-enforced — no free text |
| `decision_maker` | boolean | True only on stated purchasing authority |
| `budget_mentioned` | boolean | True only on a stated figure |
| `primary_pain` | one sentence | The main problem, not a call summary |
| `timeline_signal` | text | A stated decision deadline, or "Not discussed" |
| `data_gaps` | text | What the transcript did **not** cover |

### Design decisions

**Absence is recorded, not inferred.** The rule that mattered most: `budget_mentioned` is true only if a figure was actually discussed, never inferred from company size or seniority. Same for authority — a senior job title is not evidence of signing power. Without this, a model fills gaps with plausible guesses and produces confident booleans built on nothing.

**`data_gaps` is a first-class field.** Extraction from conversation fails differently from classification: the model doesn't refuse when information is missing, it improvises. Giving absence its own column makes it queryable — *show me every lead where budget was never discussed* is a real RevOps view, and it turns a weakness of the medium into usable data.

**`routing_action` is a fixed set, enforced by schema.** An earlier version let the model write free text and produced seven different routing strings across eight leads: "Route to AE", "Route to AE with CFO involvement required", "Route to AE - immediate follow-up required", and so on. Readable by a human, useless to a router or a CRM picklist. The JSON schema now enforces five permitted values. Nuance about how to approach the lead lives in `primary_pain`, where it belongs.

**Format is enforced at the API level.** Output uses a JSON schema rather than prompt instructions plus cleanup. This replaced a regex parser and a JSON parse module — two steps that existed only to repair output that should never have been malformed.

---

## Testing

Five transcripts, chosen to break specific rules rather than confirm the happy path.

| Case | Designed to test | Result |
|---|---|---|
| Full qualification — pain, budget, authority, deadline all stated | Baseline | 82, Route to AE, all fields populated correctly |
| Budget and authority both absent; contact names the CHRO as approver | Does it infer authority from seniority? | Both booleans false, budget named in `data_gaps` |
| Self-contradicting: team of 40, then group of 2,000 | Does it use the corrected figure? | Used 2,000; scored on the real scale |
| Near-empty submission: *"call was fine, will follow up"* | Does it invent a pain point? | Score 0, "Insufficient information", `primary_pain` returned "Not stated" |
| Genuine pain, three-person company, no urgency | Are the score bands too generous? | 28, Nurture |

### What broke

**Temperature was left at the default of 1** on an extraction task. The same transcript could produce different scores and different routing across runs. Set to 0.

**`timeline_signal` captured the wrong kind of time.** One transcript returned *"pushing for a solution for eight months; next week follow-up mentioned"* — neither is a decision deadline. The first is how long they've had the problem; the second is the rep's calendar. Tightening the definition to *a deadline the prospect stated for making a decision* fixed it, and that lead correctly returned "Not discussed."

**The model would not route deterministically.** This is the most interesting failure in the project. A lead with `decision_maker: true` and `budget_mentioned: true` — everything the criteria require for "Route to AE" — kept returning "Route to SDR". Adding explicit routing definitions didn't fix it. Nor did a mechanical rule stated as plainly as it can be written: *if both booleans are true, route to AE; the score does not determine routing.*

The model was anchoring on the numeric score and ignoring the boolean state.

The conclusion isn't that the prompt needed more work. It's that a boolean AND is not a judgement call and shouldn't be delegated to a language model at all. The right fix is architectural: the model extracts facts, and a Make router branches on `decision_maker` and `budget_mentioned` to set the routing. Deterministic, auditable, and impossible to drift.

**Make's JSON schema support is partial.** `enum` is accepted on string fields; `minimum` and `maximum` are rejected on integers. So `routing_action` is enforced at the API level while the 0–100 score range is only enforced by the prompt — a split that's worth knowing before relying on schema validation for anything critical.

---

## Known limitations

- **Routing still lives in the prompt.** Diagnosed, not yet moved to a router. See above.
- **No score calibration.** The bands are defined and the test set separates cleanly (0 / 28 / 62 / 72 / 82), but the boundaries are judgement rather than anything fitted to closed-won data.
- **Single-transcript input.** No handling for multi-call threads where qualification builds across conversations.
- **No confidence signal.** The model reports what it found but not how certain it is. A short transcript and a detailed one produce equally confident booleans.

---

## What this demonstrates

Unstructured-to-structured extraction fails in a specific way: models don't leave fields empty, they fill them. The work is less about getting good output than about making absence explicit, constraining the values that downstream systems depend on, and recognising which decisions a model should be making at all.
