---
name: kenobi-pages/workflows
description: Create and modify Kenobi workflows that wire data sources to AI generation steps. Use when building a content pipeline, connecting CRM data to page generation, or configuring how AI produces personalized content.
---

# Building Workflows

A workflow is a pipeline that takes data from external sources (CRM, databases, etc.), runs AI generation, and produces personalized content keyed by a slug. This skill covers creating and modifying workflows programmatically via the CLI.

## Prerequisite — Do Not Skip

You must have already completed the discovery question and setup from the parent skill (`kenobi-pages/SKILL.md`). If the user hasn't been asked whether they have a workflow yet, STOP — go back to the parent skill and start from Phase 1. Do not continue reading this file.

## First: Does a Workflow Already Exist?

```bash
npx kenobi-pages workflows
```

If a workflow already exists and the user wants to modify it:

```bash
npx kenobi-pages workflow get <id>
```

This returns the full config JSON. Edit it and push it back with `workflow update`.

If starting fresh, continue below.

---

## Building a New Workflow

### 1. Understand what the user wants

Ask:

> What's this workflow for? For example: "personalized landing pages sent after sales calls" or "custom product pages for each prospect."

Then ask about their data:

> Where does the lead data come from? For example:
> - A Notion database (CRM contacts, company info)
> - HubSpot (contacts, deals)
> - Or will you pass everything as parameters when triggering a run?

### 2. Discover connected sources

If they have external data sources:

```bash
npx kenobi-pages sources
```

This lists all data sources connected to their Kenobi org with their schemas. Share what you find:

After you pick a source you plan to use in the workflow, preview real rows so lookup fields and formats are obvious:

```bash
npx kenobi-pages sources sample <sourceKey>
```

Use the `sourceKey` from the list output (for example `notion:…` or `hubspot-crm:…`). Check values in columns you will match on — for example if the workflow uses lookup resolution on a domain field, confirm whether stored values look like `acme.com` or `https://acme.com`, and align the runtime param you pass accordingly.

> I can see these data sources connected to your Kenobi account:
> - "CRM Contacts" (Notion) — has fields: Name, Email, Domain, Company Size
> - "Deals" (HubSpot) — has fields: deal_name, amount, stage
>
> Which of these should feed into the workflow?

If they don't have sources connected, tell them:

> You'll need to connect your data sources in the Kenobi workflow builder UI first. Go to the workflow builder and use the integrations panel to connect Notion, HubSpot, or other services. Then come back and I can set up the workflow.
>
> Alternatively, you can build a workflow that only uses runtime parameters (values you pass in when triggering a run) — no external data sources needed.

### 3. Define the output schema

Ask:

> What content should be generated for each lead? For example:
> - A headline and value proposition
> - A list of relevant use cases
> - A hero image
> - A CTA button text

If a schema was already pushed via the pages skill (reverse mode), use that. Otherwise, define it together with the user.

### 4. Assemble the config

A workflow config is a JSON object. Here's the structure — fill in the values based on what you learned from the user:

```json
{
  "id": "<workflow-slug>",
  "description": "<what this workflow does — this gets prepended to AI generation prompts>",
  "params": [
    { "name": "slug", "description": "URL slug for the generated page" }
  ],
  "sources": [],
  "output": {
    "provider": "kenobi-pages",
    "dataSourceKey": "kenobi-pages:<schema-slug>",
    "title": "<human-readable name>",
    "schema": { "<field definitions>" }
  },
  "fields": {
    "slug": { "strategy": "param", "paramName": "slug" }
  },
  "generationGroups": [],
  "destinations": [
    { "id": "pages", "adapter": "kenobi-pages-store", "adapterConfig": {} }
  ]
}
```

Key rules:
- `id` must match the workflow slug
- Every workflow must have a `slug` field bound to `{ "strategy": "param", "paramName": "slug" }`
- Each output field needs a binding: `param` (copy from runtime parameter), `passthrough` (copy from source field), `generate` (AI text), or `generate-image` (AI image)
- Fields bound to `generate` or `generate-image` reference a `generationGroups` entry by `groupId`
- A generation group has a `system` prompt, a `strategy` ("generate" or "generate-image"), and `contextSources` listing which source data the AI sees
- Source `resolution` determines how data is fetched: `single` (take first record), `lookup` (filter by a param value), or `collection` (fetch all, optionally with AI selection)
- **Connections are auto-resolved** — you do NOT need to include `connectionRef` on sources. Cortex automatically resolves the connection from the source's `dataSourceKey` using the connection that was established when the data source was originally connected (e.g. via Notion OAuth). Only specify `connectionRef` if you need to override to a different connection than the one discovered with the source.

