# Plasmic + GitHub + Amplify Workflow Fixes

This document covers the fixes implemented to ensure smooth deployment of Plasmic projects to AWS Amplify (also works with Netlify and Vercel).

---

## Table of Contents

1. [Animation Fixes](#animation-fixes)
2. [Package Lock Sync Workflow](#package-lock-sync-workflow)
3. [How It Works](#how-it-works)
4. [Setup for New Projects](#setup-for-new-projects)

---

## Animation Fixes

### Problem 1: Horizontal Sliders Not Working

**Issue:** React-slick horizontal slider animations worked in Plasmic Studio but not in the deployed build.

**Cause:** The slick-carousel CSS files were not being loaded in the Next.js app.

**Solution:** Add CSS imports to `pages/_app.tsx`:

```tsx
import '@/styles/globals.css'
import "slick-carousel/slick/slick.css";
import "slick-carousel/slick/slick-theme.css";
import { PlasmicRootProvider } from "@plasmicapp/react-web";
import type { AppProps } from "next/app";
import Head from "next/head";
import Link from "next/link";

export default function MyApp({ Component, pageProps }: AppProps) {
  return (
    <PlasmicRootProvider Head={Head} Link={Link}>
      <Component {...pageProps} />
    </PlasmicRootProvider>
  );
}
```

---

### Problem 2: Stacking/Sticky Box Animations Not Working

**Issue:** Scroll-based stacking animations using `position: sticky` worked in Plasmic Studio but not in the deployed build.

**Cause:** CSS `overflow-x: hidden` on the `body` element breaks `position: sticky`.

**Solution:** In `styles/globals.css`, apply `overflow-x: hidden` only to `html`, not `body`:

```css
/* CORRECT - Only on html */
html {
  overflow-x: hidden;
}

body {
  max-width: 100vw;
  /* NO overflow-x: hidden here */
}
```

**WRONG (breaks sticky):**
```css
html, body {
  overflow-x: hidden; /* This breaks position: sticky */
}
```

---

## Package Lock Sync Workflow

### Problem

When Plasmic syncs to GitHub, it may add new dependencies to `package.json` but doesn't update `package-lock.json`. This causes Amplify builds to fail with:

```
npm error `npm ci` can only install packages when your package.json and package-lock.json are in sync.
npm error Missing: [package]@[version] from lock file
```

### Solution

A GitHub Action workflow that automatically syncs `package-lock.json` after every push.

**File:** `.github/workflows/sync-lockfile.yml`

```yaml
name: Sync package-lock.json

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write

jobs:
  sync-lockfile:
    runs-on: ubuntu-latest
    steps:
      - name: Check if should skip
        id: skip-check
        run: |
          if [[ "${{ github.event.head_commit.message }}" == *"chore: sync package-lock.json"* ]]; then
            echo "skip=true" >> $GITHUB_OUTPUT
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi

      - name: Checkout repository
        if: steps.skip-check.outputs.skip != 'true'
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        if: steps.skip-check.outputs.skip != 'true'
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Update package-lock.json
        if: steps.skip-check.outputs.skip != 'true'
        run: npm install --package-lock-only --legacy-peer-deps

      - name: Check for changes
        if: steps.skip-check.outputs.skip != 'true'
        id: check
        run: |
          if git diff --quiet package-lock.json; then
            echo "changed=false" >> $GITHUB_OUTPUT
          else
            echo "changed=true" >> $GITHUB_OUTPUT
          fi

      - name: Commit and push changes
        if: steps.skip-check.outputs.skip != 'true' && steps.check.outputs.changed == 'true'
        run: |
          git config --local user.email "github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          git add package-lock.json
          git commit -m "chore: sync package-lock.json with package.json"
          git push
```

### Key Features

- **Runs on every push to main** - Catches any sync issues
- **Manual trigger** - Can run from GitHub Actions tab via `workflow_dispatch`
- **Explicit permissions** - `contents: write` ensures the workflow can push changes
- **Skip check inside steps** - Handles null commit messages gracefully (avoids job-level errors)
- **Skip bot commits** - Avoids infinite loops by checking commit message
- **Legacy peer deps** - Handles React version conflicts with `--legacy-peer-deps`

---

## How It Works

### Plasmic to Amplify Flow

```
Plasmic Studio
     |
     | (Push changes)
     v
GitHub Repository
     |
     | (Triggers workflow)
     v
sync-lockfile.yml runs
     |
     | (Updates package-lock.json if needed)
     v
GitHub Repository (updated)
     |
     | (Webhook triggers build)
     v
AWS Amplify builds successfully
```

### VS Code to Amplify Flow

```
VS Code (local edits)
     |
     | (git push)
     v
GitHub Repository
     |
     | (Triggers workflow)
     v
sync-lockfile.yml runs
     |
     | (Updates package-lock.json if needed)
     v
GitHub Repository (updated)
     |
     | (Webhook triggers build)
     v
AWS Amplify builds successfully
```

---

## Setup for New Projects

When creating a new Plasmic project that deploys to Amplify/Netlify/Vercel:

### 1. Add the sync-lockfile workflow

Create `.github/workflows/sync-lockfile.yml` with the content shown above.

### 2. Fix _app.tsx for slick animations

Add slick-carousel CSS imports if using slider components:

```tsx
import "slick-carousel/slick/slick.css";
import "slick-carousel/slick/slick-theme.css";
```

### 3. Fix globals.css for sticky animations

Ensure `overflow-x: hidden` is only on `html`, not `body`:

```css
html {
  overflow-x: hidden;
}

body {
  max-width: 100vw;
  /* Do NOT add overflow-x: hidden here */
}
```

### 4. Push to GitHub

The workflow will automatically run and keep everything in sync.

---

## Platforms Supported

This workflow works with:
- AWS Amplify
- Netlify
- Vercel
- Any CI/CD platform that uses `npm ci`

---

## Troubleshooting

### Build still fails after workflow runs

1. Check if the workflow completed successfully in GitHub Actions tab
2. Manually trigger the workflow using "Run workflow" button
3. Verify the lock file was updated by checking the commit history

### Workflow creates infinite loop

The workflow checks if the commit message contains "chore: sync package-lock.json" and skips if true. If you see infinite loops, verify this condition is working.

### Animations still not working

1. Clear browser cache
2. Verify the CSS imports are in `_app.tsx`
3. Check browser dev tools for CSS loading errors
4. Verify `overflow-x: hidden` is NOT on `body` element

---

*Last updated: January 2026*
