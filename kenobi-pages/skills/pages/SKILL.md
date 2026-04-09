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

### 2. Fetch the schema

```bash
npx kenobi-pages schema get <workflowId>
```

Review the schema. Define a TypeScript interface in the project that matches the field names and types. Tell the user what fields will be available on the page. For example:

> Your workflow produces these fields for each lead:
>
> - `headline` (string) — personalized headline
> - `value_proposition` (string) — why your product matters to them
> - `use_cases` (array) — 3-5 relevant use cases
> - `hero_image_url` (url) — AI-generated hero image
> - `cta_text` (string) — call-to-action button text
>
> I'll build a page that renders all of these. Sound good?

### 3. Build the dynamic route

Create `app/for/[slug]/page.tsx` (or whatever route the user prefers). The page should:

- Import and use the `kenobi` client from `lib/kenobi.ts`
- Call `kenobi.getPage(WORKFLOW_ID, slug)` in a server component
- Return `notFound()` if the page returns `null`
- Use `{ revalidate: 60 }` (or whatever caching the user wants) as the third argument
- Cast `page.content` to the generated type

The user's brand, layout preferences, and design system should drive the page design. Don't use a generic template — build it to match their project.

### 4. Placeholder content for development

If no workflow runs have completed yet, create a placeholder file with realistic mock data matching the type interface. Use it as a fallback in development only:

```ts
if (!page && process.env.NODE_ENV === "development") return PLACEHOLDER_CONTENT;
```

Tell the user to remove the placeholder once real content is flowing.

### 5. Caching

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

After confirmation, construct the schema JSON and push it:

```bash
npx kenobi-pages schema push "Page Name" '{"fields":{...}}'
```

Tell the user what to do next:

> Schema pushed to Kenobi. Next steps:
>
> 1. Go to the Kenobi workflow builder
> 2. Select this schema as your output target
> 3. Connect your data sources and configure AI generation
> 4. Run the workflow for your leads
>
> Once a workflow is configured, I can refactor this page to fetch content dynamically. Want me to do that now with placeholder data, or wait until the workflow is ready?

### 4. Refactor the page

Replace hardcoded content with the same pattern as forward mode — `kenobi.getPage()` in a server component, with placeholders for development.

---

## Error Handling

`getPage` returns `null` when content doesn't exist for a slug — it does not throw. Handle it with `notFound()`.

For API key issues, the SDK throws `KenobiUnauthorizedError`. Catch it if you want custom error UI, otherwise let it bubble to the error boundary.

```ts
import { KenobiUnauthorizedError, KenobiPagesError } from "kenobi-pages";
```
