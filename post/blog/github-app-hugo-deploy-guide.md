---
authors:
  - matt2ology
categories:
  - blog
  - tutorial-guide
date: 2025-09-01T14:35:02-07:00
draft: false
tags:
  - Hugo
  - GitHub
  - Markdown
  - Automation
title: "Step-by-step Guide: Replace PAT with GitHub App + Automate Markdown + Deploy Hugo"
---
## Step-by-step Guide: Replace PAT with GitHub App + Automate Markdown + Deploy Hugo

This guide shows how to:

- Format Markdown in your **notes repo** (`obsidian-notes-repo` - your vault)
- Push changes into your **site repo** (`username/username.github.io`)
- Build & deploy Hugo to GitHub Pages
- Use a **GitHub App** instead of a PAT (no more token rotation)
- Ignore Obsidian templates in Hugo builds

---

### Prepare Repos

- **`obsidian-notes-repo`**: your notes & content
- **`username/username.github.io`**: Hugo site

Why: Keeps raw notes separate from your published site.

---

### Create a Minimal GitHub App (one-time)

1. Go to [GitHub → Settings → Developer settings → GitHub Apps](https://github.com/settings/apps).
2. Click **New GitHub App**.
3. Fill in:

   - **GitHub App name** → e.g. `Notes Publisher`
   - **Homepage URL** → `https://github.com/username/username.github.io` (just a link to your site repo)
   - **Callback URL** → leave blank
   - **Webhook URL** → leave blank

4. Permissions:

   - **Repository → Contents: Read & write**
   - **Metadata: Read-only**

5. Save.

---

### Install the App on the Site Repo

1. Open your new App in [GitHub Apps settings](https://github.com/settings/apps).
2. Click **Install App** (left sidebar).
3. Choose your personal account.
4. Select **Only select repositories** → choose `username/username.github.io`.
5. Confirm installation.

---

### Generate and Save App Credentials

1. In your App settings → **Generate a private key**. Save the `.pem` file.
2. Copy your **App ID** (number shown at the top).
3. Go to **`username/username.github.io`** (`username.github.io`) → `Settings → Secrets and variables → Actions → New repository secret`:

   - `APP_ID` → the numeric App ID of your GitHub App (from the App settings page, not Client ID).
   - `PRIVATE_KEY` → generate a **Private Key** if you don’t already have one. Save the PEM file, open it, and paste the entire contents (including `-----BEGIN PRIVATE KEY-----` … `-----END PRIVATE KEY-----`) into a secret called `PRIVATE_KEY`.

✅ This allows Actions to authenticate as the GitHub App.

---

### Configure Markdown Formatter Workflow (`obsidian-notes-repo`)

In `obsidian-notes-repo/.github/workflows/format-and-push.yaml`:

```yaml
name: Format Markdown and Push to Site Repo

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  format:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Install Prettier
        run: npm install -g prettier

      - name: Run Prettier on Markdown
        run: prettier --write "**/*.md"

      - name: Commit formatted changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add .
          git diff-index --quiet HEAD || git commit -m "chore: format markdown"

      - name: Generate GitHub App token
        uses: tibdex/github-app-token@v2
        id: generate-token
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}

      - name: Push changes to site repo
        run: git push https://x-access-token:${{ steps.generate-token.outputs.token }}@github.com/username/username.github.io main
```

---

### Update Hugo Build Workflow (`username/username.github.io`)

Use your updated `.github/workflows/hugo.yaml` (the one with Dart Sass, Go, Hugo, Node, caching, and deploy to GitHub Pages).
That workflow builds & deploys the site.

```yml
name: Build and Deploy Hugo Site
on:
  workflow_run:
    workflows: ["Update Submodule Reference"]
    types: [completed]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.90.0
      GO_VERSION: 1.24.5
      HUGO_VERSION: 0.148.2
      NODE_VERSION: 22.18.0
      TZ: Europe/Oslo
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Create directory for user-specific executable files
        run: |
          mkdir -p "${HOME}/.local"
      - name: Install Dart Sass
        run: |
          curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"
      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Verify installations
        run: |
          echo "Dart Sass: $(sass --version)"
          echo "Go: $(go version)"
          echo "Hugo: $(hugo version)"
          echo "Node.js: $(node --version)"
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true
      - name: Configure Git
        run: |
          git config core.quotepath false
      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys: hugo-
      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
      - name: Cache save
        if: always()
        id: cache-save
        uses: actions/cache/save@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

### Update Hugo Config to Ignore Obsidian Templates

In `username.github.io/hugo.toml` add:

```toml
[module]
  [[module.mounts]]
    source = "content"
    target = "content"
    excludeFiles = ["templates/*"]
```

This prevents Hugo from building Obsidian’s internal `templates/` folder.

---

### Verify Setup

1. Push a note to **`obsidian-notes-repo`**.
2. Workflow runs → formats Markdown → pushes to **`username/username.github.io`**.
3. Hugo workflow runs in **`username/username.github.io`** → builds & deploys to GitHub Pages.
4. Your site updates automatically, with clean Markdown and no Obsidian templates included.

---

✅ Now you now have a fully automated, GitHub-App-powered publishing pipeline.
