# Repository Guidelines

## Project Structure & Module Organization
This repository is currently lightweight: the tracked root files are `README.md` and `LICENSE`. The `README.md` also defines the intended GitHub Pages layout for future content:

- `index.html` for the hub landing page
- `slides/` for HTML slide decks
- `docs/` for documentation pages
- `pages/` for standalone HTML pages
- `.github/workflows/pages.yml` for Pages deployment

When adding new content, follow that structure and keep links relative to the repository root so GitHub Pages publishes correctly.

## Build, Test, and Development Commands
There is no build pipeline today; this is a plain HTML and Markdown repository.

- `python -m http.server 8080` starts a local preview server
- `npx serve .` provides the same preview flow through Node.js
- `git status` verifies you are only committing intended files

Use local preview before opening a PR, especially for navigation, asset paths, and broken links.

## Coding Style & Naming Conventions
Keep Markdown and HTML simple, readable, and semantic.

- Use concise headings and short sections
- Prefer lowercase, hyphenated file names such as `getting-started.html`
- Keep directory placement consistent with content type: slides in `slides/`, docs in `docs/`, general pages in `pages/`
- Use relative links and repository-safe asset paths

Match the surrounding style in existing files rather than introducing a new format.

## Testing Guidelines
No automated test suite is configured yet. Validation is manual:

- Preview locally in a browser
- Check internal links and asset references
- Confirm GitHub Pages content works without a build step
- For UI changes, verify desktop and mobile rendering

If you add JavaScript or a larger content surface, document any new validation steps in `README.md`.

## Commit & Pull Request Guidelines
Recent history uses short, imperative commit subjects such as `Enhance README with GitHub Pages hub info` and `Initial plan`. Follow that pattern.

- Keep commits focused on one logical change
- Use PR descriptions that summarize user-visible impact
- Link related issues when applicable
- Include screenshots for layout or visual page changes
- Note any new files, routes, or deployment implications

## GitHub Pages Notes
This repository is intended for GitHub Pages publishing. Keep generated or external build artifacts out of version control unless the deployment workflow explicitly requires them.
