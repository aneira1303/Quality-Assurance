# Task 2: n8n API Integration Workflow ("Morning Brief")

**Author:** Aneira Patricio
**Workflow file:** `Task2_Workflow_AneiraPatricio.json`

## Overview
An hourly workflow (for testing) that finds the most-starred new LLM repositories on GitHub, enriches each one with its README, sorts them into tiers, and posts a single digest to a Discord channel. If anything fails, a separate alert is posted instead of the run failing silently.

**Flow:** Schedule → GitHub Search → Reshape Top 5 → Fetch README → Attach Snippet → IF stars > 1000 → Tag → Merge → Build Digest → Discord

## APIs used and why
| API | Purpose | Why |
|---|---|---|
| **GitHub REST: `/search/repositories`** | Find repos with `topic:llm` created in the last 30 days, sorted by stars | Free and well documented. It has no official "trending" endpoint, so recent and most-starred is a close proxy. |
| **GitHub REST: `/repos/{owner}/{repo}/readme`** | Second call that enriches each repo with its README | Another endpoint of the same API, so one credential covers both calls. |
| **Discord webhook** | Delivers the digest | Free, instant, and easy to verify visually. |

## What the transformation does
The **Reshape Top 5** Code node takes the 20 search results and (1) drops archived repos and forks, (2) sorts by stars, descending, (3) keeps the top 5, and (4) reduces each to `full_name, description, stars, language, url`. **Attach README Snippet** adds the first 200 characters of each README. **Build Digest** formats everything into one message, grouped by tier.

## Conditional branch
An **IF** node checks `stars > 1000`. Repos above the threshold are tagged **🔥 Hot** and the rest **🌱 Rising**. The branches are rejoined with a Merge node, so one digest is sent with both sections.

## Error handling
- **GitHub Search** and **Reshape Top 5** use *On Error → Continue (using error output)*. Any failure (rate limit, timeout, bad response, zero results) goes to **Format Error Message → Send Failure Alert to Discord**, which posts the reason, execution ID, and time.
- **Fetch README** uses *Continue (regular output)*, so one repo without a README shows "README unavailable" instead of breaking the digest.
- HTTP and Discord nodes **retry 3 times** (2 s apart) before the error path is taken, and HTTP calls have a 15 s timeout.

## Credentials
No secrets are in the JSON. Two credentials live in n8n's Credentials store and are referenced by name:
- **GitHub API:** a fine-grained personal access token (public repositories, read-only, no permissions) to raise the rate limit.
- **Discord Webhook API:** the channel's webhook URL.

## Testing
A manual run completed with all main-path nodes green: 1 search response → 5 repos → 5 enriched → 1 digest, delivered to Discord. To test the error path, change the GitHub Search URL to an invalid path and re-run; a ⚠️ alert appears in Discord.

## To run
Import the JSON, attach both credentials, and click **Execute Workflow**. For production, change the trigger interval to a daily morning schedule.
