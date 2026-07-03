# Prototype design — Cognigy project-membership check for "Open published AI agent" (CSA-86274)

**Date:** 2026-07-03
**Author:** Sujoy Datta (PM)
**Type:** Stakeholder prototype (standalone HTML, non-production)
**Related ticket:** CSA-86274

---

## Problem

In the AI Agent Preview / AI Agent Details surface, the **"Open published AI agent"** control appears whenever an
agent has a Published version — regardless of whether the current user can actually open it in Cognigy AI. Cognigy
grants access **per project**, so a Published agent living in a project the user doesn't belong to yields a dead-end
click.

The existing ticket proposes hiding the button unless the current user *created/published* the agent. That is an
**identity check standing in for an authorization check**. Projects can have multiple members, so "am I the publisher"
gives false negatives (a teammate who is also a project member gets the control wrongly hidden) and degrades further as
team-shared projects become the norm.

## Recommended behavior (Option B)

Check each published version's Cognigy `projectId` against the set of projects the current user belongs to. Reuse the
**same membership data that already populates the Project Name dropdown** — `fetchCognigyProjects()` in
`src/services/cognigy-projects-service.ts` returns `CognigyProject[]` (`{ projectId, name, ... }`). No new data source.

Access is evaluated **per published version**, because an agent can be published to multiple different projects — each
`PublishedMetadata` entry carries its own `projectId`.

## Grounding in the real codebase

Mirrors these real components (repo `nice-illuminate/csa-agentic-analytics-ao-webapp`):

- `src/lib/registry/components/view-ai-agent-published-versions.tsx` — `PublishedVersionsChip`: renders a **single ghost
  link** (`ExternalLink` icon + "Open published AI agent", color `#126BCE`) when there is 1 version, and a **dropdown**
  (chevron) listing versions when there are >1. Today it is only disabled when a version has no `cognigyAgentUrl`.
- `src/lib/registry/components/view-ai-agent-header.tsx` — `ViewAgentHeader`: breadcrumb context, title
  (`Topic: <name>`), `Project Name:` select, `Created By` / `Creation Date` / `Status` blue badge
  (`#E5F2FF` bg / `#126BCE` text). Already contains the **canonical disabled-button-with-tooltip pattern** (a `<span>`
  wrapper around a disabled `Button` + Radix `Tooltip`) for the empty-projects case — the prototype reuses this exact
  treatment for the "no access" state.
- `src/services/cognigy-projects-service.ts` / `cognigy-projects-types.ts` — membership source.
- Design system: shadcn/ui (`base-nova`) + `@nice/cognigy-ui-component-library`, Geist font, primary `#126BCE`,
  soft `#E5F2FF`, border `#E5E7EB`.

## The four scenarios (the core of this prototype)

| # | Published | User access | Control shape | Behavior |
|---|---|---|---|---|
| 1 | 1 version | Has access | Single link | Enabled — opens normally |
| 2 | 1 version | No access | Single link | Disabled + tooltip naming the project |
| 3 | Multiple versions | Access to all projects | Dropdown | Trigger enabled; every item enabled |
| 4 | Multiple versions | Access to some | Dropdown | Trigger enabled; accessible items enabled, inaccessible items disabled + per-item tooltip naming the project |
| 4b | Multiple versions | Access to **none** | Dropdown | **Trigger still enabled and opens.** Every item disabled + per-item tooltip naming its project, plus a header line inside the dropdown: "You don't have access to any of these projects." |

**Design principle — the dropdown is consistent.** For multiple versions the trigger *always* opens, regardless of
access. Access is gated **per row**, never at the trigger. This collapses scenarios 3 / 4 / 4b into one behavior that
only varies row-by-row — one mental model for the user, and it mirrors the single-version rule (Scenario 2 disables the
*action*, it never hides the control). The no-access case (4b) adds an in-dropdown header line for a summary, but still
shows every version and its project so the user knows which owner to ask.

