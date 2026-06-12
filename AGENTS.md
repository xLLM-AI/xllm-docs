# AGENTS.md

This repository is the Astro + Starlight documentation site for the xLLM project.

## Commands

All commands are run from the root of the project, from a terminal:

| Command                   | Action                                           |
| :------------------------ | :----------------------------------------------- |
| `npm install`             | Installs dependencies                            |
| `npm run dev`             | Starts local dev server at `localhost:4321`      |
| `npm run build`           | Build your production site to `./dist/`          |
| `npm run preview`         | Preview your build locally, before deploying     |
| `npm run check`           | Run Astro, Starlight, and TypeScript checks      |
| `npm run astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `npm run astro -- --help` | Get help using the Astro CLI                     |

## Project Structure

| Path                                           | Purpose                                                |
| :--------------------------------------------- | :----------------------------------------------------- |
| `src/pages/`                                   | Astro routes                                           |
| `src/pages/index.astro`                        | Redirects the root route `/` to `/en/`                 |
| `src/content/docs/en/`                         | English documentation content                          |
| `src/content/docs/zh/`                         | Simplified Chinese documentation content               |
| `src/content/docs/assets/`                     | Shared documentation images and diagrams               |
| `src/content/docs/*/getting_started/`          | Getting-started guides                                 |
| `src/content/docs/*/cookbook/`                 | Model-specific recipes and quick-start guides          |
| `src/content/docs/*/features/`                 | User-facing feature documentation                      |
| `src/content/docs/*/dev_guide/`                | Developer guides                                       |
| `src/content/docs/*/design/`                   | Design documents                                       |
| `src/content.config.ts`                        | Starlight content collection schema                    |
| `src/components/`                              | Starlight component overrides                          |
| `src/plugins/rehypeSimpleIcons.mjs`            | Markdown transform for supported Simple Icons tokens   |
| `src/styles/theme.css`                         | Project theme customizations                           |
| `src/assets/`                                  | Site-level assets such as the logo                     |
| `public/`                                      | Static assets copied directly into the build output    |
| `dist/`                                        | Generated production build output; do not edit by hand |
| `.astro/`                                      | Generated Astro metadata; do not edit by hand          |
| `astro.config.mjs`                             | Starlight locale, sidebar, theme, and component config |
| `package.json`                                 | npm scripts and dependencies                           |

## Development Notes

- Put user-facing documentation under `src/content/docs/en` and `src/content/docs/zh`.
- Keep matching relative paths in the English and Chinese trees when a page exists in both languages.
- Every documentation page should start with valid Starlight frontmatter, usually `title` and optional `sidebar.order`.
- Do not add a duplicate Markdown H1 when Starlight already renders the frontmatter `title`.
- Store shared images in `src/content/docs/assets/` and reference them from both locales when possible.
- Update the `sidebar` section in `astro.config.mjs` when adding pages or sections that are not covered by existing `autogenerate` entries.
- Existing autogen sidebar directories are `features`, `dev_guide`, and `design`.
- For cookbook model pages, keep paired files under matching `src/content/docs/{en,zh}/cookbook/` relative paths and update the manual cookbook sidebar entries in `astro.config.mjs`.
- Preserve commands, flags, paths, environment variables, image tags, branch names, URLs, and model identifiers exactly unless the requested change is explicitly about those values.
- Prefer placeholders like `/path/to/<model>/`, `<master-host>`, and `<local-host>` for host-specific examples; do not invent private IPs.
- Use `:simple-github:` only in normal Markdown text, not inside code blocks, when the GitHub icon token is needed.
- Run at least `npm run check` and `npm run build` before submitting documentation, component, config, or theme changes.
- For layout or visual changes, also verify in the local dev server.
