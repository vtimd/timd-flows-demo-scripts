# Incident enrichment

When PagerDuty triggers an incident, this flow pulls the service's last five GitHub deployments and its Datadog monitors that are alerting or warning. Claude (Anthropic app) writes a short summary of what changed and what is alerting. The flow then creates an incident channel in Slack, posts the summary there, and adds it as a note on the PagerDuty incident.

Definition: [`incident-enrichment.yaml`](incident-enrichment.yaml)

## Trigger

**Incident Triggered**, a PagerDuty incident subscription filtered to the `triggered` status on one service. It is a lifecycle block: it creates a webhook subscription in PagerDuty, so it imports as a draft and does nothing until you set the service ID and confirm it.

## How it works

Seven blocks in a straight line, no branches. The canvas groups them into four colored stages with a note on each: Trigger (step 1), Gather context (2 and 3), Summarize (4) and Share (5 to 7).

1. **Incident Triggered** emits the PagerDuty incident.
2. **Fetch GitHub Deploys** calls `GET /repos/YOUR_ORG/<service>/deployments?per_page=5` through the GitHub app. `<service>` is the PagerDuty service name, lowercased, with spaces turned into hyphens. That assumes each repository is named after its PagerDuty service.
3. **Fetch Datadog Monitors** lists monitors tagged `service:<service>`, using the same name.
4. **Generate Incident Summary** sends the incident details, the deployments and the monitors in `Alert` or `Warn` state to `claude-sonnet-4-6` (1024 max tokens). The system prompt asks for two plain-text sections, "What changed" and "What's alerting", a closing line on the most likely causal deploy, and a statement when a section has no data rather than invented content.
5. **Create Incident Channel** creates `inc-<incident id>` in Slack.
6. **Post Summary to Slack** posts a link to the incident and the summary in that channel.
7. **Add Summary to PagerDuty** adds the same summary as a note on the incident.

## What it needs

| Requirement | Detail |
|---|---|
| App installations | `anthropic` (Anthropic 0.8.0), `datadog` (Datadog 0.1.2), `github` (GitHub 0.6.0), `pagerduty` (PagerDuty 0.2.0), `slack` (Slack 0.2.2). |
| Secrets | `PAGERDUTY_FROM_EMAIL`: the email of a PagerDuty user. PagerDuty attributes the incident note to this user. |
| Placeholders to replace | `PXXXXXX` in Incident Triggered (your PagerDuty service ID) and `YOUR_ORG` in Fetch GitHub Deploys (your GitHub org). |
| Naming convention | GitHub repositories and Datadog `service:` tags have to match the PagerDuty service name after lowercasing and hyphenating. If they do not, the two calls fail differently. GitHub answers a repository that doesn't exist with a 404, so Fetch GitHub Deploys errors and nothing after it runs for that incident. A Datadog tag that matches nothing just returns no monitors and the flow carries on. An unreplaced `YOUR_ORG` gets the same 404. |
| GitHub deployments | Deploys have to be recorded as GitHub Deployments. Commits and releases alone do not show up. |
| Slack permissions | The Slack app needs permission to create channels. |
| Spacelift account link | Not needed; this flow has no Spacelift blocks. |
| Data tables | None. |

## How to run and test it

1. Create a flow, paste the YAML onto the empty canvas, and point each app block at your installations.
2. Add the `PAGERDUTY_FROM_EMAIL` secret and replace both placeholders. The Errors panel should be empty.
3. Confirm Incident Triggered so Flows creates the PagerDuty webhook subscription.
4. Trigger a test incident on that PagerDuty service.
5. Check for a new `inc-<id>` channel with the summary, and for the same text as a note on the incident.

## Known gaps

Checked 2026-11-09 against a copy of this flow in a Flows project. The flow matches this file block for block, and the Errors panel shows only the missing `PAGERDUTY_FROM_EMAIL` secret. The placeholders do not show as errors because they are valid strings, so the flow looks ready before it is.

- Nobody gets invited to the new incident channel. Responders have to find it and join.
- Every block waits on the one before it and there is no error handler. If any block errors, nothing after it runs for that incident, including the Slack channel.
