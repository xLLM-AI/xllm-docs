---
name: add-cookbook
description: Guide for adding or updating xLLM cookbook documentation pages under `src/content/docs/{en,zh}/cookbook`. Use when Codex is asked to add a new cookbook model recipe, translate a cookbook guide between Chinese and English, fix schema/frontmatter issues in cookbook model docs, move model quick-start content into cookbook, or keep paired zh/en cookbook pages and manual sidebar entries consistent in the docs repo.
---

# Add Cookbook

## Overview

Use this workflow when adding or updating model cookbook documentation in the xLLM docs site. Keep the Chinese and English pages paired, schema-valid, technically exact, and consistent with the existing Starlight cookbook structure.

## Workflow

1. Inspect the repo before editing:
   - Run `git status --short`.
   - List existing cookbook files with `rg --files src/content/docs/{en,zh}/cookbook`.
   - Read the closest existing examples in the same category, model family, hardware target, or runtime style.
   - Read the cookbook sidebar section in `astro.config.mjs`.

2. Decide the target files:
   - Put paired files under matching relative paths in `src/content/docs/en/cookbook/` and `src/content/docs/zh/cookbook/`.
   - Use the existing category structure when it fits:
     - Autoregressive models: `cookbook/autoregressive_models/<family>/<model>.md`
     - Diffusion models: `cookbook/diffusion_models/<family>/<model>.md`
   - Keep directory names and filenames ASCII, lowercase, and stable. Prefer existing local conventions such as `glm_5.md`, `qwen3_vl.md`, or `minmax_m2_7.md`.
   - If the user only asks for one language, update that language only, but mention the missing paired page if it matters.

3. Add valid Starlight frontmatter at the top of every page:

```yaml
---
title: "<Display Model Name>"
description: "<Short cookbook recipe description>"
---
```

Use `title` only if no useful description is available. Do not add a duplicate Markdown H1 after the frontmatter; Starlight renders the title.

4. Update navigation when needed:
   - Cookbook pages are listed manually in `astro.config.mjs`; add or rename sidebar entries when adding, moving, or retitling cookbook pages.
   - Match zh/en labels through the `translations` field.
   - Keep the sidebar order consistent with the surrounding cookbook model family.

5. Structure model recipes consistently:
   - Source repository links.
   - Weight download links and any converted, quantized, or dequantized weight links.
   - Weight preparation, if needed.
   - Image/container setup.
   - Source checkout and build steps.
   - Model startup prerequisites.
   - Environment variables.
   - Startup command and key option notes.

6. Preserve technical correctness:
   - Keep executable commands, paths, flags, environment variable names, image tags, branch names, URLs, and model identifiers unchanged unless the user asks to change them.
   - Translate prose and explanatory shell comments when creating the other locale.
   - Prefer placeholders such as `/path/to/<model>/`, `<master-host>`, and `<local-host>` for new host-specific values. Do not invent private IPs.
   - If adding images, store shared assets under `src/content/docs/assets/` and reference them from both locales when possible.

7. Keep zh/en pages aligned:
   - Match relative paths and frontmatter `title`.
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
