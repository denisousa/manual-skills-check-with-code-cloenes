<!-- Internet-Service-Development/agent-plugin-test:plugins/F2E2/jira-ticket-scan/skills/SKILL.md | https://github.com/Internet-Service-Development/agent-plugin-test/blob/HEAD/plugins/F2E2/jira-ticket-scan/skills/SKILL.md -->

---
name: jira-ticket-scan
description: "Scan and summarize a person's Jira daily update using Jira MCP tools. Useful for daily standups, personal daily reports, and assignee progress checks. Trigger phrases: jira daily tasks, daily jira summary, assignee daily scan, today Jira task summary."
metadata:
  author: asus-cms
  version: "1.0"
---

# Jira Ticket Scan

This skill summarises a person's Jira daily update, including yesterday's progress, today's plan, blockers, risks, and links. It requires Jira MCP data and should stay grounded in fields that are actually returned. After generating the summary, send it to Teams with `scripts/send-digest-teams.ps1`.

## When to Use

- The user asks to summarize what a specific teammate progressed yesterday and what they should focus on today.
- A personal task summary is needed for a daily standup.
- A quick progress and risk report is needed for a specific assignee.

## MCP Connection Principles

- Always connect to Jira through Jira MCP tools. Do not assume data or use unverified sources.
- Prioritize the following tools:
  - `mcp_atlassian-jir_jira_searchIssues`: Batch JQL search
  - `mcp_atlassian-jir_jira_getIssue`: Fetch detailed fields for a single issue
- Prefer building JQL using `assignee + explicit daily buckets`, then retrieve the issue list.
- For a lightweight daily summary, start with issue search results. Fetch per-issue details only when the search result fields are insufficient for an accurate summary.

## Core Principles

- Use only data returned by Jira MCP. Do not infer fields that are not present.
- Resolve the assignee before retrieving data.
- Next steps must come from explicit issue fields, status, due dates, or user-provided context. Do not invent them.
- Counts must be verifiable. The total count and grouped counts must match.
- If no assignee is provided, assume the assignee is the person invoking the Skill.
- Use the current user’s Jira identity from the MCP/session context when available.
- Do not ask for assignee confirmation unless the current user cannot be resolved.

## Language Policy

- Use the user device/UI locale when available from session context or OS locale probes.
- Do not infer language only from the current message text.
- Treat `C`, `POSIX`, `C.UTF-8`, empty values, or failed OS probes as unknown.
- If locale cannot be determined, use Traditional Chinese (`zh-TW`) as team default.
- Keep the section structure from the template unchanged, but localise labels and bullet text.

## Workflow

### 1. Parse the Request and Input Type

Identify and complete the following inputs:

- Assignee: prioritize accountId, then displayName or email
- Output preference: concise standup version or fuller daily report version

For the daily report, anchor the summary to two explicit windows in the user's timezone or the Jira instance timezone:

- Yesterday: `startOfDay(-1d)` to `startOfDay()`
- Today: `startOfDay()` to `endOfDay()`

### 2. Retrieve Task Data Through MCP

Use JQL to query the assignee’s tickets for the daily report. Work from three explicit buckets.

All issues relevant to the reminder bucket:

```text
assignee = <ASSIGNEE>
AND (
  updated < startOfDay(-1d)
  AND statusCategory != Done
)
ORDER BY priority DESC, updated DESC, duedate ASC
```

This bucket is for issues that were not updated yesterday and are not simply today's planned items. Use it to surface older open work the assignee may have forgotten about.

Yesterday issues: any issue with an update yesterday.

```text
assignee = <ASSIGNEE>
AND updated >= startOfDay(-1d)
AND updated < startOfDay()
ORDER BY priority DESC, updated DESC
```

Today issues: anything due today, plus anything updated yesterday that is not finished.

```text
assignee = <ASSIGNEE>
AND (
  due = startOfDay()
  OR (
    updated >= startOfDay()
    AND updated <= endOfDay()
    AND statusCategory != Done
  )
)
ORDER BY priority DESC, status ASC, duedate ASC, updated DESC
```

If the Jira instance does not support `statusCategory`, use the local equivalent for "not finished", such as unresolved issues or a project-specific done-status list.

### 3. Normalise and Validate Results

