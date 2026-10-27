# CodeSage landing page

The website and documentation for [CodeSage](https://github.com/ntanwir10/codesage-algolia-challenge), live at **https://codesage-docs.vercel.app**.

It is built with [Astro](https://astro.build) and [Starlight](https://starlight.astro.build). The landing page is `src/pages/index.astro`, plus the hero in `src/components/Hero.astro`. The docs pages are Markdown and MDX under `src/content/docs/docs/`, and the sidebar order is set in `astro.config.mjs`.

## Run it

You need Node.js 22.23.4 or newer.

```bash
npm ci
npm run dev     # http://localhost:4321
npm run build   # static site in dist/, with the Pagefind search index
```

## Deploy

The Vercel project `codesage-docs` is connected to this repository. Each push to `main` deploys to production, and every other branch gets a preview deployment. To deploy without pushing, run this from this folder, which is linked through `.vercel/`:

```bash
vercel deploy --prod
```

## Writing pages

- **Diagrams:** in `.mdx`, use `<Mermaid code={`...`} caption="..." />` from `~/components/Mermaid.astro`. The diagrams redraw when the theme changes.
- **Screenshots:** use `<Shot src={img} alt="..." caption="..." />` from `~/components/Shot.astro`. The images live in `src/assets/screenshots/` and were captured from a local CodeSage run.
- **Accuracy:** facts come from the CodeSage source. When its README and its code disagree, follow the code.