**Fail-open overlay** (applies over all of the above): if the membership check itself fails/unavailable, controls stay
**enabled** with a small amber "unverified access" marker + tooltip, rather than silently disabling a possibly-valid
action.

## Three button states — distinct visual treatment

- **Member (enabled):** ghost blue button/item, `external-link` icon, fully interactive.
- **Not a member (disabled):** `<span>`-wrapped disabled button (muted, `pointer-events-none`) + hover/focus tooltip:
  *"You don't have access to project '[Project Name]' in Cognigy — ask a project owner to add you."* Status badge stays
  **Published** — only the broken action is disabled, never the data.
- **Unverified (fail-open):** enabled button + amber `alert-triangle` "Unverified access" badge alongside; tooltip:
  *"Couldn't verify your Cognigy access — the link may not work."*

## Prototype structure (single self-contained `.html` file)

File: `prototype/ao-open-published-project-check.html`

1. **Simulation panel (sticky top)** — the simulated input standing in for the membership API:
   - "Your Cognigy project memberships" — 3 toggle chips: `Billing Ops`, `Retention Team`, `Onboarding`.
   - "Membership check" — toggle `Working ↔ Failing (fail-open)`.
   - Inline annotation: "Stands in for `fetchCognigyProjects()` — same source as the Project Name dropdown."
2. **Scenario 1 — AI Agent Details page recreation** — full header (breadcrumb `Automation Opportunities › AI Agents`,
   title `Topic: Bill Explanation`, Project Name field, Created By / Creation Date / Status badge) with a
   **"1 version ↔ 3 versions" switcher** so reviewers see both the single-link and dropdown forms live, reacting to the
   toggles.
3. **States reference strip** — scenarios 1, 2, 3, 4 and 4b rendered as compact, labeled, live examples side-by-side so
   developers can eyeball every state at once — including the 4b dropdown opened with all rows disabled + the no-access
   header line.
4. **Scenario 2 — dashboard cards** — 3 topic cards, each published by a *different* creator to a *different* project
   (`Bill Explanation`/Billing Ops/Sarah Chen, `Cancel Subscription`/Retention Team/Marcus Lee,
   `Reset Password`/Onboarding/Priya Nair), single-button controls. Teaching point: a card published by someone else to
   a project you *do* belong to is still openable — proving "creator == me" is the wrong check.
5. **Notes & open questions (collapsible footer)** — carries the stakeholder discussion points, each pre-answered where
   the repo already tells us.

## Tech / fidelity

- Standalone HTML + inline CSS + vanilla JS (no build). Matches the existing `prototype/*.html` workflow and is
  shareable via GitHub Pages like the current `index.html`.
- Geist font from `prototype/c26/fonts/`; Lucide icons via CDN (`external-link`, `chevron-down`, `alert-triangle`,
  `check`). Repo colors used verbatim (`#126BCE`, `#E5F2FF`, `#E5E7EB`) so developers recognize their own components.

## Open questions (surfaced in-prototype for stakeholders)

1. **Is "my Cognigy projects" data already on this page?** → **Answered: yes.** `fetchCognigyProjects()` already backs
   the Project Name dropdown; reuse it — no new fetch.
2. **Do we track "Published By" separately from "Created By"?** → **Still open.** Which is authoritative if they diverge?
   (Access is a project-membership question, so likely neither is authoritative — but flag for confirmation.)
3. **Per-version access scope** → **Answered: in scope.** Each version has its own `projectId`; access is per-version.
4. **Disabled-button visual treatment** → **Answered: reuse** the existing empty-projects `<span>`+Tooltip pattern in
   `view-ai-agent-header.tsx`; do not invent a new one.

## Out of scope

- Real Cognigy API integration — membership is simulated via UI toggles.
- Changes to the Publish flow or the Status field logic.
- Any production/React code — this is a throwaway visual prototype for stakeholder alignment.
