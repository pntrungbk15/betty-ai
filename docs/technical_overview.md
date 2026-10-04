# Technical overview

[← Back to README](../README.md)

This page covers the parts of Betty other than retrieval (see
[retrieval_and_estimation.md](retrieval_and_estimation.md)).

## Execution planner

Each user message is planned before anything expensive runs.

| Output | Values | Used for |
|---|---|---|
| Route | `chat`, `create`, `edit`, `clarify`, `confirm` | Which stages run |
| Maturity | Exploration → Definition → Confirmed | Shown in the header; decides whether a draft is possible |
| Capabilities | image analysis, ticket search, ticket generation, metadata inference, validation | Which tools the turn may use |
| Signals | project, screen, problem stated, request stated, expected stated, attachments | Why the route was chosen |
| Retrieval query | the user's words, expanded with image findings | Input to hybrid retrieval |

If the conversation is still in *Exploration* (for example "Something is off when we create bookings"), the planner
keeps the turn in `chat` and Betty asks targeted questions instead of drafting.

![Planner](../assets/screenshots/04-planner-inspector.png)

## Screenshot analysis

1. **Pixel regions** — red error states and the user's orange annotations are found as connected components.
2. **UI elements and visible text** — labels, fields, buttons, messages.
3. **Findings** — inferred from geometry and text: an error message that sits under one field but names another,
   header-to-cell offsets in tables, clipped columns, error toasts.
4. **Reproduction steps** derived from the screen's breadcrumb and the findings.

> **Showcase limitation.** Step 1 is computed from pixels. Steps 2–3 use text and element boxes that the generator
> embedded in each synthetic screenshot, and the UI says so. This demonstrates the downstream reasoning; it is not a
> claim of full visual understanding. A vision-language model backend would provide steps 2–3 from the pixels.

![Screenshot analysis](../assets/screenshots/05-screenshot-analysis.png)

## Ticket generation (structured output)

The model returns JSON that matches a schema: title, context, observed and expected behaviour, steps to reproduce,
acceptance criteria (each with its origin: a reference ticket, the clarification, the bug template or an edit), type,
importance and the estimate with reasoning. The pipeline then:

- infers metadata from the references,
- assigns the key and repairs the hierarchy in code,
- renders the card with a template.

The model never writes HTML, ticket keys or hierarchy repairs.

## Validation

**Structural validator** (deterministic):

- the path project › module › process exists,
- the iteration belongs to the project and is not closed.

A problem with exactly one sensible fix is repaired and reported (e.g. a closed sprint → the active sprint). Anything
else becomes a question with the valid options.

**Model validator** (content): title quality, observed and expected behaviour present, number of acceptance
criteria, effort within the references' range, and how strongly the references agree on importance.

## Clarification wizard

A small state machine — `idle` → `asking` → `complete` — over the open findings. Each question offers options drawn
from the references plus free text, and can be skipped. Answers update the draft and re-run validation.

![Validation and clarification](../assets/screenshots/08-validation-clarification.png)

## Human editing and re-estimation

Edits arrive as chat ("set importance to critical", "add a criterion that …", "also cover …") or as direct field
edits. Chat edits are classified by impact — `cosmetic`, `clarification`, `metadata` or `scope` — and only a scope
change triggers re-estimation. Every change creates a new draft version; the Versions tab shows the impact class,
the estimate before and after, and a field-by-field diff.

![Edit and re-estimate](../assets/screenshots/09-edit-re-estimate-diff.png)

## Observability

Every step records its measured latency and whether it was a model call or a code step; model calls also record
input and output token counts. The panel aggregates per conversation (turns, model calls, total time, tokens) and
lists every step.

> In the showcase, token counts are **estimates** from a word-piece heuristic over the rendered prompt and the JSON
> answer, because the default backend calls no model API. The panel labels them as such.

![Observability](../assets/screenshots/10-observability.png)

## Model backends

| Backend | Behaviour |
|---|---|
| Scripted demo model (default) | Deterministic, computed from the inputs (conversation, findings, references); the same conversation always yields the same draft. Understands only the walkthrough's English phrasing and edit commands. |
| OpenAI-compatible endpoint (optional) | Any model with JSON-schema output (and image input for screenshots), e.g. a local model via Ollama, or a hosted API with an OpenAI-compatible interface. |

## Quality and testing approach

- The core (retrieval, planning, validation, rendering, wizard, versions) runs without a web server.
- The demo model is deterministic, so the full walkthrough is reproducible.
- No accuracy, time-saving or adoption figures exist for the showcase or the production assistant, and none are
  claimed.