### 5. Confirm before creating

Show the user a summary of what you're about to create. Do not proceed without confirmation.

> Here's the workflow I'm about to create:
>
> **Name:** Post-Call Landing Page
> **Slug:** post-call-page
> **Sources:** HubSpot Contacts (lookup by company domain)
> **Output fields:** slug, headline, value_proposition, use_cases (array), cta_text
> **AI generation:** One text generation step using HubSpot contact data as context
> **Destination:** Kenobi Pages (stored as page content)
>
> Should I go ahead and create this?

### 6. Create

```bash
npx kenobi-pages workflow create \
  --name "<name>" \
  --slug "<slug>" \
  --description "<description>" \
  --file workflow-config.json
```

Or pass the config inline with `--config '<json>'`.

---

## Full Example Config

A workflow that reads HubSpot contacts and generates landing page content:

```json
{
  "id": "post-call-page",
  "description": "Generate personalized post-call landing pages. Emphasize how our product solves the lead's specific pain points.",
  "params": [
    { "name": "slug", "description": "URL slug for the page" },
    { "name": "company_domain", "description": "Lead company domain for CRM lookup" }
  ],
  "sources": [
    {
      "id": "hubspot-contacts",
      "provider": "hubspot-crm",
      "dataSourceKey": "hubspot-crm:contacts",
      "title": "HubSpot Contacts",
      "schema": {
        "firstname": { "type": "string" },
        "lastname": { "type": "string" },
        "company": { "type": "string" },
        "website": { "type": "string" }
      },
      "resolution": { "kind": "lookup", "field": "website", "param": "company_domain" }
    }
  ],
  "output": {
    "provider": "kenobi-pages",
    "dataSourceKey": "kenobi-pages:post-call-page",
    "title": "Post-Call Page",
    "schema": {
      "slug": { "type": "string", "description": "URL slug" },
      "headline": { "type": "string", "description": "Attention-grabbing personalized headline" },
      "value_proposition": { "type": "string", "description": "Why our product matters to this lead" },
      "use_cases": {
        "type": "array",
        "items": { "type": "object", "fields": { "title": { "type": "string" }, "description": { "type": "string" } } },
        "min": 3, "max": 5,
        "description": "Relevant use cases for the lead"
      },
      "cta_text": { "type": "string", "description": "Call-to-action button text" }
    }
  },
  "fields": {
    "slug": { "strategy": "param", "paramName": "slug" },
    "headline": { "strategy": "generate", "groupId": "text-gen" },
    "value_proposition": { "strategy": "generate", "groupId": "text-gen" },
    "use_cases": { "strategy": "generate", "groupId": "text-gen" },
    "cta_text": { "strategy": "generate", "groupId": "text-gen" }
  },
  "generationGroups": [
    {
      "id": "text-gen",
      "strategy": "generate",
      "system": "Write personalized landing page content for a B2B SaaS product. Use the lead's company data to craft compelling, specific copy. Be concise and action-oriented.",
      "contextSources": [{ "sourceId": "hubspot-contacts" }]
    }
  ],
  "destinations": [
    { "id": "pages-dest", "adapter": "kenobi-pages-store", "adapterConfig": {} }
  ]
}
```

## Modifying Workflows

```bash
# Get current config
npx kenobi-pages workflow get <id>

# Update (partial — only send what changed)
npx kenobi-pages workflow update <id> --config '<json>'
npx kenobi-pages workflow update <id> --name "New Name"
npx kenobi-pages workflow update <id> --description "New description"

# Delete (confirm with user first — this is destructive)
npx kenobi-pages workflow delete <id>
```

Always use `workflow get` before updating to avoid overwriting changes made in the Kenobi UI.

## Tips

- Field `description` values matter — they're included in AI generation prompts, so be specific about what you want
- Test incrementally: create with a few fields first, trigger a test run, verify output, then add complexity
- The `description` on the workflow itself is prepended to every generation group's system prompt — use it for global instructions like tone, audience, or constraints
