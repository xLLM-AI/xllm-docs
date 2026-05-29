---
name: add-popular-model-usage-doc
description: Guide for adding or updating xLLM documentation pages under `src/content/docs/{en,zh}/popular_model_usage`. Use when Codex is asked to add a new Popular Model Usage model guide, translate a Popular Model Usage guide between Chinese and English, fix schema/frontmatter issues in these model docs, or keep paired zh/en model quick-start pages consistent in the docs repo.
---

# Add Popular Model Usage Doc

## Overview

Use this workflow when adding model quick-start documentation to the xLLM docs site. Keep the Chinese and English pages paired, schema-valid, and consistent with the existing Starlight content structure.

## Workflow

1. Inspect the repo before editing:
   - Run `git status --short`.
   - List existing files with `rg --files src/content/docs/{en,zh}/popular_model_usage`.
   - Read the closest existing examples, especially `quick_start_GLM5.md` and any model guide with similar hardware or runtime requirements.

2. Decide the target files:
   - Chinese path: `src/content/docs/zh/popular_model_usage/quick_start_<ModelName>.md`.
   - English path: `src/content/docs/en/popular_model_usage/quick_start_<ModelName>.md`.
   - Use the same filename stem in both locales. Keep the stem ASCII and stable; use model casing for readability, e.g. `quick_start_MiniMaxM2.md`.
   - If the user only asks for one language, update that language only, but mention the missing paired page if it matters.

3. Add valid Starlight frontmatter at the top of every page:

```yaml
---
title: "<Display Model Name>"
sidebar:
  order: <number>
---
```

Choose `sidebar.order` by inspecting existing Popular Model Usage pages and using the next sensible order unless the user specifies one. Keep the order the same across zh/en paired pages. Do not add a duplicate Markdown H1 after the frontmatter; Starlight renders the title.

4. Structure the guide consistently:
   - Source repository links.
   - Weight download links and any converted/quantized weight links.
   - Weight preparation, if needed.
   - Image/container setup.
   - Source checkout and build steps.
   - Model startup prerequisites.
   - Environment variables.
   - Startup command and key option notes.

5. Preserve technical correctness:
   - Keep executable commands, paths, flags, environment variable names, image tags, branch names, URLs, and model identifiers unchanged unless the user asks to change them.
   - Translate prose and explanatory shell comments when creating the other locale.
   - Prefer placeholders such as `/path/to/<model>/`, `<master-host>`, and `<local-host>` for new host-specific values. Do not invent private IPs.
   - If adding images, store shared assets under `src/content/docs/assets/` and reference them the same way existing docs do.

6. Keep zh/en pages aligned:
   - Match frontmatter `title` and `sidebar.order`.
   - Keep section order equivalent.
   - Keep commands semantically identical.
   - Translate user-facing text naturally rather than word-for-word when the existing English docs use clearer phrasing.

## Validation

Run these from the docs repo after edits:

```bash
npm run check
npm run build
```

`npm run check` may print existing hints from `src/components/PageTitle.astro`; treat the command as passing if it exits successfully with zero errors. If checking `npm run dev`, confirm the site reaches the ready state, then stop only the dev process you started. If port `4321` is occupied, Astro may choose another port; report the actual URL.
