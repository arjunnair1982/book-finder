# AGENTS.md — Folio Book Finder

This file gives AI coding agents the context needed to work on this codebase safely and effectively.

---

## Project Overview

Folio is a single-file (`index.html`) mobile-first web app. There is no build step, no bundler, and no framework. All HTML, CSS, and JavaScript live in one file. Firebase is loaded via CDN ES module imports; no `npm` or `package.json` is involved.

---

## Repository Layout

```
/
├── index.html      # The entire application
├── README.md       # Setup and usage documentation
└── AGENTS.md       # This file
```

Do not create additional files unless explicitly asked. The single-file constraint is intentional — it keeps GitHub Pages deployment trivial.

---

## Architecture at a Glance

```
index.html
├── <head>
│   ├── Google Fonts (Playfair Display, DM Sans)
│   └── <script type="module">          ← all JS lives here
│       ├── Firebase imports (CDN)
│       ├── firebaseConfig               ← user-supplied credentials
│       ├── State: readingList[], searchResults[]
│       ├── Helpers: coverUrl(), placeholderSvg()
│       ├── Firebase ops: loadList(), saveBook(), toggleStatus(), removeBook()
│       ├── Search: doSearch()
│       ├── Renderers: renderResults(), renderList(), updateSavedBadges()
│       ├── Toast: showToast()
│       └── DOMContentLoaded: tab switching, form submit, filter chips
└── <body>
    ├── <header>        — sticky header with logo + tab bar
    ├── <main>
    │   ├── #pane-search — search form + results
    │   └── #pane-list   — reading list + filter chips
    └── #toast          — global notification element
```

---

## Key Invariants

These must be preserved when making any change:

1. **No build step.** Never introduce `npm`, Webpack, Vite, or any bundler. All dependencies must be loadable via CDN `import` or `<script src>`.

2. **Single file.** All code stays in `index.html`. Do not split into separate `.js` or `.css` files unless the user explicitly requests it and understands it will require a server to run locally.

3. **Firebase SDK version is pinned.** The two CDN import URLs both reference `firebase@10.12.0`. If upgrading, both lines must be updated together and tested — the SDK has breaking changes between major versions.

4. **OpenLibrary key field is `olKey`.** The `olKey` field (the OpenLibrary `/works/OLxxxxW` path) is the deduplication key. `saveBook()` checks for an existing `olKey` before writing. Any code that touches book objects must preserve this field name.

5. **Firestore collection name is `"books"`.** All four Firestore operations (`loadList`, `saveBook`, `toggleStatus`, `removeBook`) target this collection. Changing the name requires updating all four.

6. **`savedAt` is a Unix timestamp in milliseconds** (`Date.now()`), not a Firestore `serverTimestamp()`. The list is ordered by this field descending. Keep it as a plain number.

7. **`status` is either `"want"` or `"read"`.** `toggleStatus()` flips between the two. CSS classes `status-want` and `status-read` are paired to these values. Adding a third status requires updating `toggleStatus`, `bookCard`, and the CSS.

8. **Global event handlers are attached via `window._*`.** Because card HTML is built with `innerHTML`, click handlers are assigned as `window._saveBook`, `window._toggle`, and `window._remove`. These window assignments must stay in sync with the `onclick` attribute strings inside `bookCard()`.

---

## Common Tasks

### Add a new book field (e.g. subject/genre)

1. Extend the `&fields=` query param in `doSearch()` to include the new OpenLibrary field name
2. Map it onto the result object inside the `.map(d => ({...}))` call in `doSearch()`
3. Pass it through `saveBook()` — it is already spread into the Firestore payload via `{ ...book, ... }`
4. Render it in `bookCard()` inside the `.book-info` div
5. No Firestore schema change needed — Firestore is schemaless

### Add a third reading status (e.g. "Currently Reading")

1. Update `toggleStatus()` — change the binary flip to a three-way cycle: `want → reading → read → want`
2. Add a `status-reading` CSS class in the `<style>` block with a distinct color
3. Update the `statusLabel`/`statusClass` logic in `bookCard()` to handle the new value
4. Update the filter chips in `<body>` and the corresponding event listener in the `DOMContentLoaded` block

### Change the search result limit

Find this line in `doSearch()`:

```js
const res = await fetch(`https://openlibrary.org/search.json?q=${...}&limit=20&fields=...`);
```

Change `&limit=20` to any value up to 100 (OpenLibrary's maximum per request).

### Add pagination to search results

OpenLibrary supports `&page=N` alongside `&limit=N`. To implement:
1. Track a `currentPage` variable in module scope
2. Add a **Load more** button below `#search-results`
3. On click, increment `currentPage`, fetch with `&page=${currentPage}`, and append (not replace) results to `searchResults` and the DOM

### Upgrade Firebase SDK

Replace both CDN version strings (currently `10.12.0`) with the new version:

```js
// Line ~15-18 — two import URLs, both must match:
import { initializeApp } from "https://www.gstatic.com/firebasejs/NEW_VERSION/firebase-app.js";
import { ... } from "https://www.gstatic.com/firebasejs/NEW_VERSION/firebase-firestore.js";
```

Check the [Firebase JS SDK changelog](https://firebase.google.com/support/release-notes/js) for breaking changes before upgrading.

---

## What Not to Do

- **Do not remove the `type="module"` attribute** from the `<script>` tag. Firebase's CDN imports require ES module context; removing it breaks all Firebase functionality.
- **Do not inline the Firebase config** into version control with real credentials. The placeholder strings (`"YOUR_API_KEY"` etc.) are intentional — users supply their own.
- **Do not use `localStorage` or `sessionStorage`** as a data store. Firestore is the source of truth; local state (`readingList[]`) is a runtime cache only.
- **Do not replace `innerHTML` rendering with a framework** without confirming the user wants to change the no-build-step constraint.
- **Do not add authentication** without also updating Firestore security rules — adding auth while rules remain open is a no-op security-wise.

---

## Testing Checklist

There are no automated tests. When verifying a change manually, confirm:

- [ ] Search returns results for a known title (e.g. "Dune")
- [ ] Cover images load; placeholder SVG appears for books without covers
- [ ] Saving a book adds it to the reading list tab and disables the Save button in search results
- [ ] Saving the same book a second time shows "Already in your list!" toast and does not duplicate
- [ ] Status pill toggles correctly between Want to Read and Read ✓
- [ ] Remove (✕) deletes the card and decrements the list badge
- [ ] Filter chips correctly show/hide cards by status
- [ ] Page refresh restores the full reading list from Firestore (the core persistence check)
- [ ] Toast messages appear and auto-dismiss after ~3 seconds
- [ ] Layout looks correct on a 375 px wide viewport (iPhone SE size)

---

## External API Reference

**OpenLibrary Search**
- Endpoint: `https://openlibrary.org/search.json`
- Params used: `q` (query string), `limit` (max results), `fields` (comma-separated field list)
- No authentication required
- Rate limits: not officially documented; be respectful — avoid firing on every keystroke

**OpenLibrary Cover Images**
- URL pattern: `https://covers.openlibrary.org/b/id/{cover_id}-{size}.jpg`
- Sizes: `S` (small), `M` (medium), `L` (large)
- `cover_i` in search results maps to `{cover_id}`

**Firebase Firestore (CDN)**
- SDK: `firebase@10.12.0`
- Functions used: `initializeApp`, `getFirestore`, `collection`, `addDoc`, `getDocs`, `updateDoc`, `deleteDoc`, `doc`, `query`, `orderBy`
