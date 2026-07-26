# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

LecturePaul ("Bibliothèque") is a personal reading-tracker web app, entirely French-language in its UI. The whole application — markup, styles, and logic — lives in a single file: `index.html`. There is no build step, no package manager, and no test suite.

## Repository layout

- `index.html` — the entire application (~1500 lines): inline `<style>` (a small hand-rolled utility-class CSS system, Tailwind-like naming but not Tailwind), inline error-handling bootstrap script, and the React app source inside `<script type="text/plain" id="app-source">`.
- `.nojekyll` — disables GitHub Pages' Jekyll processing so `index.html` is served as-is. This confirms the deployment target is GitHub Pages (or any static file host).

There is no `src/`, no `package.json`, no CI config, no test directory.

## Running / developing

There is no install or build command — this is not an npm project.

- **Open directly**: opening `index.html` as a `file://` URL works for most features (React/Babel/ZXing are loaded from `unpkg.com` CDNs at runtime, so an internet connection is required).
- **Camera / ISBN barcode scanner**: requires a secure context (HTTPS or `localhost`) per browser API restrictions — `file://` will not work for this one feature. Serve the directory locally to test it, e.g. `python3 -m http.server` then visit `http://localhost:8000/`.
- **No lint, no test, no build commands exist.** Verify changes by loading the page in a browser and exercising the feature manually (this app has no automated test coverage at all).

### How the app boots

`index.html` does not ship pre-compiled JS. The React component tree is written as JSX inside a `<script type="text/plain" id="app-source">` block (inert to the browser). A trailing `<script>` at the bottom of the file reads that text content, runs it through `Babel.transform` (loaded from `@babel/standalone` via CDN) at page-load time, and injects the resulting compiled code as a live `<script>` tag. Any Babel compile error is caught and rendered into the `#fallback-error` div instead of failing silently — check that div (and the `window.onerror`/`unhandledrejection` handlers wired near the top of the file) when debugging a blank page.

When editing app logic, you are editing **JSX inside that inert `<script type="text/plain">` block**, not a normal `<script>` — keep edits inside that block's boundaries.

## Architecture

Everything is one component tree defined with React 18 (UMD build, `React.createElement` via classic JSX runtime) and no JSX/TS tooling beyond in-browser Babel:

- **`App()`** is a single, large component holding essentially all state (book list, categories, current view, form state, filters, dashboard filters, import/sync/scanner state) via `useState`/`useMemo`/`useRef`. There is no router, no context, no state management library — view switching is done with a `view` string state (`'list' | 'stats' | 'settings' | 'form'`).
- Small presentational components (`Cover`, `Pill`, `StarCard`/`StatCard`, `BarRow`, `StarRating`, the `Icon.*` SVG set) are defined above `App` and are pure/stateless aside from `StarRating`/`Cover`'s own local UI state.
- **Design tokens** live in the top-level `COLORS` object and the `STATUS_OPTIONS` array (the five reading-status states: `a_acheter`, `a_lire`, `en_cours`, `lu`, `abandonne`, each with a label and color) — reuse these rather than hardcoding colors or status strings elsewhere.

### Data model & persistence

A "book" is a plain object built by `emptyBook()` (title, author, pageCount, isbn, status, rating, categories, cover, startDate, endDate, pagesRead, comment, description).

Persistence is layered, and the layering matters:
1. **`localStorage` (key `STORAGE_KEY = 'biblio-data'`) is the source of truth**, written synchronously on every mutation via `persist(books, categories)`. It also stores the Google Books API key and Airtable token (client-side only — this is a personal single-user tool, not a hardened app).
2. **Airtable is a manual, explicit backup/restore target only** — `pushToAirtable()` and `pullFromAirtable()` are user-triggered, each behind a `window.confirm`, and each is destructive to one side (push wipes-then-recreates all Airtable records; pull replaces all local books). There is no automatic/background sync, by design (to avoid burning the free-tier Airtable quota). Field mapping between the local book shape and Airtable columns lives in `recordToBook()` / `bookToFields()` — keep these in sync if the book shape changes.
3. **CSV export/import** (`exportData()` / `importData()`) is an additional manual backup path with its own hand-rolled CSV parser/escaper (semicolon-delimited, `CSV_HEADERS` constant defines column order — update it alongside the book shape).

### External integrations (all called directly from the browser, no backend)

- **Google Books API** (`fetchGoogleBooks`, `runImportSearch`, `fetchMissingCovers`, `lookupIsbnSilently`) — metadata/cover search by title/author/ISBN. Retries only on HTTP 503. An optional user-supplied API key raises the rate limit.
- **MyMemory translation API** (`translateToFrench`) — free, keyless; used to auto-translate Google Books descriptions to French when the source language isn't French. Long text is chunked by UTF-8 byte length (`splitIntoChunks`, ~450 bytes/request) because the API has a per-request size limit.
- **ZXing** (`@zxing/library` via CDN) — camera-based ISBN barcode scanning (`showScanner` flow). Only works in a secure context; failures are surfaced as French user-facing error strings rather than thrown.

## Conventions specific to this codebase

- **User-facing strings, comments, and Airtable field names are French**; JS identifiers (variables, functions, component names) are English. Keep this split when adding code.
- Comments are sparse and only explain non-obvious *why* (e.g. why Airtable sync is manual-only, why retries only happen on 503, why cover-image load errors are swallowed). Match that style rather than adding narrative comments.
- Styling uses the bespoke utility classes defined in the `<style>` block (e.g. `.flex`, `.gap-2`, `.rounded-lg`) plus inline `style={{ ... }}` for anything token-driven (colors, dynamic widths/heights). There is no Tailwind build — if a utility class you need doesn't exist in the `<style>` block, add it there rather than assuming it's available.
- Destructive user actions (deleting all books, overwriting Airtable in either direction) must stay behind a `window.confirm` — this is the app's only safeguard against data loss given there's no undo.
