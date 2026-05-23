# Folio — Book Finder & Reading List

A mobile-first web app for searching books and tracking what you want to read or have already read. Search by title or author, save books to a personal reading list, and toggle their status between **Want to Read** and **Read**. Your list persists across sessions via Firebase Firestore.

**Live demo structure:** single `index.html` file, no build step, deployable to GitHub Pages in minutes.

---

## Features

- **Search** any book title, author, or keyword via the [OpenLibrary API](https://openlibrary.org/developers/api) — no API key required
- **Cover images** fetched automatically; graceful SVG placeholder generated from title initials when no cover exists
- **Save books** to a personal Firestore reading list with one tap
- **Status toggle** — flip a book between *Want to Read* and *Read ✓* at any time
- **Remove books** from the list with the ✕ button
- **Filter chips** to view All, Want to Read, or Read books
- **Persistent storage** — Firestore keeps your list across page refreshes and devices
- **Mobile-first** responsive layout; works on any screen size

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, Vanilla JavaScript (ES modules) |
| Database | Firebase Firestore (v10, CDN) |
| Book search | OpenLibrary Search API |
| Fonts | Playfair Display + DM Sans (Google Fonts) |
| Hosting | GitHub Pages |

---

## Project Structure

```
/
└── index.html      # Entire app — markup, styles, and logic in one file
└── README.md
└── AGENTS.md
```

---

## Getting Started

### Prerequisites

- A [Firebase](https://firebase.google.com) account (free Spark plan is sufficient)
- A GitHub account for hosting

### 1. Create a Firebase Project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and click **Add project**
2. Give it a name (e.g. `folio-books`) and follow the setup wizard
3. In the left sidebar, click **Build → Firestore Database**
4. Click **Create database**, choose a region close to your users, and select **Start in test mode**
   > Test mode allows all reads and writes for 30 days. See [Security Rules](#firestore-security-rules) below before going to production.
5. Go to **Project Settings** (gear icon) → **Your apps** → click the **`</>`** (Web) icon
6. Register your app with a nickname, then copy the `firebaseConfig` object shown

### 2. Add Your Firebase Config

Open `index.html` and find the config block near the top of the `<script type="module">` tag. Replace the placeholder values:

```js
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT.firebaseapp.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId:             "YOUR_APP_ID"
};
```

### 3. Deploy to GitHub Pages

1. Push the repository to GitHub
2. Go to **Settings → Pages**
3. Under **Source**, select your branch (`main`) and folder (`/ (root)`)
4. Click **Save** — your app will be live at `https://<username>.github.io/<repo-name>`

---

## How It Works

### Search Flow

1. User types a query and submits the search form
2. App fetches `https://openlibrary.org/search.json?q=<query>&limit=20&fields=key,title,author_name,cover_i,first_publish_year`
3. Results are rendered as cards with cover art, title, author, and year
4. Books already in the reading list show a disabled **Saved ✓** button

### Save Flow

1. User clicks **Save** on a search result
2. A document is written to the `books` Firestore collection with fields: `olKey`, `title`, `author`, `cover`, `year`, `status` (`"want"`), `savedAt` (Unix timestamp)
3. The card in search results updates to **Saved ✓** immediately (optimistic UI)

### Reading List Flow

1. On page load, all documents in `books` are fetched and sorted by `savedAt` descending
2. Each book card shows a status pill (**Want to Read** / **Read ✓**) — clicking it calls `updateDoc` to toggle the `status` field
3. The ✕ button calls `deleteDoc` to remove the entry
4. Filter chips perform client-side visibility toggling (no extra Firestore reads)

---

## Firestore Data Model

**Collection:** `books`

| Field | Type | Description |
|---|---|---|
| `olKey` | string | OpenLibrary work key (e.g. `/works/OL45804W`) used to deduplicate |
| `title` | string | Book title |
| `author` | string | Comma-separated author name(s) |
| `cover` | number \| null | OpenLibrary cover ID; `null` if unavailable |
| `year` | number \| string | First publish year |
| `status` | `"want"` \| `"read"` | Reading status |
| `savedAt` | number | Unix timestamp (ms) used for ordering |

---

## Firestore Security Rules

The default test-mode rules expire after 30 days. For a personal app, replace them in the Firebase Console under **Firestore → Rules**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /books/{bookId} {
      allow read, write: if true; // open — fine for a personal/private deployment
    }
  }
}
```

For a multi-user app, add Firebase Authentication and scope rules to `request.auth.uid`.

---

## Local Development

No build tools needed. Just open `index.html` in a browser — but note that Firebase module imports require a server context (not `file://`). Use any static server:

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .
```

Then open `http://localhost:8080`.

---

## Customisation Notes

- **Result limit** — change `&limit=20` in the fetch URL to return more or fewer results
- **Fields fetched** — extend the `&fields=` parameter with any [OpenLibrary field](https://openlibrary.org/developers/api) (e.g. `subject`, `isbn`)
- **Statuses** — add more status values (e.g. `"reading"`) by extending the `toggleStatus` function and adding corresponding CSS classes
- **Firebase version** — pinned to `10.12.0` via CDN; update the version string in the two import URLs to upgrade

---

## License

MIT — do whatever you like with it.
