# AGENTS.md

## Project

Hugo static site (personal blog). Theme: PaperMod (git submodule — never edit `themes/PaperMod/`).

## Commands

```bash
hugo server          # local dev server
hugo --gc --minify   # production build (used by CI)
```

No Makefile, no package.json, no other build tooling.

## Content

- **Posts** go in `content/posts/`. Standalone articles.
- **Collections** go in `content/collections/<series>/`. Ordered article series with custom layouts; each article needs a `weight` field.
- All content uses **YAML frontmatter** (`---`), not TOML.
- Required frontmatter: `title`, `date`, `draft`, `tags`, `summary`.
- Site is bilingual (Chinese primary, English secondary). English posts add `"English"` tag.

## Gotchas

- Hugo version: **0.152.1 extended** (CI pins this exact version).
- `markup.goldmark.renderer.unsafe = true` — raw HTML in Markdown is intentional.
- Mermaid diagrams work via a custom code block renderer (`layouts/_default/_markup/render-codeblock-mermaid.html`).
- Images go in `static/images/`.
- Pushing to `main` triggers GitHub Pages deployment automatically.
