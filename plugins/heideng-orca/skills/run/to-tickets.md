# To tickets

Break the spec into a set of **tickets**: tracer-bullet vertical slices, each declaring the tickets that **block** it, plus one **verify ticket** `V`.

`$F` is the scratch dir of SKILL.md. The issue tracker and triage label vocabulary come from the home repo's `## Agent skills` block.

## 1. Gather context

Read the spec in full. If it references an issue (number or URL), fetch it and read its full body and comments.

## 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code. Ticket titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

Look for opportunities to prefactor the code to make the implementation easier. "Make the change easy, then make the easy change."

## 3. Draft vertical slices

Break the work into **tracer bullet** tickets.

<vertical-slice-rules>

- Each slice cuts a narrow but COMPLETE path through every layer (schema, API, UI, tests): vertical, NOT a horizontal slice of one layer
- A completed slice is demoable or verifiable on its own
- Each slice is sized to fit in a single fresh context window
- Any prefactoring should be done first

</vertical-slice-rules>

Give each ticket its **blocking edges**: the other tickets that must complete before it can start. A ticket with no blockers can start immediately.

**One repo per ticket.** Every ticket carries `Repo:`, a name from `repos.md` (SKILL.md, Repos). A slice that crosses repos becomes one ticket per repo, chained by blocking edges in the order the layers depend on each other (API before the UI that calls it). Only the last ticket of the chain proves the slice live; the ones before it prove themselves with build and tests.

Mark the **skeleton**: the first tracer bullet, after which the app launches and one path works end to end. A skeleton that crosses repos is a chain; "skeleton merged" means its last ticket merged.

**Wide refactors are the exception to vertical slicing.** A **wide refactor** is one mechanical change (rename a column, retype a shared symbol) whose **blast radius** fans across the whole codebase, so a single edit breaks thousands of call sites at once and no vertical slice can land green. Don't force it into a tracer bullet; sequence it as **expand-contract**. First expand: add the new form beside the old so nothing breaks. Then migrate the call sites over in batches sized by blast radius (per package, per directory), each batch its own ticket blocked by the expand, keeping CI green batch to batch because the old form still exists. Finally contract: delete the old form once no caller remains, in a ticket blocked by every migrate batch. When even the batches can't stay green alone, keep the sequence but let them share an integration branch that all block a final integrate-and-verify ticket; green is promised only there.

## 4. Draft the verify ticket

Add the **verify ticket** `V` to the breakdown, with `Repo:` the home repo, where `.claude/skills/verify/` lives. It blocks every other agent ticket except the skeleton. The verify skill takes the path of each repo checkout as a parameter and hardcodes none: the same skill drives the system from a ticket's worktree plus the integration worktrees of the other repos.

| Repo state | `V` runs | `V` blocked by |
|---|---|---|
| no `.claude/skills/verify/` | `create-verification-skill` | the skeleton |
| `.claude/skills/verify/` exists | `maintain-verification-skill` on the base branch: its live pass is the **baseline** of what worked before this spec | nothing |

Either way `V` also adds one feature file per acceptance criterion of the spec, marked `pending` until its ticket lands.

Write `V`'s worker spec to `$F/specs/V.md`: the skill it runs, the feature files to add, ship by committing on its branch (no PR), product gaps go in its report.

## 5. Quiz the advisor

Write the proposed breakdown to `$F/tickets/draft.md` as a numbered list, `V` included. For each ticket, show:

- **Title**: short descriptive name
- **Repo**: the repo the ticket changes
- **Blocked by**: which other tickets (if any) must complete first
- **What it delivers**: the end-to-end behaviour this ticket makes work

Ask the advisor:

- Does the granularity feel right? (too coarse / too fine)
- Are the blocking edges correct: does each ticket only depend on tickets that genuinely gate it?
- Should any tickets be merged or split further?
- Does every acceptance criterion of the spec land in a ticket?
- Is the skeleton a true tracer bullet: after it, does the app launch with one path working end to end?
- Does each cross-repo chain run in the order its layers depend on each other, with its last ticket proving the slice live?

The advisor writes its verdict to `$F/tickets/review.md`. Apply the changes and iterate until the advisor approves the breakdown.

## 6. Publish the tickets to the configured tracker

Publish the approved tickets. **How** depends on the configured tracker; the tickets are the same either way, only the shape of the blocking edges changes:

- **Local files** → write one file per ticket under `.scratch/<feature-slug>/issues/<NN>-<slug>.md`, numbered from `01` in dependency order (blockers first). Each file's "Blocked by" lists the numbers/titles it depends on. Use the per-ticket file template below: one ticket per file, never a single combined file.
- **A real issue tracker (GitHub, Linear, ...)** → publish one issue per ticket in dependency order (blockers first) so each ticket's blocking edges can reference real identifiers. Use the platform's native blocking relationship where it has one (older `gh` lacks `--blocked-by`: use the dependencies API from `docs/agents/issue-tracker.md`); otherwise set each ticket's "Blocked by" to the blocking issues. If the source was an existing issue, make each ticket its sub-issue (tracker doc's operation).

Apply the `ready-for-agent` triage label. Tickets needing a real login or GUI get `ready-for-human` and stay outside the implement graph.

Do NOT close or modify any parent issue.

<local-ticket-template>

# <NN>: <Ticket title>

**What to build:** the end-to-end behaviour this ticket makes work, from the user's perspective, not a layer-by-layer implementation list.

**Repo:** the repo name from `repos.md`.

**Blocked by:** the numbers/titles of the tickets that gate this one, or "None (can start immediately)".

**Status:** ready-for-agent

- [ ] Acceptance criterion 1
- [ ] Acceptance criterion 2

</local-ticket-template>

<issue-template>

## Parent

A reference to the parent issue on the tracker (if the source was an existing issue, otherwise omit this section).

## Repo

The repo name from `repos.md`.

## What to build

The end-to-end behaviour this ticket makes work, from the user's perspective, not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2

## Blocked by

- A reference to each blocking ticket, or "None (can start immediately)". Omit this section when blockers were set as native edges.

</issue-template>

In either form, avoid specific file paths or code snippets: they go stale fast. Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it and note briefly that it came from a prototype. Trim to the decision-rich parts, not a working demo, just the important bits.
