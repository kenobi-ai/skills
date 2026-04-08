---
name: kenobi-pages/run
description: Trigger Kenobi workflow runs to generate personalized content for leads. Use when generating pages, running workflows for prospects, creating content after sales calls, or batch-generating landing pages.
---

# Running Workflows

This skill covers triggering workflow runs, polling for completion, and verifying the generated content. A run executes a workflow for one lead and produces the content that the page will render.

## When to Use

- User wants to generate content for a specific lead or company
- User has finished a call and wants to create a personalized follow-up page
- User wants to batch-generate pages for multiple leads
- User mentions "run", "generate", "trigger", "create pages for"

## Before Running

A workflow must already exist. If the user hasn't created one, read the **workflows** sub-skill first.

Check what's available:

```bash
npx kenobi-pages workflows
```

If there are multiple workflows, ask which one to use. If there's only one, confirm it:

> I found a workflow called "Post-Call Landing Page" (ID: 42). It requires these parameters:
> - `slug` — URL slug for the page
> - `company_domain` — the lead's company domain
>
> Is this the right workflow?

---

## Single Run

### 1. Gather the inputs

Ask the user for the values needed. At minimum, every workflow needs a `slug`:

> What's the lead's company? I'll use that to generate the slug and look up their data.

Derive the slug from the company name (e.g., "Acme Corp" -> `acme-corp`).

### 2. Check for context

Ask if they have additional context for the AI:

> Do you have any context I should pass along? For example:
> - A call transcript or meeting notes
> - Specific talking points to emphasize
> - Tone or style preferences for this particular lead
>
> This is optional — the workflow will use its default instructions if you don't provide any.

Context is ephemeral — it's prepended to the AI generation prompts for this run only and isn't stored.

### 3. Confirm before triggering

Runs consume AI generation credits. Always confirm:

> I'm about to run the "Post-Call Landing Page" workflow for **acme-corp**. This will:
> - Look up Acme Corp's data from HubSpot
> - Generate personalized content using AI
> - Store it so the page at `/for/acme-corp` renders the content
>
> Go ahead?

### 4. Trigger

```bash
npx kenobi-pages run <workflowId> --params '{"slug":"acme-corp","company_domain":"acme.com"}'
```

With context:

```bash
npx kenobi-pages run <workflowId> \
  --params '{"slug":"acme-corp","company_domain":"acme.com"}' \
  --context "Call notes: They need faster onboarding. Emphasize self-serve setup."
```

The CLI polls automatically and prints the final result to stdout when done.

### 5. Verify

Once the run completes, verify the content looks right:

```bash
npx kenobi-pages page get <workflowId> acme-corp
```

Share the key content with the user so they can sanity-check it before sharing the link.

---

## Batch Runs

If the user wants to generate pages for multiple leads:

### 1. Get the list

Ask for all the leads:

> Which leads do you want to generate pages for? Give me company names and domains (or whatever parameters the workflow needs).

### 2. Confirm the batch

> I'll run the workflow for these 5 leads:
> 1. acme-corp (acme.com)
> 2. globex (globex.com)
> 3. initech (initech.com)
> 4. hooli (hooli.com)
> 5. piedpiper (piedpiper.com)
>
> Each run takes 30-60 seconds. Total estimated time: ~3-5 minutes. Proceed?

### 3. Run sequentially

Trigger each run one at a time. Report progress after each completion:

> Completed 2/5: acme-corp (success), globex (success). Running initech...

If a run fails, report it and continue with the remaining leads. Ask the user if they want to retry failures at the end.
