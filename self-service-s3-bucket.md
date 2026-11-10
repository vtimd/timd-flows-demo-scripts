# Self-service environment request

**Despite the file name, this flow does not create an S3 bucket.** It takes a request from a ServiceNow catalog item, runs it through two Slack approvals (team lead, then budget owner), and deploys a Spacelift template that its block description calls "Aurora Database Environment". Ephemeral environments are torn down automatically when their TTL runs out.

Definition: [`self-service-s3-bucket.yaml`](self-service-s3-bucket.yaml)

## Trigger

**Request Form**, a ServiceNow catalog item named "Request a New Environment". It is a lifecycle block: it creates the catalog item in ServiceNow, so it imports as a draft and does nothing until you confirm it. The form asks for:

| Field | Rule |
|---|---|
| Project identifier | Required. 3 to 30 characters, lowercase letters, digits and hyphens, starting with a letter. |
| Database engine | `aurora-postgresql` (default) or `aurora-mysql`. |
| Database name | Required, default `appdb`. Letters, digits and underscores. |
| Environment type | `staging` (default), `sandbox` or `ephemeral`. |
| TTL in minutes | Required for `ephemeral`, 1 to 1440, even though the form marks it optional. Ignored for the other types. |
| Justification | Required. |

## How it works

The canvas groups the 15 blocks into four colored stages with a note on each. The same stages, in order:

### 1. Request & validate

**Validate Request** checks the project identifier and database name against regexes and, for ephemeral requests, the TTL range. A bad request goes to **Reject Invalid Request**, which marks the ServiceNow request item `FAILED` with a comment listing the rules. A good one moves on to stage 2.

### 2. Acknowledge & approve

**Request Received** sets the item to `WORK_IN_PROGRESS` with a comment that it is waiting on team lead approval. At the same time, **Team Lead Approval** posts the request to Slack with Approve and Reject buttons. **Team Lead Approved** passes only when the clicked button is Approve. On approval, **Budget Owner Approval** posts a second message to the same channel, noting who gave the first approval, and **Budget Owner Approved** checks that click the same way. A Reject at either level goes to **Notify Rejected**, which posts who rejected the request and at which level.

### 3. Deploy

**Deploy Environment** deploys the Spacelift template from `SPACELIFT_TEMPLATE_ID` into the space from `SPACELIFT_SPACE`. The deployment is named `<project identifier>-<environment type>`, with the project identifier, engine and database name as template inputs. When it completes, **Notify Deployed** posts the outcome to Slack: the template outputs on success, or the final state if it did not finish.

### 4. Ephemeral teardown

**Is Ephemeral** passes when the environment type is `ephemeral` and the deployment finished. **Wait TTL** sleeps for the requested minutes. **Delete Deployment** removes the template deployment and its stacks through the Spacelift GraphQL API, and **Notify Teardown** posts that it is gone.

## What it needs

| Requirement | Detail |
|---|---|
| App installations | `servicenow` (ServiceNow 0.2.0) and `slack` (Slack 0.2.2). |
| Spacelift account link | Deploy Environment and Delete Deployment are Spacelift Deploy blocks. The Flows organization has to be linked to a Spacelift account; an account admin starts the link from Spacelift. |
| Spacelift template | A template that takes `project_identifier`, `engine` and `database_name` as inputs. The template and the cloud credentials its stacks use are outside this flow. |
| Secrets | `SLACK_CHANNEL_ID` (where both approvals and all notices go), `SPACELIFT_TEMPLATE_ID`, `SPACELIFT_SPACE`. |
| Data tables | None. |

## How to run and test it

1. Create a flow, paste the YAML onto the empty canvas, and point the blocks at your ServiceNow and Slack installations.
2. Add the three secrets and confirm the Spacelift account link. The Errors panel should be empty.
3. Confirm Request Form so the catalog item shows up in ServiceNow.
4. Submit a request with a bad project identifier (for example `1-bad`) and check that the request item goes to `FAILED`.
5. Submit a valid `ephemeral` request with a 5-minute TTL. Approve both Slack messages, watch the deployment in Spacelift, and check for the teardown notice about five minutes after it finishes.
6. Submit one more and reject it at the budget owner step. Check that the rejection notice names the budget owner.

## Known gaps

Checked 2026-11-09 against a copy of this flow in a Flows project. The flow matches this file block for block. The Errors panel shows the three missing secrets and the unlinked Spacelift account, nothing else.

- Anyone in the channel can click Approve at either level, and one person can approve both. The conditions check which button was clicked, not who clicked it.
- ServiceNow only hears about validation. After `WORK_IN_PROGRESS` the request item is never set to `COMPLETED` on deploy or to `FAILED` on rejection; those outcomes go to Slack only.
- An `ephemeral` request without a TTL is rejected. The form marks the field optional, but Validate Request treats a missing TTL as 0 and fails it, so the 2-minute fallback in Wait TTL never runs.
- Validation is stricter than the form text. The database name must also start with a letter and be at most 63 characters, and the project identifier cannot end with a hyphen. A request that breaks only those rules gets a rejection comment that does not explain why.
