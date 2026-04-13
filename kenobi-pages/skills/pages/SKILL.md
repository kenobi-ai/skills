---
name: kenobi-pages/pages
description: Build a Next.js dynamic route that renders personalized, AI-generated content per lead. Use when creating landing pages, building pages from a Kenobi workflow schema, or personalizing an existing page with dynamic content.
---

# Building Pages

This skill covers creating the Next.js page that renders personalized content fetched from Kenobi at runtime.

## Prerequisite — Do Not Skip

You must have already completed the discovery question and setup from the parent skill (`kenobi-pages/SKILL.md`). If the user hasn't been asked whether they have a workflow yet, STOP — go back to the parent skill and start from Phase 1. Do not continue reading this file.

## Determine the Mode

You should already know which mode to use from the discovery question in the parent skill. If not, ask:

> Do you already have a Kenobi workflow with a defined output schema, or do you have an existing page you'd like to personalize?

- **"I have a workflow"** → Forward mode (schema already exists, build the page around it)
- **"I have a page I want to personalize"** → Reverse mode (analyze the page, define a schema, push it to Kenobi)
- **"Neither" / unclear** → They probably need to set up a workflow first. Read the **workflows** sub-skill. Come back here once a workflow exists.

---

## Forward Mode — Build a Page From an Existing Schema

The workflow already exists. You need to discover the schema and build the dynamic route.

### 1. Get the workflow ID

If you don't know it, list workflows:

```bash
npx kenobi-pages workflows
```

Ask the user which workflow this page is for.

### 2. Fetch the schema and generate types

```bash
npx kenobi-pages schema get <workflowId>
```

Review the schema. Tell the user what fields will be available on the page:

> Your workflow produces these fields for each lead:
>
> - `headline` (string) — personalized headline
> - `value_proposition` (string) — why your product matters to them
> - `use_cases` (array) — 3-5 relevant use cases
> - `hero_image_url` (url) — AI-generated hero image
> - `cta_text` (string) — call-to-action button text
>
> I'll build a page that renders all of these. Sound good?

Generate the TypeScript interface and place it inside the route directory:

```bash
npx kenobi-pages schema codegen <workflowId> > app/for/[slug]/types.ts
```

If the user prefers a different route path, adjust accordingly.

### 3. Build the dynamic route

Create the route directory and page files. **All page-specific code goes inside the route directory** — this keeps things self-contained and means deleting the route cleans up everything.

The route directory should contain:

- `page.tsx` — the server component (calls `kenobi.getPage()`, returns `notFound()` if null)
- `types.ts` — the TypeScript interface (generated in step 2)
- `view.tsx` — the presentational component (receives typed content as props)
- `content.ts` — content parsing and dev placeholder data

The `page.tsx` server component should:

- Import and use the `kenobi` client (from wherever it was placed during setup — see parent skill Phase 3)
- Call `kenobi.getPage(WORKFLOW_ID, slug)` with caching options — `WORKFLOW_ID` can be the numeric ID (e.g. `42`) or the workflow slug (e.g. `"post-call-page"`)
- Cast `page.content` to the type from `types.ts`
- Pass the typed content to the `view.tsx` component

The user's brand, layout preferences, and design system should drive the page design. Don't use a generic template — build it to match their project.

### 4. Placeholder content for development

If no workflow runs have completed yet, add realistic mock data to `content.ts` matching the type interface. Use it as a fallback in development only:

```ts
if (!page && process.env.NODE_ENV === "development") return PLACEHOLDER_CONTENT;
```

**Page-first flow (Path A):** If the workflow doesn't exist yet (the user is designing the page before building the workflow), `getPage` will return `null` since the workflow can't be found. The placeholder fallback above handles this — the page renders with mock data during development, and real content flows once a workflow exists and has been run.

Tell the user to remove the placeholder once real content is flowing.

### 5. Verify

Suggest the user starts their dev server and visits `/for/<slug>` to check the page renders correctly with placeholder data. If the user wants to see real content, offer to trigger a test run (see the **run** sub-skill).

### 6. Caching

Explain the caching options and ask what they want:

