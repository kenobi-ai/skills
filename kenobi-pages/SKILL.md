---
name: kenobi-pages
description: Build personalized landing pages powered by Kenobi workflow content. Supports schema-first (forward) and code-first (reverse) flows.
---

# Kenobi Pages — Personalized Landing Pages

Build a single page template that renders unique, AI-generated content for every lead. One workflow. One template. N personalized pages.

Each page gets a unique URL like `https://yoursite.com/for/acme-corp` and fetches its content from the Kenobi Pages API at runtime (or build time).

---

## Prerequisites

- A Kenobi account with an organization
- A **Pages API key** from the Kenobi workflow builder (starts with `pk_live_` or `pk_test_`)
- A Next.js project (App Router)
- Either a configured Kenobi workflow (forward mode) or an existing page to personalize (reverse mode)

---

## Installation

```bash
pnpm add kenobi-pages
```

Then ask the user to run the setup command in their terminal:

```bash
npx kenobi-pages init
```

This will prompt them for their API key (from the Kenobi workflow builder at `/testing/cortex`), save it globally at `~/.kenobi/config.json`, and add `KENOBI_PAGES_KEY` to the project's `.env.local` automatically.

---

## Client Setup

Create `lib/kenobi.ts`:

```ts
import { createKenobiPagesClient } from "kenobi-pages"

export const kenobi = createKenobiPagesClient({
  apiKey: process.env.KENOBI_PAGES_KEY!,
})
```

If your Kenobi instance runs at a custom domain, pass `baseUrl`:

```ts
export const kenobi = createKenobiPagesClient({
  apiKey: process.env.KENOBI_PAGES_KEY!,
  baseUrl: "https://your-kenobi-instance.com",
})
```

---

## Forward Mode (Schema-First)

Use this when a Kenobi workflow already exists and you want to build a page around its output schema.

### Step 1 — Discover the Schema

The workflow has a defined output schema. Fetch it using the CLI:

```bash
npx kenobi-pages schema get 42
```

This outputs the schema JSON:

```json
{
  "schema": {
    "fields": {
      "headline": { "type": "string", "description": "Personalized headline" },
      "value_proposition": { "type": "string", "description": "Why this matters to the lead" },
      "use_cases": { "type": "array", "items": { "type": "object", "fields": {
        "title": { "type": "string" },
        "description": { "type": "string" }
      }}, "min": 3, "max": 5 },
      "hero_image_url": { "type": "url", "description": "AI-generated hero image" },
      "cta_text": { "type": "string" }
    }
  },
  "provider": "kenobi-pages",
  "title": "Post-Call Landing Page"
}
```

### Step 2 — Generate TypeScript Types

Auto-generate a typed interface from the schema:

```bash
npx kenobi-pages types 42
```

This outputs a TypeScript interface you can save to `lib/kenobi-types.ts`:

```ts
export interface PostCallLandingPage {
  /** Personalized headline */
  headline: string
  /** Why this matters to the lead */
  value_proposition: string
  use_cases: Array<{
    title: string
    description: string
  }>
  /** AI-generated hero image */
  hero_image_url: string
  cta_text: string
}
```

