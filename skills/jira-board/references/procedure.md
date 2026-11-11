---
name: doug-jira
description: Pull up Doug's INFERENG Midstream kanban board, triage issues, create/close/comment on Jira tickets, and plan work. Use when Doug asks to check his board, review in-progress work, create or close Jira issues, do standup prep, or plan a burn-down day. Triggers on mentions of "my board", "kanban", "jira", "in progress", "review", "standup", "burn down", or specific INFERENG ticket keys.
---

# Doug's Jira Board Skill

Interactive skill for working with Doug's INFERENG Midstream kanban board via
the Atlassian CLI (`acli`). Covers board review, issue triage, creation,
commenting, linking, transitions, and work planning. Ported from the retired
go-jira procedure on 2026-11-11; all board-specific knowledge is preserved.

The generic `acli` reference (command tables, ADF node catalog, INFERENG custom
fields) lives in the team skill `~/.openclaw/workspace/skills/atlassian-jira/`.
This file records what is specific to Doug's board on top of that.

## CLI and Auth

- `acli` lives at `~/.local/bin/acli` and is already authenticated (api_token)
  at `https://redhat.atlassian.net` as `dosmith@redhat.com`.
- Verify with `acli jira auth status` before a session's first mutation.
- API tokens expire after ~3 months; if acli starts returning 401, re-auth is
  needed (`acli jira auth login`, token from
  https://id.atlassian.com/manage-profile/security/api-tokens).
- **REST calls need `.bashrc` creds:** the Gateway process environment carries a
  STALE `JIRA_API_TOKEN` (401s). Always run REST through a shell that sources
  the valid pair:
  `bash -c 'source ~/.bashrc; curl -s -u "$JIRA_USERNAME:$JIRA_API_TOKEN" ...'`
- The old `jira` binary (go-jira 1.7.0, `/usr/local/bin/jira`) is vestigial.
  Do not use it for board work anymore.

## Board Context

- **Project:** INFERENG
- **Board:** #2672 "INFERENG Midstream Team" (kanban, no sprints)
- **Board filter:** component = `INFERENG Midstream` — this is critical. A
  plain assignee search returns issues from ALL components across INFERENG,
  which is way more than what shows on the kanban board.
- **User:** dosmith@redhat.com
- **Jira instance:** https://redhat.atlassian.net

## Pulling Up the Board

Always filter by component to match what the kanban board actually shows:

```bash
# In Progress — the active work column
acli jira workitem search \
  --jql "project = INFERENG AND assignee = currentUser() AND status = 'In Progress' AND component = 'INFERENG Midstream'" \
  --limit 20 --fields "key,issuetype,status,priority,summary"

# Review column
acli jira workitem search \
  --jql "project = INFERENG AND assignee = currentUser() AND status = 'Review' AND component = 'INFERENG Midstream'" \
  --limit 20 --fields "key,issuetype,status,priority,summary"

# New / backlog (the "To Do" board column maps to status Backlog)
acli jira workitem search \
  --jql "project = INFERENG AND assignee = currentUser() AND status = 'Backlog' AND component = 'INFERENG Midstream'" \
  --limit 20 --fields "key,issuetype,status,priority,summary"
```

Without `AND component = 'INFERENG Midstream'` in the JQL, you'll pull in
issues from other teams' components that don't appear on Doug's board. Always
use the component filter.

Useful variants:

```bash
# Count only (sizing a column)
acli jira workitem search \
  --jql "project = INFERENG AND assignee = currentUser() AND status = 'New' AND component = 'INFERENG Midstream'" \
  --count

# JSON output for scripted processing
acli jira workitem search --jql "..." --json --limit 50
```

## Issue Types on This Board

The board has a mix of:

- **Epics** — long-running initiatives (e.g. "Midstream/CI-SIG: Phase 1",
  "Tooling for 0-day release", 0-day codename releases). These are ongoing
  umbrellas, not single-day items.
- **Stories** — concrete deliverables (e.g. "0-day build: DeepSeek V4")
- **Bugs** — defects, often blockers (e.g. "[ROCm] vLLM fails to start with...")
- **Sub-tasks** — children of stories/epics

When reviewing the board, break things out by type so epics don't clutter the
actionable work list. Include `issuetype` in `--fields` (it is in the default
field set) or read it from the view output.

## Available Status Transitions

Statuses on this board: `New`, `Backlog`, `In Progress`, `Review`, `Closed`

The kanban board column labeled **"To Do"** maps to the Jira status `Backlog`.

```bash
# Move an issue (no interactive picker; --yes skips confirmation)
acli jira workitem transition --key INFERENG-XXXX --status "In Progress" --yes
acli jira workitem transition --key INFERENG-XXXX --status "Closed" --yes

# Bulk — comma-separated keys or JQL
acli jira workitem transition --key "INFERENG-1111,INFERENG-2222" --status "Closed" --yes --ignore-errors
acli jira workitem transition --jql "project = INFERENG AND component = 'INFERENG Midstream' AND key IN (INFERENG-1111, INFERENG-2222)" --status "Closed" --yes
```

Note: the status name must match exactly — "Done" doesn't exist, use "Closed".
"In Review" doesn't exist, use "Review". "To Do" doesn't exist as a status, use
"Backlog". "Refinement" doesn't exist on this board.

## Formatting: ADF, Not Markdown

**Jira Cloud does not render Markdown or wiki markup.** `##`, `**bold**`,
`- item`, and `| tables |` appear as literal characters in descriptions and
comments. This is the opposite of the old go-jira convention (which
auto-converted markdown).

- Write descriptions and comments as **ADF JSON** and pass them via
  `--description-file` / `--body-file`.
- The ADF node/mark catalog with copy-paste-ready examples is in the team
  skill: `~/.openclaw/workspace/skills/atlassian-jira/references/adf-format.md`.
  Read it before authoring ADF for the first time in a session.
- Validate JSON before sending:
  `python3 -c 'import json,sys; json.load(open(sys.argv[1]))' file.json`
- Only exception: a truly trivial one-liner with no headings, lists, tables,
  code, links, or emphasis may use plain text.

## Creating Issues

Build a creation JSON file with summary, project, type, assignee, ADF
description, and the board fields in `additionalAttributes`:

```bash
cat > /tmp/jira-create.json << 'EOF'
{
  "summary": "Summary here",
  "projectKey": "INFERENG",
  "type": "Story",
  "assignee": "dosmith@redhat.com",
  "parent": "INFERENG-9000",
  "description": {
    "type": "doc", "version": 1,
    "content": [
      {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Background"}]},
      {"type": "paragraph", "content": [{"type": "text", "text": "Description body."}]},
      {"type": "paragraph", "content": [
        {"type": "text", "text": "PR: "},
        {"type": "text", "marks": [{"type": "link", "attrs": {"href": "https://github.com/org/repo/pull/1"}}], "text": "link text"}
      ]}
    ]
  },
  "additionalAttributes": {
    "priority": {"name": "Major"},
    "components": [{"name": "INFERENG Midstream"}],
    "customfield_10001": "ec74d716-af36-4b3c-950f-f79213d08f71-1602"
  }
}
EOF

python3 -c 'import json; json.load(open("/tmp/jira-create.json"))'  # validate
acli jira workitem create --from-json /tmp/jira-create.json
```

Common issue types: `Bug`, `Story`, `Epic`, `Sub-task`
Common priorities: `Blocker`, `Critical`, `Major`, `Normal`, `Minor`, `Undefined`

`additionalAttributes` is verified to set at create time (2026-11-11, live
test): `priority`, `components`, and the team field `customfield_10001`.

- Always assign to `dosmith@redhat.com` unless told otherwise.
- Always set component `INFERENG Midstream` so it appears on the board.
- Always set team `customfield_10001` to the INFERENG Midstream UUID above, or
  the issue won't appear correctly on the kanban board.
- `--parent "INFERENG-9000"` makes the issue a child of an epic at creation.
  This is the correct epic-parenting mechanism — not issue links.

**Verify after create:** `acli jira workitem view` does NOT return custom
fields (priority, components, team all show as null even when set — verified
2026-11-11). To prove component/team/priority stuck, check via REST:

```bash
bash -c 'source ~/.bashrc; curl -s -u "$JIRA_USERNAME:$JIRA_API_TOKEN" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  | python3 -c "import json,sys; f=json.load(sys.stdin)[\"fields\"]; \
     print(\"components:\", [c[\"name\"] for c in f.get(\"components\") or []]); \
     print(\"priority:\", (f.get(\"priority\") or {}).get(\"name\")); \
     print(\"team:\", (f.get(\"customfield_10001\") or {}).get(\"name\"))"'
```

## Commenting

Write the comment as ADF JSON in a temp file, then attach it:

```bash
cat > /tmp/jira-comment.json << 'EOF'
{
  "type": "doc", "version": 1,
  "content": [
    {"type": "heading", "attrs": {"level": 3}, "content": [{"type": "text", "text": "Comment heading"}]},
    {"type": "paragraph", "content": [{"type": "text", "text": "Comment body."}]}
  ]
}
EOF

acli jira workitem comment create --key INFERENG-XXXX --body-file /tmp/jira-comment.json
```

`--edit-last` modifies the most recent comment by the same author (fix a
mistaken comment instead of stacking a correction).

## Epic Link on Existing Issues

At creation, use `--parent`. To re-parent an existing issue, REST is still the
reliable path (verified pattern from the go-jira era; the parent relationship
is stored in `customfield_10014` / the standard `parent` field):

```bash
bash -c 'source ~/.bashrc; curl -s -X PUT \
  -H "Content-Type: application/json" \
  -u "$JIRA_USERNAME:$JIRA_API_TOKEN" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  -d "{\"fields\":{\"customfield_10014\":\"INFERENG-9000\"}}"'
```

**Common mistake:** using an issue link (`link create --type Incorporates`)
instead. That creates a visible link but does NOT set the epic parent, so the
issue won't show as a child on the epic's board view.

## Linking Issues

```bash
# Discover available link types
acli jira workitem link type

# Create links (--out blocks/comes before --in for directional types)
acli jira workitem link create --out INFERENG-1111 --in INFERENG-2222 --type "Related" --yes
acli jira workitem link create --out INFERENG-1111 --in INFERENG-2222 --type "Blocks" --yes
```

## Reading Issues

```bash
# Summary view
acli jira workitem view INFERENG-XXXX

# Subset of fields
acli jira workitem view INFERENG-XXXX --fields summary,status,assignee,priority

# Machine-readable
acli jira workitem view INFERENG-XXXX --json
```

Remember: custom fields (component, team) don't appear in `view` output — use
REST (pattern above) when you need them.

## Work Planning / Burn-Down Prep

When Doug asks for a work plan or standup prep:

1. Pull In Progress and Review columns (with component filter)
2. Separate epics from stories/bugs — epics are background, stories/bugs are actionable
3. Sort actionable items by priority (Blocker > Critical > Major > rest)
4. Identify quick wins (doc feedback, small PRs) vs deep work (blockers, investigations)
5. Suggest a time-boxed plan: quick wins morning, deep work midday, triage afternoon

## Bulk Operations

For cleaning up stale tickets (e.g. old sub-tasks from other teams), transition
in bulk:

```bash
acli jira workitem transition \
  --jql "project = INFERENG AND component = 'INFERENG Midstream' AND key IN (INFERENG-1111, INFERENG-2222, INFERENG-3333)" \
  --status "Closed" --yes --ignore-errors
```

**There is no delete permission in INFERENG** (verified 2026-11-11:
`acli jira workitem delete` fails with "You do not have permission to delete
issues in this project"). Cleanup always means transitioning to `Closed`.

## Team Field (customfield_10001)

The INFERENG team is stored in `customfield_10001`. The ID for INFERENG
Midstream is `ec74d716-af36-4b3c-950f-f79213d08f71-1602`.

- At create: put it in `additionalAttributes` (see Creating Issues).
- On an existing issue: REST PUT is the reliable path:

```bash
bash -c 'source ~/.bashrc; curl -s -X PUT \
  -H "Content-Type: application/json" \
  -u "$JIRA_USERNAME:$JIRA_API_TOKEN" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  -d "{\"fields\":{\"customfield_10001\":\"ec74d716-af36-4b3c-950f-f79213d08f71-1602\"}}"'
```

Without this, issues won't appear correctly on the kanban board.

## SME Label Triage (ms-* workstreams)

Doug is the SME for the `ms-upstream` label and the newer `ms-omni` label
(omni-specific work). The full workstream label set: `ms-infra`, `ms-release`,
`ms-mv`, `ms-ci`, `ms-modelopt`, `ms-agentic`, `ms-other`, `ms-upstream`,
`ms-omni`.

**One primary workstream label per item.** Pick the single bucket where the
staffing help is needed — e.g. omni-upstream items get `ms-omni`, not
`ms-upstream` — because the triage view groups by label and multi-labeling
makes groups noisy.

**Subject-matter ownership (2026-11-11 rules):**

- "Omni wins for subject matter": omni work that was cut from the release
  lane belongs in `ms-omni`, not `ms-release`.
- **Exception — Tarun's items stay put:** anything Takumar (`takumar@redhat.com`)
  created or is assigned keeps its lane (`ms-ci`, `ms-mv`, and his
  `ms-release` items) — he is his own SME there. Check assignee AND creator
  before moving anything between buckets.
- Moving between buckets is two steps: `--labels "ms-new"` then
  `--remove-labels "ms-old"` (see Label discovery and edits below).

**Always scope label work to the team.** Every SME label query and every
labeling action must include `AND component = 'INFERENG Midstream'`. Omni and
upstream work owned by other teams (e.g. Alex Brooks' non-midstream omni
items) is out of scope — the component filter is the boundary. Verified
2026-11-11: of 21 unlabeled omni items project-wide, exactly 1 carried the
Midstream component.

**Exclude dead weight in working views.** The full label set holds hundreds
of Closed items; triage queries should always carry `AND status not in
(Closed)`.

### The SME triage / staffing queue

```bash
# Items needing assignment — what to bring to "SME Backlog Prep"
acli jira workitem search \
  --jql "project = INFERENG AND labels in (ms-upstream, ms-omni) AND component = 'INFERENG Midstream' AND status in (New, Backlog) AND assignee is EMPTY ORDER BY rank ASC" \
  --limit 50 --fields "key,issuetype,status,priority,summary,labels"
```

- `rank` is `customfield_10019` (the browser list view orders by `cf[10019]
  ASC`). `ORDER BY rank ASC` in JQL matches it. `rank` itself is not
  requestable in `--fields`.
- `labels` is NOT in the default search field set — request it explicitly in
  `--fields` or label data silently won't come back.

### SME orchestration workflow (delegation prep)

Doug's triage pattern as SME — getting work ready to hand off:

1. Sort the working view: `ORDER BY status ASC, priority DESC, cf[10019] ASC`
   — triage-ready first, priority-weighted.
2. Pick items in `New` that Doug hasn't gotten to and won't do himself:
   unassign them and transition them to `Backlog`. **Backlog + unassigned is
   the delegation shelf** — the "ready for pickup" signal.

```bash
acli jira workitem edit --key INFERENG-XXXX --remove-assignee --yes
acli jira workitem transition --key INFERENG-XXXX --status "Backlog" --yes
```

3. What stays with Doug: active Critical work, tracker/coordination stories
   (they are orchestration artifacts, not work items), and
   relationship-driven collaboration items.
4. Delegation quality: strong delegates are crisp, bounded, and verifiable
   (a fix, a build-and-validate, an automation with testable outcome).
   "Evaluate"/"investigate" items need a definition pass first — either
   sharpen the acceptance criteria or keep them parked.

### Label discovery and edits

- **Text search is unreliable for finding workstream items**: JQL `summary ~
  'omni'` does not match "vLLM-Omni" (hyphenation tokenization). Discover
  work by epic children instead: `"Epic Link" = INFERENG-1234` or
  `parent = INFERENG-1234`, then label the stories that need staffing.
- **Add a label:**
  `acli jira workitem edit --key INFERENG-XXXX --labels "ms-omni" --yes`
- **Remove a label:**
  `acli jira workitem edit --key INFERENG-XXXX --remove-labels "ms-omni" --yes`
- **`--labels` APPENDS, it does not replace** (verified 2026-11-11 on acli
  1.3.40: existing labels survive, including ms-* ones). Moving an item
  between workstream buckets therefore takes two steps: `--labels "ms-new"`
  then `--remove-labels "ms-old"`. Never assume a `--labels` call cleared
  anything — verify with a search afterward.
- **Bulk edit by JQL:**
  `acli jira workitem edit --jql "<query>" --labels "ms-omni" --yes --ignore-errors`
  Prefer explicit `--key` lists over broad JQL for anything destructive.
- Labels are also settable at create time via `additionalAttributes`
  (`"labels": ["ms-omni"]`).

## Gotchas

- **ADF not Markdown:** `##` and `**bold**` render as literal characters.
  Descriptions and comments must be ADF JSON (see the team skill's
  `adf-format.md` reference). This reverses the old go-jira convention.
- **`acli jira workitem view` hides custom fields:** component/team/priority
  show as null even when set. Verify via REST after create.
- **No delete permission in INFERENG:** cleanup = transition to `Closed`.
- **Component filter is essential:** without `AND component = 'INFERENG
  Midstream'`, results include tickets from other components that aren't on
  the kanban board.
- **Status names are exact:** "Closed" not "Done", "Review" not "In Review",
  "Backlog" not "To Do".
- **Gateway env token is stale:** REST calls must run through
  `bash -c 'source ~/.bashrc; ...'`. Bare exec-env `JIRA_API_TOKEN` 401s.
- **acli auth is separate from go-jira:** acli stores its own api_token
  (expires ~3 months). If acli 401s, re-auth with `acli jira auth login`.
- **Validate every ADF JSON file** with python3 before sending it.
- **INFERENG priority scheme has no "Low":** the rungs are Blocker, Critical,
  Major, Normal, Minor, Undefined — Minor is the lowest. There is no acli
  flag for priority; use REST PUT `{"fields":{"priority":{"name":"Minor"}}}`
  (with the `.bashrc` creds shell).
- **`parentIssueId` in create `--from-json` is broken** (rejected as both
  string and integer). To parent at create: CLI flags `--parent KEY
  --description-file <adf.json>`, then one REST PUT for component, team,
  priority, and labels.
- **acli `--fields` rejects `created`, `parent`, and `rank`.** Put date
  predicates directly in the JQL instead (`created <= "2025-12-31"`), and
  fetch epic children via `"Epic Link" = KEY` queries.
- **REST `/rest/api/3/search` is removed** (410 Gone). The replacement
  `/rest/api/3/search/jql` may reject payloads ("Invalid request payload"
  even minimal); prefer acli search with JQL predicates.
- **SME label work is component-scoped:** never label, triage, or close work
  outside `component = 'INFERENG Midstream'` — check the component first via
  REST (view hides it).
