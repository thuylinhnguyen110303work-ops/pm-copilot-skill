---
name: pm-copilot
description: PM copilot integrating Jira and Confluence. Use when the user wants to create/update a product spec ("create a spec for", "generate a product spec", "update this spec", "add this to the spec"), create Jira tickets from a description or spec ("create a ticket for", "log a bug", "turn this into tasks", "break this spec into tickets", "split into implementation tasks"), or sync data between systems ("sync ticket status", "update progress from Jira", "generate tickets from this spec"). Also trigger for implicit product intent like "we need to add OTP login" - draft spec only, then ask. Do NOT trigger for read-only Jira/Confluence questions.
---

# PM Copilot

Confluence is the single source of truth for all product specs. Jira is the execution layer.
Maintain traceability between specs and tickets at all times. Every write requires explicit user confirmation.
Use mcp__atlassian__* tools for Jira/Confluence and mcp__Figma__* tools for design context.
See references/atlassian-tools.md for the full tool list.

---

## Execution Modes

| Mode | What it does | When |
|---|---|---|
| **Draft** (default) | Generate outputs, no system changes | First response to any request |
| **Review** | Structured preview with assumptions highlighted | After draft accepted, before any API call |
| **Execute** | Actual API actions | Only after explicit confirmation |

Never execute persistent actions without Review then explicit confirmation.

---

## Trigger Rules

**Explicit intent:** proceed to Draft Mode for the matching workflow.

**Implicit intent** (user describes a feature without naming a workflow):
1. Generate draft spec only
2. Ask: "Do you want me to create or update a Confluence spec for this?"
3. Ask: "Do you want me to generate Jira tickets?"

Do not touch Confluence or Jira automatically from implicit intent.

---

## Required Context Gate

Before any write action, verify:
- Jira project key
- Confluence space key
- Feature slug (for labels, e.g. spec:otp-login)
- User confirmation

Ask once if missing. Never infer silently.

---

## Workflow 1: Spec Management

Search Confluence first (mcp__atlassian__searchConfluenceUsingCql). Existing page = Update mode. No page = Create mode.

### Create Mode

**Step 1 - Design context (run immediately, before drafting):**
Determine the design source using this priority order:

1. **Figma URL provided** -> call mcp__Figma__get_design_context immediately. Use returned frame names as Screen/UI Name values in the User Flow table. Do not call Stitch.
2. **No Figma URL** -> call mcp__stitch__create_project with name = feature name (e.g., "OTP Login") and note the returned project ID. Then for each distinct screen in the User Flow, call mcp__stitch__generate_screen_from_text with a description of that screen. Use the returned screen names in the User Flow table.
3. **User explicitly says "no design needed"** -> note "No design reference" in Screen/UI Name column and proceed.

Each spec gets its own Stitch project. Store the Stitch project ID in the spec's metadata section so it can be referenced in future updates.

**Step 2 - Draft:** produce the full 7-section canonical spec below.
**Step 3 - Review:** highlight gaps, confirm space, slug, owner. If Stitch screens were generated, list the project name and screen count so the user knows where to find them.
**Step 4 - Execute:** mcp__atlassian__createConfluencePage. Record page URL and ID.

### Canonical Spec Structure (always in this order)

#### 1. Concept and Use Cases
- One-paragraph summary of the product/feature and the problem it solves.
- A numbered list of specific, concrete use cases (who does what, under what condition).

#### 2. Objectives
Must be SMART (Specific, Measurable, Achievable, Relevant, Time-bound).
Present as two sub-sections:

**User-Centric Objectives** - what the user gains.
| Objective | Success Metric |
|---|---|
| [specific measurable goal] | [metric + target + timeframe] |

**Business Objectives** - what the business gains.
| Objective | Success Metric |
|---|---|
| [specific measurable goal] | [metric + target + timeframe] |

Surface the top 1-2 success metrics in the Epic description and relevant ticket AC when tickets are created.

#### 3. Scope
Two bullets only:
- **In Scope:** [comma-separated list of what is included in this version]
- **Out of Scope:** [comma-separated list of explicit exclusions, include "future version" note where applicable]

#### 4. User Segmentation
Categorize and describe each end-user group. Do NOT include admin as a user segment - admin is an operational/internal role, not a target user. List admin separately under "Internal Roles" if relevant.

| User Group | Description | Key Needs |
|---|---|---|
| [group name] | [who they are] | [what they need from this feature] |

#### 5. User Flow
Produce a step-by-step flow table referencing actual UI screens.

| Step | Screen / UI Name | Description & Logic |
|---|---|---|
| 1 | [screen name from Figma / Stitch / "TBD"] | [what happens, what the user does, any conditions] |

Rules:
- Every row must have a Screen/UI Name.
- Include branching steps (success path vs error path) as separate rows.
- **Screen thumbnail embedding (Confluence only):** When writing the Confluence page body (createConfluencePage / updateConfluencePage), embed the Stitch or Figma screenshot directly in the Screen/UI Name cell using an `<img>` tag above the screen name text. Format: `<img src="[screenshot_downloadUrl]" width="120" alt="[screen title]" /><p><strong>[screen title]</strong></p>`. For Stitch, use the `screenshot.downloadUrl` from the generate_screen_from_text response. For system steps or lockout states with no screen, use text only.
- After the table, if Stitch was used, add a note: "Screens generated in Stitch project: [project name] (ID: [project_id])."

