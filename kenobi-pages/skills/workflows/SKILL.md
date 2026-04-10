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
npx kenobi-pages workflow get <id> > .kenobi/workflows/<slug>.json
```

This saves the full config JSON. Edit the file and push it back with `workflow update`.

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
    { "id": "pages", "adapter": "kenobi-pages", "adapterConfig": { "slugField": "slug" } }
  ]
}
```

Key rules:
- `id` must match the workflow slug
- **The `slug` field must appear in three places:** `output.schema` (so its value is included in the output), `fields` (bound to `{ "strategy": "param", "paramName": "slug" }`), and `destinations[].adapterConfig.slugField`. If it's missing from `output.schema`, runs will succeed but content will be stored under an auto-generated slug instead of the one you passed — and pages will 404
- Each output field needs a binding: `param` (copy from runtime parameter), `passthrough` (copy from source field), `generate` (AI text), or `generate-image` (AI image)
- Fields bound to `generate` or `generate-image` reference a `generationGroups` entry by `groupId`
- A generation group has a `system` prompt, a `strategy` ("generate" or "generate-image"), and `contextSources` listing which source data the AI sees. Image groups also have `prompt` (task-specific directive), `images` (input images), and `imageConfig` — see the reference section below
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

Save the config to `.kenobi/workflows/<slug>.json` (create the directory if it doesn't exist) and pass it to the CLI:

```bash
mkdir -p .kenobi/workflows
npx kenobi-pages workflow create \
  --name "<name>" \
  --slug "<slug>" \
  --description "<description>" \
  --file .kenobi/workflows/<slug>.json
```

Or for simple configs, pass inline with `--config '<json>'`.

The `.kenobi/workflows/` directory is for workflow infrastructure files only — like `.github/workflows/`. It keeps config out of your app code. The file is your declarative workflow definition; the API is the source of truth once pushed.

---

## Full Example Config — Text Generation

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
      "company_name": { "type": "string", "description": "Lead company name (passed through from source)" },
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
    "company_name": { "strategy": "passthrough", "sourceId": "hubspot-contacts", "sourceField": "company" },
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
    { "id": "pages-dest", "adapter": "kenobi-pages", "adapterConfig": { "slugField": "slug" } }
  ]
}
```

## Modifying Workflows

Config updates require the full config object — the API does not accept partial configs. Use the get-modify-put pattern:

```bash
# 1. Get the full current config
npx kenobi-pages workflow get <id> > .kenobi/workflows/<slug>.json
# 2. Edit .kenobi/workflows/<slug>.json
# 3. Push the full config back
npx kenobi-pages workflow update <id> --file .kenobi/workflows/<slug>.json

# Name and description can still be updated independently:
npx kenobi-pages workflow update <id> --name "New Name"
npx kenobi-pages workflow update <id> --description "New description"

