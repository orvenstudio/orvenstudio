# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo hosts **static public-facing pages for orvenstudio's apps** — currently just the bilingual (한국어 / English) privacy policy for the **Sprig** app at `sprig/privacy-policy.html`. The Sprig mobile app's source code lives in a different repository; this repo only contains the legal/marketing surface served as static HTML.

The existing `README.md` is a GitHub profile placeholder and is not meaningful for working context.

## How pages are built and served

- Pages are **plain single-file HTML** with inline CSS and inline `<script>`. No build step, no bundler, no package manager, no tests.
- To preview changes, open the HTML file directly in a browser (`open sprig/privacy-policy.html`) — there is nothing to compile.
- The privacy policy is intended to be linked from inside the Sprig app and app-store listings, so **the file path / URL is externally referenced**. Don't rename or move `sprig/privacy-policy.html` without coordinating with the app side.

## Conventions when editing `privacy-policy.html`

- **Bilingual structure is load-bearing.** The page has three view modes toggled by `setLang('all' | 'ko' | 'en')`, which flips `body` classes `ko-only` / `en-only`. Korean content lives inside `.section-ko`, English inside `.section-en`. When adding or changing a clause, update **both** language sections to keep them in sync.
- **Effective Date / 시행일** in the header (currently "2025년 1월 1일 · January 1, 2025") should be bumped whenever the policy's substance changes.
- Design tokens (green palette, DM Serif Display + DM Sans, `--radius`, etc.) are defined as CSS custom properties under `:root` at the top of the `<style>` block — reuse them instead of introducing new one-off colors or fonts.
- The file is self-contained by design (inline styles + inline JS). Don't split it into external CSS/JS files unless the user explicitly asks — it's meant to be trivially serveable and viewable.

## What is NOT in this repo

If the user asks about Sprig's actual features, data-sync implementation, or Firebase setup, note that the app source is elsewhere — this repo only documents behavior via the privacy policy; it does not implement it. The `.gitignore` lists Flutter/Dart/Android/Xcode entries but there is no such code here; those entries are inherited boilerplate.
