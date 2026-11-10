# Spacelift drift remediation

Every morning this flow looks for Spacelift stacks with a run waiting for confirmation, works out which of those runs are drift reconciliation runs, opens a Jira ticket for each one, and posts a summary to Slack with a Confirm button. Confirming from Slack confirms the run in Spacelift. When the run finishes, the result goes back to the Slack thread.

Definition: [`spacelift-drift-remediation.yaml`](spacelift-drift-remediation.yaml)

**This one is a layout, not a working flow yet.** The structure is complete, but several required fields are empty and a few expressions reference blocks that do not exist. See [Known gaps](#known-gaps) before you import it expecting it to run.

## Triggers

- **Morning Schedule**: cron `0 8 * * *` in UTC, so 08:00 UTC every day. Starts stages 1 to 4.
- **On Run Finished**: Spacelift run events, fires when any run reaches a terminal state. Starts stage 5.

## How it works

The canvas groups the 19 blocks into five colored stages with a note on each. The same stages, in order:

### 1. Discover stacks

**Unconfirmed Stacks** runs a `searchStacks` GraphQL query against Spacelift for stacks in the `UNCONFIRMED` state. **Has Next Page?** checks `pageInfo.hasNextPage` and loops back to fetch the next page. **Collect Pages** gathers the pages for that morning's run (grouped on the schedule time), emits once there is no next page, and flattens the results into one list of stacks.

### 2. Inspect each stack

**Each Stack** (Explode, up to 1000 items) emits one event per stack. **Run Details** queries the stack's blocking run: type, state, the `driftDetection` flag and the planned resource changes. **Is Drift Run?** sends drift detection runs to stage 3's ticket and everything else straight to the collector. The **Transform** block in this group is not linked to anything.

### 3. Ticket and aggregate

**Create Drift Ticket** opens a Jira issue for each drift run (stack name, drifted resources, run link). **Collect Results** waits until every stack from stage 2 has either a ticket or a skip, then emits the full list. **Any Drift?** stops the flow here if nothing drifted.

### 4. Slack summary and confirm

**Post Drift Summary** posts an interactive message with a Confirm button per stack. When someone clicks it, **Remember Confirmation** records the click in flow memory, **Confirm Reconciliation Run** confirms the run in Spacelift, and **Reply Confirmed** replies in the thread.

### 5. Report run result

**On Run Finished** fires for every finished run. **Lookup Confirmation** checks memory for a Slack confirmation of that run, **Confirmed From Slack?** drops runs that were not confirmed through the button, and **Reply With Result** posts the final result to the Slack thread.

## What it needs

| Requirement | Detail |
|---|---|
| App installations | `jira` (Jira 0.5.1) and `slack` (Slack 0.2.2). Use these keys or re-point the blocks after import. |
| Spacelift account link | Unconfirmed Stacks, Run Details, Confirm Reconciliation Run and On Run Finished are Spacelift Deploy blocks. The Flows organization has to be linked to a Spacelift account; an account admin starts the link from Spacelift. |
| Secrets | None. |
| Data tables | None. Confirmation state uses flow memory, collection `default`. |
| Slack | A channel for the summary (the description says `#platform-ops`) with the Slack app in it. |
| Jira | A project key and issue type for drift tickets. |

## How to run and test it

1. Create a flow and paste the YAML onto the empty canvas.
2. Point the `jira` and `slack` blocks at your installations and confirm the Spacelift account link.
3. Fill in the config listed under [Known gaps](#known-gaps) and fix the three expressions. Work until the Errors panel is empty.
4. Set up a test stack with autodeploy off and drift detection with reconciliation on, then change something outside of IaC. The reconciliation run waits for confirmation, which is what stage 1 looks for. To avoid waiting for 08:00 UTC, set the cron a few minutes ahead.
5. Check the Slack summary, click Confirm on one stack, and watch for the thread reply when the run finishes.

## Known gaps

Checked 2026-11-09 against a copy of this flow in a Flows project and reading the Errors panel. Nothing here changes the flow's logic; it is the list of what to finish.

**Expressions that point at blocks that do not exist**

- **Collect Results** groups and counts on `outputs.explode.groupKey` and `outputs.explode.count`. The Explode block is named "Each Stack", so these resolve to nothing. They should read `outputs.eachStack.groupKey` and `outputs.eachStack.count`.
- **Remember Confirmation** and **Lookup Confirmation** key on `outputs.block.key`. No block is named "Block". The key needs to be something both sides know, such as the run ID.
- **Remember Confirmation** stores a counter (`!stored ? 1 : stored + 1`), which also fails type checking. Its description says it stores the Slack thread, which is what stage 5 needs to reply in the right place.

**Inputs that are not wired**

- **Unconfirmed Stacks** never sets the required `input` variable, which is where the `UNCONFIRMED` filter and the page cursor go. Without the cursor, Has Next Page? would refetch the same page.
- **Run Details** has no `stack` or `run` variables.

**Required fields left empty**

- Create Drift Ticket: issue type, project key, summary.
- Post Drift Summary: channel, message blocks.
- Reply Confirmed and Reply With Result: channel, message text.
- Confirm Reconciliation Run: stack ID.

The three Condition blocks (Is Drift Run?, Any Drift?, Confirmed From Slack?) also have no condition expression in the YAML. The Errors panel does not flag them, but the routing their descriptions describe is not written yet.

With the schedule enabled the flow fires every morning, and none of these runs gets past Unconfirmed Stacks.
