---
name: doug-jira
description: Pull up Doug's INFERENG Midstream kanban board, triage issues, create/close/comment on Jira tickets, and plan work. Use when Doug asks to check his board, review in-progress work, create or close Jira issues, do standup prep, or plan a burn-down day. Triggers on mentions of "my board", "kanban", "jira", "in progress", "review", "standup", "burn down", or specific INFERENG ticket keys.
---

# Doug's Jira Board Skill

Interactive skill for working with Doug's INFERENG Midstream kanban board via the `jira` CLI. Covers board review, issue triage, creation, commenting, linking, transitions, and work planning.

## Board Context

- **Project:** INFERENG
- **Board:** #2672 "INFERENG Midstream Team" (kanban, no sprints)
- **Board filter:** component = `INFERENG Midstream` — this is critical. A plain `jira issue list` by assignee returns issues from ALL components across INFERENG, which is way more than what shows on the kanban board.
- **User:** dosmith@redhat.com (`jira me` returns this)
- **Jira instance:** https://redhat.atlassian.net

## Pulling Up the Board

Always filter by component to match what the kanban board actually shows:

```bash
# In Progress — the active work column
jira issue list --assignee "$(jira me)" -s "In Progress" \
  -C "INFERENG Midstream" \
  --plain --no-headers --columns KEY,STATUS,SUMMARY,PRIORITY

# Review column
jira issue list --assignee "$(jira me)" -s "Review" \
  -C "INFERENG Midstream" \
  --plain --no-headers --columns KEY,STATUS,SUMMARY,PRIORITY

# New / backlog
jira issue list --assignee "$(jira me)" -s "New" \
  -C "INFERENG Midstream" \
  --plain --no-headers --columns KEY,STATUS,SUMMARY,PRIORITY
```

Without `-C "INFERENG Midstream"`, you'll pull in issues from other teams' components (like "internal process" sub-tasks) that don't appear on Doug's board. Always use the component filter.

## Issue Types on This Board

The board has a mix of:
- **Epics** — long-running initiatives (e.g. "Midstream/CI-SIG: Phase 1", "Tooling for 0-day release", 0-day codename releases). These are ongoing umbrellas, not single-day items.
- **Stories** — concrete deliverables (e.g. "0-day build: DeepSeek V4")
- **Bugs** — defects, often blockers (e.g. "[ROCm] vLLM fails to start with...")
- **Sub-tasks** — children of stories/epics

When reviewing the board, break things out by type so epics don't clutter the actionable work list. To check types, read the first line of `jira issue view <KEY> --plain` output.

## Available Status Transitions

Statuses on this board: `New`, `Backlog`, `In Progress`, `Review`, `Closed`

The kanban board column labeled **"To Do"** maps to the Jira status `Backlog`. Use `-s "Backlog"` to query it.

```bash
# Move an issue
jira issue move INFERENG-XXXX "In Progress"
jira issue move INFERENG-XXXX "Closed"

# Query the "To Do" column
jira issue list --assignee "$(jira me)" -s "Backlog" \
  -C "INFERENG Midstream" \
  --plain --no-headers --columns KEY,TYPE,STATUS,SUMMARY,PRIORITY
```

Note: the status name must match exactly — "Done" doesn't exist, use "Closed". "In Review" doesn't exist, use "Review". "To Do" doesn't exist as a status, use "Backlog". "Refinement" doesn't exist on this board.

## Creating Issues

Use a temp file or heredoc for the description body. Never use `$'...\n...'` — it renders literal `\n` in Jira.

Use **plain markdown** (not Jira wiki markup). The CLI converts markdown to ADF automatically. Use `###` headings, `**bold**`, `[text](url)` links, and standard markdown tables.

```bash
# Write description to temp file
cat > /tmp/jira-desc.md << 'EOF'
## Background

Description in plain markdown.

## Plan

1. Step one
2. Step two

| Column | Value |
|--------|-------|
| PR | [link text](https://github.com/...) |
EOF

# Create the issue
jira issue create \
  -p INFERENG \
  -t Bug \
  -s "Summary here" \
  -y Major \
  -C "INFERENG Midstream" \
  -a "dosmith@redhat.com" \
  -T /tmp/jira-desc.md \
  --no-input
```

Common issue types: `Bug`, `Story`, `Epic`, `Sub-task`
Common priorities: `Blocker`, `Critical`, `Major`, `Normal`, `Minor`, `Undefined`

Always assign to `dosmith@redhat.com` unless told otherwise. Always set component to `INFERENG Midstream` so it appears on the board.

## Commenting

Write comment body to a temp file first. Do NOT use inline heredocs with `jira issue comment add` — it hangs. The `-T` (template) flag reads from the file.

