# ClaudeXCodex
# ⚡ ClaudeXCodex
A POC exploring how Claude (Anthropic) and Codex (OpenAI) can work in tandem to boost developer productivity.
## 🌐 Team Hub (GitHub Pages)
The site is published at:
**https://r-sadat-sayem.github.io/ClaudeXCodex/**
> **First-time setup:** Go to **Settings → Pages** in this repository and set **Source** to **GitHub Actions**. After that, every push to `main` will automatically deploy the site.
## 📁 Repository Structure
```
ClaudeXCodex/
├── index.html          ← Hub landing page
├── slides/             ← Presentation slide decks (.html)
├── docs/               ← Documentation pages (.html)
├── pages/              ← Standalone HTML pages
└── .github/
    └── workflows/
        └── pages.yml   ← Auto-deploy to GitHub Pages on push to main
```
## ➕ Adding Content
| Content type | Where to add the file | Section in `index.html` |
|---|---|---|
| Slide deck | `slides/your-deck.html` | Slides |
| Documentation | `docs/your-doc.html` | Documentation |
| Standalone page | `pages/your-page.html` | Pages |
See [`docs/getting-started.html`](docs/getting-started.html) for a full guide.
## 🖥️ Local Preview
No build step required — everything is plain HTML.
```bash
# Python
python -m http.server 8080
# Node.js
npx serve .
```
Then open http://localhost:8080 in your browser.