- Remove duplicate issues if the retrieval flow returns the same issue more than once.
- Record the issue key, summary, status, priority, updated time, and link for every matched issue.
- Record the assignee email from Jira user fields such as `assignee.emailAddress` for Teams delivery.
- Read the Jira field `Website System` for every matched issue as the default source for website categorisation.
- Record the website category from `Website System`.
- Read Jira issue links for every matched issue and use them to capture related GitLab links when present.
- Record the start date when present.
- Record the end date when present for stage work.
- Record the due date when present for production work.
- If the search result does not contain enough data to classify an issue accurately, fetch that issue’s details.
- If no issues are returned, produce the template with explicit empty-section placeholders.
- If `Website System` is missing from the retrieved fields, fetch issue details before falling back.
- If `Website System` exists but is empty, use the clearest supported project-specific field or label. If it still cannot be determined, group the issue under `Unknown site`.
- Apply final presentation sorting after retrieval:
  - end date first for stage work
  - due date first for production work
  - updated time as the next tiebreaker
  - start date as the final tiebreaker

### 4. Build the Summary

- Group issues inside each section by the `Website System` value first, then by status.
- Use the Jira status when available. Preserve project-specific statuses instead of collapsing them unless the user asks for a simplified mapping.
- Put only reminder-bucket issues into section 0. Do not repeat issues already covered by the yesterday or today sections unless the user explicitly asks for a fully exhaustive report.
- Put yesterday issues into section 1 when Jira shows they had any update yesterday.
- Put today issues into section 2 when they are due today or were updated yesterday and are not finished.
- Put explicit blockers, waiting states, or external dependencies into section 3.
- Put scope changes, delayed items, or technical uncertainty into section 4.
- Put the Jira issue link for every included ticket into section 5, plus any directly referenced GitHub/GitLab PR, MR, commit, or discussion links.
- If a section has no evidence-backed content, keep the section and write `None`.

### 5. Send the Summary to Teams

After formatting the summary from the template, send that exact Markdown summary to Teams by running the bundled script:

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File ./scripts/send-digest-teams.ps1 `
  -TargetUserEmail "<assignee email from Jira>" `
  -DigestTitle "Jira Daily Summary - <assignee> - <date range>" `
  -DigestFile "<path-to-summary-markdown>" `
  -Assignee "<assignee>" `
  -DateRange "<date range>"
```

Use `-DigestFile` when the summary is more than a few lines so Markdown tables and localised text are preserved exactly. If the environment cannot write a temporary summary file, pass the generated summary through `-DigestMarkdown` instead.

Use the resolved Jira assignee email as `-TargetUserEmail`. Prefer the assignee email from the matched Jira issue data. If search results omit the email field, fetch issue details before giving up. If Jira does not expose an email address and no current-user Jira email is available, report that Teams delivery could not be attempted because the recipient email was unavailable.

Keep Teams delivery focused on the Jira digest. Do not pass merge request, review, lint, test, build, status, or sender metadata to this script.

The Power Automate manual trigger and Teams adaptive card must use the Jira digest contract, not the older merge request contract:

- [Teams Request Body Schema](./assets/teams-request-body-schema.json)
- [Teams Adaptive Card](./assets/teams-adaptive-card.json)

If the send fails, report that Teams delivery failed and include the script error. Do not silently fall back to another delivery method unless the user explicitly asks.

Only skip Teams delivery when the user asks for a draft/preview without sending, or when required Jira data could not be retrieved.

## Output Template

Always format the final summary using this template file:

- [Daily Summary Template](./assets/daily-summary-template.md)

When there is no data for a section, keep the section and explicitly write `None`.

## Formatting Rules

- Render issue details as Markdown tables, not per-issue bullets.
- Keep each row concrete and outcome-oriented.
- Use lightweight emojis whenever they improve scanability, for example `✅` for done, `🟡` for in progress or carry over, `⛔` for blockers, and `⚠️` for risks.
- Make the `Ticket` cell itself a clickable Markdown link to the Jira issue for every reported ticket.
- Use Jira issue links as the default source for related GitLab links, and include those links whenever present.
- Within each section, group output by website first and then by status.
- Use the `Website System` field as the primary website grouping field.
- Use headings or labels that preserve the website name and the exact Jira status, for example `ROG site -> In Progress`.
- Indent each table under its website and status grouping so the hierarchy is visually obvious.
- Sort reported tickets by:
  - end date for stage work
  - due date for production work
  - updated time as the next tiebreaker
  - start date as the final tiebreaker
- Include explicit date columns where relevant, especially start date, updated time, and end date or due date.
- Do not add extra columns such as `Why In Scope`, `Evidence`, `Next Action`, `Expected Deliverable`, or `Dependency Or Risk`.
- Keep the summary evidence-based: if Jira data does not show who needs to act, what changed, or what comes next, say that it is unclear.
- Localise placeholders like `None` based on output language.

## Final Response After Teams Delivery

- Confirm whether the Teams digest was sent successfully.
- Include the same formatted summary in the response unless the user explicitly asked to send only.
- If sending failed, include the summary and the exact failure reason so the user can retry.
