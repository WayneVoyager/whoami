# WayneVoyager's Blog

Where I throw coins and record both sides.

A personal blog built with [Astro](https://astro.build) and the [astro-whono](https://github.com/cxro/astro-whono) theme, deployed on [GitHub Pages](https://pages.github.com).

## About

I'm WayneVoyager. I write about product, AI, and things in life worth noting down. Every attempt gets recorded here, heads or tails.

## Project Structure

```text
├── public/               # Static assets (images, fonts, favicon)
│   ├── author/           # Author avatar
│   ├── bits/             # Bits (short notes) images
│   ├── fonts/            # Web fonts (subset WOFF2)
│   └── images/           # Post images
├── src/
│   ├── components/       # Reusable UI components
│   ├── content/          # Content collections
│   │   ├── essay/        # Long-form posts (航行日志)
│   │   ├── bits/         # Short notes (瓶中信)
│   │   └── memo/         # Memories (锚地)
│   ├── layouts/          # Page layouts
│   ├── pages/            # Routes and pages
│   ├── scripts/          # Client-side scripts
│   └── styles/           # Global styles and CSS
├── astro.config.mjs      # Astro configuration
├── site.config.mjs       # Site metadata and settings
└── package.json          # Dependencies and scripts
```

## Commands

| Command           | Action                                     |
| :---------------- | :----------------------------------------- |
| `npm install`     | Install dependencies                       |
| `npm run dev`     | Start local dev server at `localhost:4321`  |
| `npm run build`   | Build the production site to `./dist/`     |
| `npm run preview` | Preview the build locally before deploying |

## Deployment

This site auto-deploys to GitHub Pages via [GitHub Actions](.github/workflows/deploy.yml) on every push to `main`.

Live at: https://waynevoyager.github.io/whoami/

## Special Thanks

- [cxro/astro-whono](https://github.com/cxro/astro-whono) — the Astro theme this blog is built on
- [elizen/elizen-blog](https://github.com/elizen/elizen-blog) — design inspiration for the theme

## License

MIT