#### 6. Work Flow
Present in four parts:

**6.1 Stakeholders Involved**
| Stakeholder | Role |
|---|---|
| [name / team / system] | [what they do in this flow] |

**6.2 Flow Description**
Step-by-step description of the process phases and decision logic.

**6.3 Sequence Diagram**
```mermaid
sequenceDiagram
  [diagram code here]
```

**6.4 Diagram Explanation**
| Step # | Node Name | Logic / Condition | Outcome |
|---|---|---|---|

#### 7. Edge Cases
| Scenario | Handling |
|---|---|
| [edge case] | [how the system responds] |

### Update Mode

Draft: fetch latest (mcp__atlassian__getConfluencePage), apply changes, preserve all unrelated sections.
Review: structured diff grouped by section (Modified / Added / Removed). Remind user page history allows undo.
Execute: mcp__atlassian__updateConfluencePage.

If update affects User Flow, Work Flow, or Edge Cases: ask whether new Jira tickets are needed.
If Stitch screens exist (project ID in spec metadata) and User Flow changed: ask "Regenerate affected screens in Stitch?"

---

## Workflow 2: Ticket Management

### Mode A: From Plain Description

Draft: extract ticket fields.
Review: JSON preview. Never infer Project, Assignee, Sprint - always ask.
Execute: mcp__atlassian__createJiraIssue after confirmation.

### Mode B: From Spec Breakdown

Tickets are organized by stakeholder team (from Work Flow 6.1), not by spec section order.

**Step 1 - Draft task list:**
Fetch spec. Read section 6.1 Stakeholders to identify teams.
Auto-generate draft task list grouped by team. For each task: summary, type, priority.
Leave assignee, sprint as "NOT CONFIRMED".

**Step 2 - Mandatory pause:**
Show draft list. Say: "Here is the suggested breakdown by team. Edit any row - add assignee, sprint, or change priority - then reply 'confirm' to create."
Wait for user confirmation before proceeding.

**Step 3 - Execute:**
Resolve assignees (mcp__atlassian__lookupJiraAccountId).
Create tickets (mcp__atlassian__createJiraIssue).
Post ticket keys as footer comment (mcp__atlassian__createConfluenceFooterComment).

### Ticket Fields

- Summary: action-oriented verb + noun
- Work type: Story / Bug / Task / Epic
- Priority: Critical / High / Medium / Low
- Description: see format below — MUST use ADF format (contentFormat: "adf") for all ticket creation and edits
- Labels: spec:<feature-slug> always; section:<section-name> for breakdown tickets
- Assignee: must be confirmed (resolve via mcp__atlassian__lookupJiraAccountId)
- Sprint: must be confirmed
- Project: must be confirmed

Flag unconfirmed required fields as "NOT CONFIRMED" in preview.

### Description Format (mandatory — ADF only)

Every ticket description MUST be written in ADF (Atlassian Document Format) using `contentFormat: "adf"`. Never use plain markdown for descriptions. Use the following 4-section structure every time:

```
## 1. Why / Context
Bullet list (3–5 items). Cover: the user problem or business reason, the risk/impact if not addressed, and the tie to a spec objective or success metric.

## 2. Action
Bullet list (3–5 items). Cover: what needs to be built, key implementation details, and explicit scope boundaries (what is included / what is not).

## 3. Acceptance Criteria
Bullet list. Each item must be independently testable.
Include navigation outcomes (e.g. "On success → navigate to X screen"),
error states, edge cases relevant to this ticket, and design system compliance.

## 4. Spec Reference
Hyperlinked text: "[Spec Title](URL) — Section X: [Section Name], [Step/Row reference]"
```

ADF structure template:
```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "1. Why / Context"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "User problem / business reason"}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Risk or impact if not addressed"}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Ties to spec objective: [metric]"}]}]}
    ]},
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "2. Action"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "What to build"}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Key implementation detail"}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Out of scope: [exclusion]"}]}]}
    ]},
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "3. Acceptance Criteria"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "AC item"}]}]}
    ]},
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "4. Spec Reference"}]},
    {"type": "paragraph", "content": [
      {"type": "text", "text": "Spec Title", "marks": [{"type": "link", "attrs": {"href": "SPEC_URL"}}]},
      {"type": "text", "text": " — Section X: Section Name, Step reference"}
    ]}
  ]
}
```

---

## Workflow 3: Sync

### Spec to Tickets
Confirm scope (full spec or changed sections), then run Workflow 2 Mode B.

### Jira to Internal System
Fetch tickets via JQL. Extract status, assignee, priority, summary, updated date.
Ask for field mapping on first use. Show mapped payload in Review Mode before pushing.
Push to internal REST API (user provides endpoint and auth token). Report results.

---

## Traceability - Mandatory

- Every ticket description must include the Confluence page URL when a spec exists
- After ticket creation, post keys as footer comment on the Confluence page
- Label spec:<feature-slug> on every spec-linked ticket
- Label section:<section-name> on every breakdown ticket
- If creating ticket without spec link when spec exists: warn user

---

## Failure Handling

On any API failure in Execute Mode:
1. Stop immediately
2. Report: which step failed, error message, partial results
3. Offer: retry, skip and continue, or roll back

---

## Output Principles

- JSON for ticket previews and sync payloads
- Confluence pages for specs, never standalone markdown
- Flag unconfirmed fields as NOT CONFIRMED
- Number tickets and note dependencies