```bash
cat > /tmp/jira-comment.md << 'EOF'
### Comment heading

Comment body in plain markdown.
EOF

jira issue comment add INFERENG-XXXX -T /tmp/jira-comment.md --no-input
```

## Epic Link (customfield_10014)

To make an issue a **child** of an epic, set `customfield_10014` via REST. This is NOT the same as using `jira issue link` — link types like "Incorporates" create a regular link that won't show the issue as a child on the epic's board view.

```bash
# Set epic link — makes INFERENG-XXXX a child of the epic
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -u "${JIRA_USERNAME}:${JIRA_API_TOKEN}" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  -d '{"fields":{"customfield_10014":"INFERENG-9000"}}'
```

**Common mistake:** using `jira issue link INFERENG-XXXX INFERENG-9000 "Incorporates"` instead. That creates a visible link but does NOT set the epic parent. Always use the REST field.

You can set this at creation time too, but the `jira` CLI doesn't support `--custom` for this field reliably, so always follow up with the REST call after `jira issue create`.

## Linking Issues

```bash
# Available link types: Related, Blocks, Depend, Duplicate, Triggers, Causality,
# Cloners, Document, Incorporates, Informs, Issue split
jira issue link INFERENG-1111 INFERENG-2222 "Related"
jira issue link INFERENG-1111 INFERENG-2222 "Blocks"
```

## Reading Issues

```bash
# Summary view
jira issue view INFERENG-XXXX --plain

# With comments (last N)
jira issue view INFERENG-XXXX --comments 10

# Just the header line gives you type, status, assignee, priority
jira issue view INFERENG-XXXX --plain 2>&1 | head -3
```

## Work Planning / Burn-Down Prep

When Doug asks for a work plan or standup prep:

1. Pull In Progress and Review columns (with component filter)
2. Separate epics from stories/bugs — epics are background, stories/bugs are actionable
3. Sort actionable items by priority (Blocker > Critical > Major > rest)
4. Identify quick wins (doc feedback, small PRs) vs deep work (blockers, investigations)
5. Suggest a time-boxed plan: quick wins morning, deep work midday, triage afternoon

## Bulk Operations

For cleaning up stale tickets (e.g. old sub-tasks from other teams):

```bash
for ticket in INFERENG-1111 INFERENG-2222 INFERENG-3333; do
  jira issue move "$ticket" "Closed"
done
```

## Team Field (customfield_10001)

The INFERENG team is stored in `customfield_10001`. The ID for INFERENG Midstream is `ec74d716-af36-4b3c-950f-f79213d08f71-1602`.

**The `jira` CLI `--custom` flag does NOT work for this field.** It warns "not configured" and silently drops the value. Use the Jira REST API instead, immediately after creating every issue:

```bash
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -u "${JIRA_USERNAME}:${JIRA_API_TOKEN}" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX" \
  -d '{"fields":{"customfield_10001":"ec74d716-af36-4b3c-950f-f79213d08f71-1602"}}'
```

Without this, issues won't appear correctly on the kanban board.

## Changing Issue Status

**Do NOT use `jira issue edit -s` to change status.** The `-s` flag on `edit` sets the **summary (title)**, not the status. It will silently overwrite the issue title with whatever you pass. This is a destructive footgun.

`jira issue move` knows the right statuses but launches an interactive picker that hangs in non-interactive mode.

**Use the REST transitions API instead:**

```bash
# Move an issue to Closed (transition ID 61)
curl -s -X POST \
  -H "Content-Type: application/json" \
  -u "${JIRA_USERNAME}:${JIRA_API_TOKEN}" \
  "https://redhat.atlassian.net/rest/api/3/issue/INFERENG-XXXX/transitions" \
  -d '{"transition":{"id":"61"}}'

# Transition IDs for this board:
# 11: New
# 21: Refinement
# 31: Backlog
# 41: In Progress
# 51: Review
# 61: Closed
```

## Gotchas

- **Jira CLI version:** must be v1.7.0+ — v1.6.0 uses a deprecated search API (`/rest/api/2/search`) that returns 410 Gone
- **Component filter is essential:** without `-C "INFERENG Midstream"`, results include tickets from other components that aren't on the kanban board
- **Status names are exact:** "Closed" not "Done", "Review" not "In Review"
- **Markdown not wiki:** `[text](url)` not `[text|url]`, `###` not `h3.`
- **No `-s "In Review"`:** the status is just `Review`
- **Comments hang with heredocs:** always use temp file + `-T` flag
- **`jira issue edit -s` is SUMMARY not STATUS:** it overwrites the title. Use REST transitions API to change status (see above)
