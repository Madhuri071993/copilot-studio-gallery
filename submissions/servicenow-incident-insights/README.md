# ServiceNow Incident Insights Agent

A **setup-required template**, not a ready-to-run ServiceNow integration.
The ServiceNow Incident Insights Agent supports grounded incident chat and weekly
email reporting through a dual-mode skill. The incident tools and ServiceNow
connection must be added to the **existing agent** in Copilot Studio after import.
No fabricated connector bindings or stub tool definitions are included.

## Included components

- ServiceNow Incident Insights Agent: read-only incident instructions and the
  **ServiceNow Incident Insights Agent** dual-mode skill.
- A weekly recurrence trigger with explicit configuration placeholders.
- The existing Outlook Mail MCP tool for report delivery. This is a mail tool,
  not a ServiceNow incident tool.
- Two calendar agents retain their Outlook capabilities; they are not incident
  data sources or ServiceNow identity resolvers. Review their dependencies before
  enabling them; no calendar tools are added by this migration.

## Required portal setup

Keep recurrence disabled and do not publish for operational use until these
steps and the tests below succeed.

1. Import the unmanaged solution into a development environment. Open the
   existing **ServiceNow Incident Insights Agent** in Copilot Studio.
2. Under **Tools → Add a tool → Connector**, select the official **ServiceNow**
   connector. Add **List Records** (operation `GetRecords`) and **Get Record**
   (operation `GetRecord`) to this agent. UI labels can vary; verify the operation
   IDs against the [official connector reference](https://learn.microsoft.com/en-us/connectors/service-now/).
   Do not add an incident MCP server or paste placeholder tool YAML.
3. Create/select the ServiceNow connection in the portal, supplying your
   `<servicenow-instance>` and your organization's supported authentication.
   Configure credentials only in the connection UI, never in this repository.
   Confirm licensing, environment policy, and least-privilege read access to
   `incident` and its requested fields. Verify which credentials the scheduled
   run actually uses; interactive user authorization does not prove unattended
   execution works.
4. Configure the tool inputs and expose the needed search/pagination inputs:

   | Tool | Inputs |
   | --- | --- |
   | List Records (`GetRecords`) | `tableType`: `incident`; `sysparm_query`: ServiceNow encoded query; `sysparm_fields`: `number,sys_id,short_description,state,priority,assigned_to,sys_created_by,sys_created_on`; explicit `sysparm_limit` and `sysparm_offset` |
   | Get Record (`GetRecord`) | `tableType`: `incident`; `sysid`: an actual returned ServiceNow `sys_id`, **not** a human `INC...` number |

   For a number lookup, List Records with `number=INC0012345`, validate a unique
   match, then use its `sys_id` in Get Record. An absent, forbidden, ambiguous,
   or failed lookup is not an invented record. Filters use **encoded queries,
   not OData**. Validate user-provided filter values/control characters before
   forming a query; reject unsupported filters instead of broadening scope.
5. Replace the following placeholders in **both the agent instructions and the
   recurrence callback**:

   | Placeholder | Maker-provided value |
   | --- | --- |
   | `<scheduled-servicenow-user-sys-id>` | Verified ServiceNow user's `sys_id` for `assigned_to`; not email, alias, display name, or an assumed current user |
   | `<scheduled-report-recipient-email>` | Explicit approved report destination |

   Also configure the ServiceNow scheduled execution connection/identity.
   The target assignee is a filter, not an authentication identity. For chat
   requests about "my" incidents, configure a verified identity mapping or ask
   for an authorized explicit ServiceNow user `sys_id`.
6. Authorize the retained Mail and Copilot Studio connection references.
   Confirm the scheduled execution context can send from the intended mailbox
   to the configured recipient. The retained Mail tool uses invoker
   authentication; do not assume a recurrence provides interactive credentials.
   If unattended authorization is unsupported in your environment, configure a
   supported delivery path in the portal and re-test before enabling recurrence.
7. Set recurrence frequency/interval (default: weekly/1), `<start-time>` and
   `<time-zone>` in the trigger. Validate ServiceNow timestamp/time-zone behavior
   using concrete UTC boundaries. Review instance-specific state/priority choice
   values and configure verified active/resolved mappings in the instructions.
   Without mappings, report raw values and unavailable classifications.
8. Test chat and scheduled execution below; then enable the schedule and publish
   deliberately. Re-export the configured solution if you need a deployable
   package containing the portal-created tool bindings and connection references.

## Weekly scope and report contract

The report covers incidents **created in the past seven days through one captured
run time (UTC), and currently assigned to the configured ServiceNow user**.
It includes all states. It does **not** reconstruct assignment history or report
all incidents ever assigned during that week.

Encoded query template (substitute verified values before use):

```text
assigned_to=<scheduled-servicenow-user-sys-id>^sys_created_on>=<window-start-utc>^sys_created_on<=<run-time-utc>^ORDERBYsys_created_on^ORDERBYsys_id
```

Use concrete timestamps in `yyyy-MM-dd HH:mm:ss`, interpreted/verified as UTC.
Fetch all pages with `sysparm_limit`/`sysparm_offset`, stable ordering, and
`sys_id` deduplication. Continue past a full page to verify the terminal page.
If execution/output limits, unstable data, ACL restrictions, or errors prevent
complete results, report incomplete/failed with unavailable counts, not zero.
Never describe access-filtered results as all incidents.

Normalized incident fields are `number`, `sys_id`, `short_description`, `state`,
`priority`, `assigned_to`, `sys_created_by`, `sys_created_on`. Missing fields use
`"not available in response"`. Preserve returned reference values. **Priority is
not severity**; do not translate between them. State and priority codes/labels
depend on your instance. On-call contacts, assignment history, mitigation history,
and work notes are not provided by these configured fields.

Scheduled output contains:

1. **Executive Summary** (plain text): Overview; Key Metrics; Latest Resolved
   Incidents; Latest Active Incidents; Mitigation Actions Taken; Actionable
   Recommendations. Include retrieval status, UTC window, assignee scope,
   completeness, and unavailable details. Latest lists use newest creation time
   within verified state categories; never fabricate mitigation actions.
2. **Incident List** (one valid JSON array on complete success). These are
   spreadsheet-ready data and Word-style text, not actual Office attachments.

Only complete successful retrieval can report counts or genuine zero records
(`[]`), and a verified zero-record run still sends a summary. Missing setup uses
`setup_required`, retrieval errors/ambiguous scheduled scope use `failed`, and
partial/truncated retrieval uses `incomplete`. Each non-success result uses JSON
`null` for Incident List and unavailable counts, with an explanatory summary.
Email that failure summary only if an approved recipient and working Mail
connection are configured; otherwise the maker must inspect the failed run.
Verify Mail success separately: retrieval success does not prove delivery.
These are instruction-level behaviors, not a new deterministic workflow; confirm
them in run traces and add portal orchestration if stronger guarantees are needed.

## Synthetic setup and acceptance tests

Use a non-production instance, a dedicated test user, and a test mailbox. For
example, `INC0099001` and `11111111111111111111111111111111` may represent a
**synthetic** incident, with assignee `22222222222222222222222222222222`.
These are not real identifiers or credentials: use actual IDs returned by your
test instance. For a fixed test run at `2026-10-20 00:00:00` UTC, the start is
`2026-10-13 00:00:00` UTC.

Create test records with known instance state/priority values:

- A record created inside the window and currently assigned to the test user.
- A record created before the window but assigned this week (must be excluded).
- A record created inside the window but now assigned to another user (excluded).
- Records exactly at both inclusive boundaries; an empty scope; more records
  than one configured page; and a record with an unavailable optional field.

**Chat checks**

- Ask "What is the state of INC0099001?" Verify number resolution by List Records,
  then Get Record using the returned `sys_id`; never pass the number as `sysid`.
- Ask for "my incidents" without a mapping: expect clarification, not a guessed
  email/sys_id. Ask for severity or on-call contacts: expect the limitation,
  not priority substitution or fabricated contacts.
- Request a count and a full JSON export spanning multiple pages. Check all
  matching records, normalized fields, missing-field markers, and no omitted
  rows or false total. The full export is one valid array.
- Remove a tool/connection or deny read permissions: expect setup_required or a
  failure explanation, not zero incidents.

**Scheduled checks**

- Run manually in the portal using an explicit test identity and approved test
  recipient. Verify the UTC window/current-assignee filter in tool traces, both
  structured outputs, and actual email receipt without a chat reply.
- Run a genuine empty query: expect success, zero counts, `[]`, and a sent email.
- Leave an identity/recipient placeholder unresolved: expect setup_required,
  unavailable counts, JSON `null`, and no guessed destination.
- Simulate an expired connection, denied table/field permission, malformed query,
  or a later-page failure/run limit: expect failed/incomplete, unavailable counts,
  JSON `null`, and a failure email only when delivery is configured.
- Deny Mail permission: verify a delivery failure is visible in the run; no
  successful-delivery claim. Check unfamiliar state codes and unavailable fields
  remain raw/unavailable rather than being invented.

The repository checks validate packaging and gallery rendering, not live
ServiceNow access, Copilot Studio import/publish, connector execution, unattended
authentication, or mail delivery. Those acceptance tests require portal access.

## Schema identity migration

The incident agent schema is now `Default_ServiceNow_Incident_Insights_5c2d38`;
its skill schema suffix is `.skill.servicenow-dual-mode-response_UF9`.
Component directories, parent links, content-type overrides, connection reference
logical/display names, and their asset relationships use these same identities.
All pre-migration GUID values and uniqueness suffixes are preserved verbatim.
The two neutral calendar agent schemas and their configurations are unchanged.

This is an explicitly approved local **schema identity migration**, not just a
display-name change. Power Platform may treat renamed schemas as different
components even with GUIDs preserved. A real development-environment import test
is required to verify component identity, ownership, connection authorization,
and behavior before use; local packaging checks do not prove import or runtime
compatibility. No ServiceNow connector bindings are invented by this migration.
The gallery slug is `servicenow-incident-insights`; solution, agent, and
skill display names use **ServiceNow Incident Insights Agent**.
The old incident MCP component, both incident connection
references, their asset relationship, and content-type override are removed.
