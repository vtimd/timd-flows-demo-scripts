# MCP gate for Spacelift

This flow turns Flows into an MCP server that gives an AI agent three Spacelift tools. Two are read-only and answer right away. The third, `trigger_run`, starts a tracked run only after a person approves it in Slack. Each call that completes, approved or rejected, is recorded in an audit table.

Definition: [`ai-agent-mcp-gate-spacelift.yaml`](ai-agent-mcp-gate-spacelift.yaml)

## Trigger

The three MCP Tool blocks. Copy the server URL from the flow menu ("Copy MCP server URL") and add it to the agent's MCP client config. Each tool call emits an event carrying the agent's `arguments` and the calling `user` (email and project role); the reply goes back to the agent when the event returns to the same tool block.

## The tools

| Tool | Arguments | What the agent gets back |
|---|---|---|
| `list_stacks` | none | Every stack with id, name, state, branch, repository, space and labels |
| `get_stack_status` | `stack_id` (required) | Stack state, the latest run, and a drift status: `in_sync`, `drifted`, `check_in_progress`, `check_failed` or `unknown` |
| `trigger_run` | `stack_id` and `reason` (required), `agent_name` (optional) | `approved` with the run ID and state, or `rejected` with a message saying no run was started |

## How it works

The canvas has a gray overview note and one colored backdrop per tool, each with a description note beside it.

### list_stacks

**Fetch Stacks** runs a `stacks` GraphQL query. **Log List Stacks** writes an audit row, and the tool returns `{ stacks }`.

### get_stack_status

**Fetch Stack Status** queries the stack with its runs, including `driftDetection` and the resource delta. **Summarize Status** picks the newest run and the newest drift detection run. Drift status comes from that drift run: still running is `check_in_progress`, failed is `check_failed`, any add, change or delete is `drifted`, otherwise `in_sync`. No drift run at all is `unknown`, and a missing stack returns `found: false`. **Log Stack Status** writes the audit row before the reply goes back.

### trigger_run

**Request Approval** posts a Slack message with the agent name, the calling user, the stack, the run type (TRACKED) and the agent's stated reason, plus Approve and Reject buttons. **Check Decision** passes only when the clicked button's `action_id` is `approve`.

- Approve: **Log Approved** records the decision and who made it, then **Start Run** triggers a TRACKED run on the stack and the tool returns the run ID.
- Anything else: **Log Rejected** records it and the tool tells the agent a person rejected the request.

The agent's call stays open until someone clicks a button.

## What it needs

| Requirement | Detail |
|---|---|
| App installations | `slack` (Slack 0.2.2). |
| Spacelift account link | Fetch Stacks, Fetch Stack Status and Start Run are Spacelift Deploy blocks. The Flows organization has to be linked to a Spacelift account; an account admin starts the link from Spacelift. |
| Secrets | `SLACK_PLATFORM_APPROVALS_CHANNEL_ID`: the channel or user ID that receives approval requests (the block description says `#platform-approvals`). Invite the Slack app to that channel. |
| Data table | `MCP Tool Call Audit Log` (key `mcptoolcallauditlog`). Columns: `tool`, `caller`, `inputs` (JSON), `decision`, `decided_by`. It is defined in the YAML and comes in with the import. |

## How to run and test it

1. Create a flow, paste the YAML onto the empty canvas, and point the Slack blocks at your installation.
2. Add the secret and confirm the Spacelift account link. The Errors panel should be empty.
3. Copy the MCP server URL from the flow menu and add it to an MCP client.
4. Ask the agent to list stacks, then to check one stack's status. Both answer directly, and each adds a row to the audit table with decision `n/a (read-only)`.
5. Ask the agent to trigger a run on a test stack. Approve it in Slack and confirm the run starts in Spacelift. Repeat and reject it; the agent gets the rejection message and no run starts.
6. Open the data table and check that each call has a row with the caller and, for `trigger_run`, who decided.

## Known gaps

Checked 2026-11-09 against a copy of this flow in a Flows project. The flow matches this file block for block. The Errors panel shows only the missing `SLACK_PLATFORM_APPROVALS_CHANNEL_ID` secret and the unlinked Spacelift account; every expression resolves.

- The approval message is never updated after a click. The Slack block emits an event for every interaction, so a second click on Approve sends the request through Check Decision again and starts a second run. For a gate, the message should be updated or the buttons removed once someone decides.
- The audit table only records calls that get far enough. The read-only tools write their row after the Spacelift query succeeds, and `trigger_run` writes its row after someone clicks a button. A failed query or an approval nobody answers leaves no row, and Log Approved writes before Start Run, so a run that fails to start still shows as approved. Don't treat the table as a complete record of tool calls.
- The flow has no timeout for approvals. If nobody clicks, the agent's call waits for as long as its MCP client allows.
