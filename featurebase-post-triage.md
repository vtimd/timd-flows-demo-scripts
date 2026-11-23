# Featurebase post triage

When someone creates a post in Featurebase, this flow opens a Slack thread for it, gathers product context from GitHub, and has Claude (Anthropic app) act as a deliberately skeptical product manager. Claude classifies the post, picks the owning engineering team, scores whether it is worth building, and drafts a customer-facing reply. Posts that are clearly spam are deleted. Everything else is assigned to the owning team's Featurebase admin, and the triage result is posted to the Slack thread with a ping to that team's PM. A second trigger mirrors new Featurebase comments into the same thread.

Definition: [`featurebase-post-triage.yaml`](featurebase-post-triage.yaml)

## Triggers

Two Featurebase event subscriptions. Both are lifecycle blocks: they create a webhook in Featurebase, so they import as drafts and do nothing until you confirm them.

- **When post created** starts the triage path.
- **Comment Created** starts the comment sync path.

## How it works

32 blocks in two independent paths. The canvas groups them into stages with a note on each, plus a Configuration note and a Memory and errors note.

**Intake**

1. **When post created** emits the new post.
2. **Get Post** fetches the full post, **Resolve custom fields** turns custom field IDs and option IDs into labels, and **Markdown content** converts the HTML body to Markdown.
3. **Start a Slack thread** posts "New Featurebase ticket" with a link to your Featurebase portal, the author and the post body. **Store FB-Slack mapping** saves the post ID to Slack `ts` pair in flow memory (collection `fb-slack-mapping`).
4. **Prepare ticket** builds the ticket object (id, slug, title, Markdown content, custom fields, author) that goes to the model.

**Context and triage**

5. Five GitHub **Fetch** blocks run in sequence and read files from your context repository on `main` (placeholder `YOUR_ORG/your-context-repo`, folder `triage/context`): team scopes, PM principles, current work, capabilities, and `team-routing.json`.
6. **Team routing** decodes `team-routing.json` once. It supplies the team names for the model's schema enum (falling back to `Platform`, `Integrations`, `Enterprise`), the Slack PM IDs and the Featurebase assignee IDs.
7. **Build prompt** wraps the ticket and the four context documents in `<ticket>`, `<team_scopes>`, `<pm_principles>`, `<current_work>` and `<capabilities>` tags.
8. **Get Initial Thread TS** looks up the Slack thread and **Indicate thinking** adds an `eyes` reaction so people know triage is running.
9. **Triage** calls the model in the **Triage model** variable (`claude-opus-4-8`, 16000 max tokens, temperature 1, extended thinking off, up to 3 retries) with the **Instructions** variable as the system prompt. The output is schema-constrained: classification (`feature_request`, `support_request`, `bug_report`, `spam`) and certainty, scope team and its reasoning, data sufficiency score, proposed response, proposed next step, an optional ClickUp-style ticket draft, and for feature requests a 0 to 100 build recommendation and a build team. The system prompt tells the model that most requests should not be built and that scores above 70 should be rare.

**Decision and outcome**