> How fresh should the content be? Options:
>
> - Revalidate every N seconds (ISR) — good default, e.g. `{ revalidate: 60 }`
> - Cache forever — `{ revalidate: false }`
> - Always fresh — `{ revalidate: 0 }`
> - On-demand — `{ tags: ["kenobi"] }` with `revalidateTag("kenobi")`

---

## Reverse Mode — Personalize an Existing Page

The user has an existing page with hardcoded content and wants to make parts of it dynamic.

### 1. Analyze the page

Read the page's JSX. Identify every piece of content that should change between leads — headings, descriptions, images, CTAs, lists, testimonials, etc.

### 2. Propose the schema to the user

Present your analysis and ask for confirmation:

> I've analyzed your page. Here are the parts I think should be personalized per lead:
>
> 1. `headline` (string) — the main H1
> 2. `subtitle` (string) — the description under the headline
> 3. `pain_points` (array of { title, description }) — the three pain points section
> 4. `cta_text` (string) — the button text
>
> Does this look right? Should I add or remove anything?

Do not proceed until the user confirms. They may want to add fields you missed or remove ones they want to keep static.

### 3. Push the schema

After confirmation, construct the schema JSON and push it inline — do not save a separate schema JSON file, as it's a one-shot operation:

```bash
npx kenobi-pages schema push "Page Name" '{"fields":{"headline":{"type":"string"},...}}'
```

After the schema is pushed, offer the user a choice:

> Schema pushed to Kenobi. I can create a workflow that targets this schema now, or you can set it up yourself in the Kenobi UI. Want me to build the workflow?

If yes, read the **workflows** sub-skill and continue — the schema is already defined, so skip step 3 (Define the output schema) and go straight to assembling the config.

If no, explain what they'll need to do in the UI before the agent can resume:

> To finish setup:
>
> 1. Connect your data sources at [kenobi.ai/setup](https://kenobi.ai/setup) if you haven't already
> 2. Go to the workflow builder and select this schema as your output target
> 3. Configure AI generation and run the workflow for your leads
>
> Once a workflow is configured, come back and I can refactor this page to fetch content dynamically.

### 4. Refactor the page

Replace hardcoded content with the same pattern as forward mode — `kenobi.getPage()` in a server component, with placeholders for development. Co-locate all new files (`types.ts`, `content.ts`, `view.tsx`) inside the route directory alongside the existing page.

---

## Schema Field Types Reference

When constructing a schema (for `schema push` or inside a workflow config's `output.schema`), use these field type definitions:

| Type                | Syntax                                                                                      | Notes                                     |
| ------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------- |
| String              | `{ "type": "string" }`                                                                      | Optional: `min`, `max` (character counts) |
| URL                 | `{ "type": "url" }`                                                                         | Validated as a well-formed URL            |
| Number              | `{ "type": "number" }`                                                                      | Optional: `min`, `max`, `integer: true`   |
| Boolean             | `{ "type": "boolean" }`                                                                     |                                           |
| Enum                | `{ "type": "enum", "values": ["a", "b"] }`                                                  |                                           |
| Object              | `{ "type": "object", "fields": { "title": { "type": "string" } } }`                         | Use `fields`, not `properties`            |
| Array of objects    | `{ "type": "array", "items": { "type": "object", "fields": { ... } }, "min": 1, "max": 5 }` | `min`/`max` are optional                  |
| Array of primitives | `{ "type": "array", "items": { "type": "string" } }`                                        |                                           |

Every field also accepts optional `description` (string, included in AI prompts) and `optional` (boolean).

**Important:** Nested objects use `fields` for their children, not `properties`. The API will accept `properties` as an alias, but always write `fields` in your configs.

---

## Error Handling

`getPage` returns `null` when content doesn't exist — whether the slug has no content, the workflow doesn't exist, or the workflow ID/slug doesn't resolve. Handle it with `notFound()`.

For API key issues, the SDK throws `KenobiUnauthorizedError`. Catch it if you want custom error UI, otherwise let it bubble to the error boundary.

```ts
import { KenobiUnauthorizedError, KenobiPagesError } from "kenobi-pages";
```
