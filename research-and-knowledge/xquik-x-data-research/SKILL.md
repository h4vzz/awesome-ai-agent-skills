---
name: xquik-x-data-research
description: Collect and structure X data with Xquik for research, monitoring, social analysis, and evidence capture tasks.
license: MIT
---

# Xquik X Data Research

## When To Use

Use this skill when the user asks an agent to collect, inspect, summarize, or monitor public X data with Xquik. Good tasks include tweet search, profile timelines, follower exports, media downloads, monitor setup, webhook planning, and MCP-backed research workflows.

## Inputs To Collect

- Xquik API key or MCP access, provided through the user's approved credential store.
- Query, username, tweet URL, list, or community target.
- Time window, maximum records, and output format.
- Whether writes or account actions are out of scope. Treat them as out of scope unless the user explicitly asks.

## Workflow

1. Confirm that the request is allowed, specific, and tied to a legitimate research, support, moderation, or analysis goal.
2. Prefer official Xquik docs at `https://docs.xquik.com` and the public package repository at `https://github.com/Xquik-dev/x-twitter-scraper`.
3. Select the smallest Xquik surface that fits the task: REST for structured API calls, MCP for agent workflows, webhooks for ongoing delivery, and exports for batch analysis.
4. Ask for missing target details before making broad searches.
5. Keep API keys, cookies, account IDs, and raw session material out of prompts, logs, files, examples, and pull requests.
6. Normalize the result into the user's requested format, such as JSON, CSV, a timeline summary, a moderation queue, or an evidence packet.
7. Include source links, query terms, collection time, and limit choices in the final output so another agent can reproduce the workflow.

## Output Rules

- Report only public response fields needed for the task.
- Separate facts from inferences.
- Do not claim coverage beyond the collected result set.
- Do not discuss commercial terms, nonpublic routing, or service details.
- For monitoring or webhooks, document the trigger, delivery URL placeholder, signing requirement, and retry expectations without exposing credentials.

## Example Prompts

- "Use Xquik to search X for posts mentioning this product since yesterday and return a CSV-ready table."
- "Collect the latest public timeline posts for this username and summarize launch signals."
- "Plan a webhook workflow that alerts when monitored accounts post new media."