# Delete (confirm with user first — this is destructive)
npx kenobi-pages workflow delete <id>
```

Always use `workflow get` before updating to avoid overwriting changes made in the Kenobi UI.

## Reference

### Field Binding Strategies

Every output field needs a binding in the `fields` object. Each strategy has its own required properties:

| Strategy | Required properties | Description |
|---|---|---|
| `param` | `paramName` | Copy value from a runtime parameter |
| `passthrough` | `sourceId`, `sourceField` | Copy value from a source record field |
| `generate` | `groupId` | AI-generated text; references a generation group |
| `generate-image` | `groupId` | AI-generated image; references a generation group |

### Generation Group Keys — Text (`"generate"`)

| Key | Required | Description |
|---|---|---|
| `id` | yes | Unique identifier; referenced by field bindings via `groupId` |
| `strategy` | yes | `"generate"` |
| `system` | yes | System prompt — global context and instructions for the AI |
| `contextSources` | yes | Array of source refs — which source data the AI sees (see below) |

A text generation group can target multiple output fields — list them all with `{ "strategy": "generate", "groupId": "<id>" }` in `fields`. The AI produces all of them in a single call.

### Generation Group Keys — Image (`"generate-image"`)

| Key | Required | Description |
|---|---|---|
| `id` | yes | Unique identifier; referenced by field bindings via `groupId` |
| `strategy` | yes | `"generate-image"` |
| `system` | yes | System prompt — high-level context (brand, tone, constraints) |
| `prompt` | recommended | Task-specific instruction — what to change, preserve, and produce. Supports template variables: `{sourceId.fieldName}` |
| `contextSources` | yes | Array of source refs — text data the AI sees alongside images |
| `images` | recommended | Image inputs (see below) |
| `imageConfig` | optional | Tuning options for the image generation pipeline |
| `transparentBackground` | optional | When `true`, strips the output image background via chroma key |

An image generation group targets exactly **one** output field.

#### `images` object

Declares which source images are inputs to the generation call:

```json
"images": {
  "primary": { "sourceId": "lead-data", "field": "Hero Image", "label": "Original hero" },
  "references": [
    { "sourceId": "brand-assets", "field": "Logo", "label": "Brand logo" },
    { "sourceId": "brand-assets", "field": "Product Screenshot", "label": "Product UI" }
  ]
}
```

| Key | Type | Description |
|---|---|---|
| `primary` | `ImageSourceRef` (optional) | The base/template image to edit or reskin |
| `references` | `ImageSourceRef[]` (optional) | Additional reference images for style, branding, or constraint guidance |

Each `ImageSourceRef` has:
- `sourceId` — which workflow source contains the image
- `field` — the field name on that source holding the image URL
- `label` — (optional) human-readable description of this image's role; included in the AI prompt

#### `prompt` vs `system`

For text generation, `system` alone is sufficient. For image generation, use both:

- **`system`** — broad context: brand guidelines, audience, visual style, what the page is for
- **`prompt`** — surgical directive: exactly what to change in the primary image, what to preserve, what dimensions/aspect ratio to target

Template variables in `prompt` reference source data: `{sourceId.fieldName}` resolves to the value from the resolved source record. Use these to inject lead-specific details into the image prompt.

**Prompt quality matters enormously for image generation.** Vague prompts like "personalize this image" produce poor results. When reskinning a template image, the prompt must be surgically specific — describe exactly which elements to replace, what to preserve, and what constraints the output must satisfy. Here's a real-world example:

```json
{
  "system": "You are a graphic designer reskinning product UI screenshots for personalized landing pages.",
  "prompt": "The first image is a product UI screenshot for a template brand. Your job is to replace all instances of the template brand in this image in order to reskin it to {lead-data.Name}. You must keep the LAYOUT, SIZE, and STRUCTURE exactly the same — only surgically alter branding elements: logos, brand name text, brand colors, and any product imagery specific to the template. Replace them with {lead-data.Name} equivalents. The remaining images contain {lead-data.Name} brand assets from their website — use their logo, color palette, and visual identity as your reference for the replacement. You MUST NOT exercise artistic licence. Keep the vibe, composition, style, and proportions of the original image identical. The output must match the exact aspect ratio and shape of the first image — do not crop, pad, or reshape it."
}
```

Notice the level of specificity: it names exactly what to change (logos, brand text, colors, product imagery), what to preserve (layout, size, structure, composition, proportions, aspect ratio), and where to find the replacement assets (reference images). This is the bar for image generation prompts.

#### `imageConfig` options

| Key | Type | Default | Description |
|---|---|---|---|
| `preAnalyze` | boolean | `false` | Run a text-only LLM pre-pass on the primary image to produce an element inventory before generation |
| `dimensionConstraint` | boolean | `false` | Auto-detect primary image dimensions and inject aspect ratio constraint into the prompt |
| `candidateCount` | number | `3` | Number of candidate images to generate; the best is selected by an LLM judge. Set to `1` to skip judging |

### Context Source Scoping

By default, a `contextSources` entry includes **all fields** from the referenced source. You can scope to specific fields with the optional `fields` array:

```json
"contextSources": [
  { "sourceId": "lead-data", "fields": ["Name", "Company", "Hero Image"] },
  { "sourceId": "brand-assets" }
]
```

This is particularly important for image generation — sending irrelevant text fields as context can dilute the model's focus. Include only the fields the AI needs to see.

## Full Example Config — Image Generation

A workflow that takes a template hero image and reskins it with lead-specific branding:

```json
{
  "id": "branded-hero",
  "description": "Generate personalized hero images by reskinning a template with the lead's brand assets.",
  "params": [
    { "name": "slug", "description": "URL slug for the page" },
    { "name": "company_domain", "description": "Lead company domain for asset lookup" }
  ],
  "sources": [
    {
      "id": "lead-data",
      "provider": "notion",
      "dataSourceKey": "notion:leads-db",
      "title": "Lead Database",
      "schema": {
        "Name": { "type": "string" },
        "Domain": { "type": "string" },
        "Hero Original Image": { "type": "url" },
        "Logo": { "type": "url" }
      },
      "resolution": { "kind": "lookup", "field": "Domain", "param": "company_domain" }
    }
  ],
  "output": {
    "provider": "kenobi-pages",
    "dataSourceKey": "kenobi-pages:branded-hero",
    "title": "Branded Hero Page",
    "schema": {
      "slug": { "type": "string", "description": "URL slug" },
      "company_name": { "type": "string", "description": "Lead company name" },
      "hero_image_url": { "type": "url", "description": "Personalized hero image" }
    }
  },
  "fields": {
    "slug": { "strategy": "param", "paramName": "slug" },
    "company_name": { "strategy": "passthrough", "sourceId": "lead-data", "sourceField": "Name" },
    "hero_image_url": { "strategy": "generate-image", "groupId": "hero-gen" }
  },
  "generationGroups": [
    {
      "id": "hero-gen",
      "strategy": "generate-image",
      "system": "You are a graphic designer reskinning product UI screenshots for personalized B2B SaaS landing pages.",
      "prompt": "The first image is a product UI screenshot for a template brand. Your job is to replace all instances of the template brand in this image in order to reskin it to {lead-data.Name}. You must keep the LAYOUT, SIZE, and STRUCTURE exactly the same — only surgically alter branding elements: logos, brand name text, brand colors, and any product imagery specific to the template. Replace them with {lead-data.Name} equivalents. The remaining images contain {lead-data.Name} brand assets — use their logo, color palette, and visual identity as your reference for the replacement. You MUST NOT exercise artistic licence. Keep the vibe, composition, style, and proportions of the original image identical. The output must match the exact aspect ratio and shape of the first image — do not crop, pad, or reshape it.",
      "contextSources": [
        { "sourceId": "lead-data", "fields": ["Name", "Domain"] }
      ],
      "images": {
        "primary": { "sourceId": "lead-data", "field": "Hero Original Image", "label": "Template hero" },
        "references": [
          { "sourceId": "lead-data", "field": "Logo", "label": "Lead's company logo" }
        ]
      },
      "imageConfig": {
        "preAnalyze": true,
        "dimensionConstraint": true,
        "candidateCount": 3
      }
    }
  ],
  "destinations": [
    { "id": "pages-dest", "adapter": "kenobi-pages", "adapterConfig": { "slugField": "slug" } }
  ]
}
```

Key differences from text workflows:
- The generation group uses `"strategy": "generate-image"` and targets exactly one field (`hero_image_url`)
- `images` declares primary and reference inputs by source + field
- `prompt` is the task-specific directive (separate from `system`)
- `contextSources` is scoped to only the fields the AI needs
- `imageConfig` enables pre-analysis and candidate judging

---

## Tips

- Field `description` values matter — they're included in AI generation prompts, so be specific about what you want
- Test incrementally: create with a few fields first, trigger a test run, verify output, then add complexity
- The `description` on the workflow itself is prepended to every generation group's system prompt — use it for global instructions like tone, audience, or constraints