10. **Is spam?** is true only when the classification is `spam` and certainty is above 90.
11. Spam path: **Delete Post** removes the post from Featurebase, **Get Spam Thread TS** finds the Slack thread, and **Notify Spam** says the post was removed.
12. Otherwise two blocks run side by side. **Update Post** assigns the post to the `featurebase_assignee_id` of the chosen scope team. **Format Triage Response** builds a Slack Block Kit message, which **Notify Triage** posts in the thread after **Get Triage Thread TS**. The message shows classification, scope team, next step, proposed ticket, build recommendation, data sufficiency and the proposed response (HTML entities and tags are stripped, long text is split into chunks under Slack's 3000 character limit). It ends with a ping to the team's PM, or a warning if the team has no entry in `team-routing.json`.

**Comment sync**

13. **Comment Created** emits the comment, **Get Slack thread mapping** looks up the thread for its post, and **Mapping found?** stops quietly if there is none. **Post comment to Slack thread** posts the commenter's name and the comment text, converted from HTML to Slack formatting.

## What it needs

| Requirement | Detail |
|---|---|
| App installations | `anthropic` (Anthropic 0.8.0), `featurebase` (Featurebase 0.3.2), `github` (GitHub 0.6.0), `slack` (Slack 0.2.2), `utilities` (Utilities 0.2.1), `variables` (Variables 0.1.1). |
| Secrets | None in the file. Credentials live in the app installations. |
| Placeholders to replace | `CXXXXXXXXXX` (your Slack channel ID) in five blocks: Start a Slack thread, Indicate thinking, Notify Spam, Notify Triage and Post comment to Slack thread. `YOUR_FEEDBACK_DOMAIN` (your Featurebase portal domain) in Start a Slack thread. `YOUR_ORG` and `your-context-repo` in the five GitHub Fetch blocks. |
| Values to adapt | The **Instructions** variable is a generic skeptical-PM prompt. Tune it to your product, value drivers and tone. The fallback team names (`Platform`, `Integrations`, `Enterprise`) should match the keys in your `team-routing.json`. |
| GitHub context files | `team-scopes.md`, `pm-principles.md`, `current-work.md`, `capabilities.md` and `team-routing.json`. The JSON needs a top-level `teams` object keyed by team name, each with `slack_pm_user_id` and `featurebase_assignee_id`. The GitHub app needs read access to the repository. Use your own file names and update the paths in the Fetch blocks if they differ. |
| Featurebase | Webhook subscriptions for post created and comment created. Assignee IDs must be Featurebase admin IDs. |
| Slack permissions | The Slack app must be in the channel and able to post messages and add reactions. |
| Spacelift account link | Not needed; this flow has no Spacelift blocks. |
| Data tables | None. The thread mapping uses flow memory, collection `fb-slack-mapping`. |

## How to run and test it

1. Create a flow, paste the YAML onto the empty canvas, and point each app block at your installations.
2. Replace `CXXXXXXXXXX`, `YOUR_FEEDBACK_DOMAIN`, `YOUR_ORG` and `your-context-repo`, and update the file paths if your context files live elsewhere. The Errors panel should be empty.
3. Confirm **When post created** and **Comment Created** so Flows creates the Featurebase webhooks.
4. Create a test post in Featurebase. Expect a new Slack thread, an `eyes` reaction, then a "Triage Complete" message with a PM ping, and the post assigned to the team's admin.
5. Add a comment to the post and check that it shows up in the thread.
6. To test the spam path, use a throwaway post that is obviously spam. Delete Post is permanent.

## Known gaps

Reviewed on 2026-11-23 by reading the YAML. It has not been imported into a Flows project, so none of the behavior above has been run.

- **Author is probably empty.** Prepare ticket reads `getPost.post.author.name` and `.email`, but the Get Post output has `user`, not `author`. The model most likely sees empty author fields. Changing both to `.post.user` should fix it.
- **No human re-triage.** The system prompt describes `<triage>` and `<human>` inputs for a PM to correct a triage, but Build prompt never sends them. Every post is triaged once and there is no feedback loop.
- **Nothing is created downstream.** The proposed ticket is text in Slack only. Nothing is opened in ClickUp, and `proposed_next_step` is advisory.
- **Spam deletion is permanent and unreviewed.** A post is deleted as soon as the model says spam with certainty above 90. A false positive loses the post.
- **Errors are caught silently.** Handle Error has no blocks after it, so a failure in the triage chain produces no Slack message. The post stays unassigned and the thread shows `eyes` forever.
- **Mapping only covers new posts.** Comments on posts created before the flow existed, or where the mapping write failed, are skipped by Mapping found?. All comments are mirrored, including private ones.
- **Model ID not verified.** `claude-opus-4-8` is set in the Triage model variable. Check it against the models available to your Anthropic installation before you rely on it.