Rename the interface if needed (the name is derived from the workflow's title).

### Step 3 — Create Placeholder Content

Create `lib/kenobi-placeholder.ts` with realistic mock data for development:

```ts
import type { LandingPageContent } from "./kenobi-types"

export const PLACEHOLDER_CONTENT: LandingPageContent = {
  headline: "Transform Your Workflow with AI-Powered Automation",
  value_proposition:
    "Reduce manual effort by 80% and ship faster with intelligent content generation tailored to each prospect.",
  use_cases: [
    { title: "Sales Outreach", description: "Personalized landing pages after every call." },
    { title: "ABM Campaigns", description: "Unique content for each target account." },
    { title: "Event Follow-Up", description: "Custom recap pages for every attendee." },
  ],
  hero_image_url: "https://placehold.co/1200x630/1a1a2e/ffffff?text=Hero+Image",
  cta_text: "Get Started",
}
```

### Step 4 — Build the Page

Create `app/for/[slug]/page.tsx`:

```tsx
import { notFound } from "next/navigation"

import { kenobi } from "@/lib/kenobi"
import { PLACEHOLDER_CONTENT } from "@/lib/kenobi-placeholder"
import type { LandingPageContent } from "@/lib/kenobi-types"

const WORKFLOW_ID = 42 // Replace with your actual workflow ID

const getContent = async (slug: string): Promise<LandingPageContent> => {
  const page = await kenobi.getPage(WORKFLOW_ID, slug, { revalidate: 60 })

  if (!page) {
    // No content yet — use placeholders during development.
    // Once ANY real content is returned, remove the placeholder fallback.
    if (process.env.NODE_ENV === "development") {
      return PLACEHOLDER_CONTENT
    }
    notFound()
  }

  return page.content as LandingPageContent
}

const ForSlugPage = async ({ params }: { params: Promise<{ slug: string }> }) => {
  const { slug } = await params
  const content = await getContent(slug)

  return (
    <main>
      <section>
        <h1>{content.headline}</h1>
        <p>{content.value_proposition}</p>
        <img src={content.hero_image_url} alt="" />
      </section>

      <section>
        <h2>Use Cases</h2>
        {content.use_cases.map((uc) => (
          <div key={uc.title}>
            <h3>{uc.title}</h3>
            <p>{uc.description}</p>
          </div>
        ))}
      </section>

      <section>
        <a href="/contact">{content.cta_text}</a>
      </section>
    </main>
  )
}

export default ForSlugPage
```

### Step 5 — Caching and Revalidation

The `{ revalidate: 60 }` option tells Next.js to cache the response for 60 seconds (ISR). Adjust as needed:

- `{ revalidate: false }` — cache indefinitely (static)
- `{ revalidate: 0 }` — no caching (always fresh)
- `{ tags: ["kenobi-page"] }` — enable on-demand revalidation via `revalidateTag("kenobi-page")`

### Step 6 — Remove Placeholders

Once the workflow has run and real content is available, remove the placeholder fallback. The `notFound()` path handles missing content in production. Delete `lib/kenobi-placeholder.ts` when it's no longer needed.

---

## Reverse Mode (Code-First)

Use this when you already have a page and want to identify which parts should be personalized, then push that schema to Kenobi.

### Step 1 — Analyze the Page

Look at the existing page's JSX and identify every piece of content that should change between leads. These are your variable fields.

Ask the user to confirm which elements are variable. Propose a schema like this:

```
I've analyzed your landing page. Here are the fields I think should be personalized:

1. headline (string) — the main H1 text
2. value_proposition (string) — the subtitle/description
3. company_logo_url (url) — the lead's company logo
4. pain_points (array of { title: string, description: string }) — the 3 pain points section
5. cta_text (string) — the call-to-action button text

Does this look right? Should I add or remove any fields?
```

### Step 2 — Push the Schema to Kenobi

After confirming with the user, construct the schema JSON and push it in a single command:

```bash
npx kenobi-pages schema push "Post-Call Landing Page" '{"fields":{"headline":{"type":"string","description":"Personalized headline for the lead"},"value_proposition":{"type":"string","description":"Why our product matters to this specific lead"},"company_logo_url":{"type":"url","description":"Lead company logo"},"pain_points":{"type":"array","items":{"type":"object","fields":{"title":{"type":"string"},"description":{"type":"string"}}},"min":2,"max":5,"description":"Pain points relevant to the lead"},"cta_text":{"type":"string","description":"Call-to-action button text"}}}'
```

The CLI outputs the result as JSON to stdout and a confirmation to stderr:

```json
{ "sourceKey": "kenobi-pages:post-call-landing-page", "name": "Post-Call Landing Page" }
```

Tell the user:

> Schema pushed to Kenobi. Go to the Kenobi workflow builder, select the
> "Post-Call Landing Page" schema as your output target, configure your data
> sources and AI generation steps, then run the workflow for your leads.

### Step 3 — Refactor the Page

Replace hardcoded content with dynamic fetching. Follow the same pattern as Forward Mode Steps 2–5:

1. Create the typed interface matching the schema
2. Create placeholder content for development
3. Use `kenobi.getPage()` in the server component
4. Render the variable fields from the API response

---

## Type Safety Pattern

For stronger typing, create a typed wrapper around `getPage`:

```ts
import type { KenobiFetchOptions } from "kenobi-pages"

import { kenobi } from "@/lib/kenobi"
import type { LandingPageContent } from "@/lib/kenobi-types"

const WORKFLOW_ID = 42

export const getLandingPage = async (
  slug: string,
  opts?: KenobiFetchOptions
): Promise<LandingPageContent | null> => {
  const page = await kenobi.getPage(WORKFLOW_ID, slug, opts)
  if (!page) return null
  return page.content as LandingPageContent
}
```

---

## Error Handling

The SDK throws typed errors:

```ts
import { KenobiUnauthorizedError, KenobiPagesError } from "kenobi-pages"

try {
  const page = await kenobi.getPage(workflowId, slug)
} catch (err) {
  if (err instanceof KenobiUnauthorizedError) {
    // API key is invalid or missing
  }
  if (err instanceof KenobiPagesError) {
    // Generic API error — check err.status and err.code
  }
}
```

`getPage` returns `null` for 404 (page not found) — it does not throw. All other HTTP errors throw.

---

## Schema Field Types Reference

When defining a schema (reverse mode), use these field types:

| Type      | Description                          | Extra Options                    |
| --------- | ------------------------------------ | -------------------------------- |
| `string`  | Free-text string                     | `min`, `max` (length bounds)     |
| `url`     | Well-formed URL                      | —                                |
| `number`  | Numeric value                        | `min`, `max`, `integer`          |
| `boolean` | True or false                        | —                                |
| `enum`    | One of a set of allowed values       | `values: ["a", "b", "c"]`       |
| `object`  | Nested object with named fields      | `fields: { ... }`               |
| `array`   | Ordered list of items                | `items: { ... }`, `min`, `max`  |

All types support `description` (helps AI generation) and `optional` (marks the field as not required).

---

## CLI Reference

The `kenobi-pages` CLI is available after installing `kenobi-pages`. It reads the API key from the `KENOBI_PAGES_KEY` environment variable first, then falls back to `~/.kenobi/config.json` (set via `npx kenobi-pages init`).

```bash
# Fetch a workflow's output schema
npx kenobi-pages schema get <workflowId>

# Generate TypeScript types and save to file
npx kenobi-pages types <workflowId> > lib/kenobi-types.ts

# Push a schema to Kenobi (inline JSON — preferred for agents)
npx kenobi-pages schema push "Schema Name" '{"fields":{"headline":{"type":"string"}}}'

# Push a schema from file (alternative)
npx kenobi-pages schema push "Schema Name" --file schema.json

# Fetch page content for a specific lead
npx kenobi-pages page get <workflowId> <slug>

# Override base URL
KENOBI_BASE_URL=https://custom.kenobi.ai npx kenobi-pages schema get 42
```

All commands output JSON to stdout and status messages to stderr.
Exit codes: 0 = success, 1 = API error, 2 = bad arguments, 3 = not found, 4 = unauthorized.

---

## SDK Quick Reference

```ts
import { createKenobiPagesClient } from "kenobi-pages"

const kenobi = createKenobiPagesClient({ apiKey: process.env.KENOBI_PAGES_KEY! })

// Fetch page content (returns null if not found)
const page = await kenobi.getPage(workflowId, "acme-corp", { revalidate: 60 })

// Fetch workflow output schema
const schema = await kenobi.getSchema(workflowId)

// Push a schema (reverse flow)
const result = await kenobi.postSchema("My Landing Page", { fields: { ... } })
```
