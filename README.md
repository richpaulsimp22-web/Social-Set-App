# Socialset — v1 (MVP PWA)

A dedicated space for Instagram creators to organize content assets separately
from their camera roll. Not a scheduler, not a cloud drive — just a fast,
offline-first media organizer built for how creators actually work.

## What's in this project

```
socialset/
├── index.html      ← the entire app (UI + storage layer + logic)
├── manifest.webmanifest   ← PWA manifest (installable, standalone display)
├── sw.js           ← service worker (offline caching + app-shell fallback)
├── icons/
│   ├── icon.svg                 ← source icon
│   ├── icon-32.png              ← browser tab favicon
│   ├── icon-152.png, icon-180.png  ← iOS home screen icon
│   ├── icon-192.png, icon-384.png, icon-512.png ← Android/Chrome icons
│   └── icon-maskable-192.png, icon-maskable-512.png ← Android adaptive icon
└── README.md
```

### PWA / installability

This meets Chrome's installability checklist out of the box: served over
HTTPS (GitHub Pages), a linked manifest with `name`, `short_name`,
`start_url`, `display: standalone`, a 192px and 512px icon (plus maskable
variants for Android's adaptive icon shape), and a registered service
worker with a `fetch` handler. Chrome will offer its install prompt
automatically after a visit or two; you can also trigger it manually via
the browser menu → "Install app".

Once installed, it opens with no browser chrome (`standalone` display),
its own icon and splash/launch screen (Android builds this from the
manifest; iOS builds it from the icon + theme color via the
`apple-mobile-web-app-*` meta tags in `index.html`), and works fully
offline — reopening it with no signal still loads instantly because the
service worker falls back to the cached app shell instead of showing a
browser error page.


The app is intentionally a single dependency-free HTML file for v1 — no
build step, no framework, no bundler. That keeps it small, fast, and easy to
audit. The architecture is still cleanly layered (see below), so this can be
migrated into a framework later without redesigning anything.

## Running it locally

No build step is required. You just need to serve the folder over HTTP
(service workers and OPFS both require a real origin — `file://` won't work
for those two features, though the app's core UI will still render).

Any of these work:

```bash
# Option A: Python
cd socialset
python3 -m http.server 8080

# Option B: Node
npx serve socialset

# Option C: VS Code
# Right-click index.html → "Open with Live Server"
```

Then open `http://localhost:8080` on your phone (same Wi-Fi) or in your
desktop browser's device-emulation mode.

## Installing as an app

**iPhone (Safari):** open the URL → Share → *Add to Home Screen*.
**Android (Chrome):** open the URL → ⋮ menu → *Install app* (or you'll see an
automatic install prompt after a visit or two).

Once installed it launches full-screen with no browser chrome, per the
`"display": "standalone"` setting in `manifest.webmanifest`.

## Deploying

Any static host works: GitHub Pages, Netlify, Vercel, Cloudflare Pages, or
your own server. Upload the four items above (`index.html`, `manifest.webmanifest`,
`sw.js`, `icons/`) to the same directory — no server-side code needed.

## Architecture

**UI** — a small, framework-free render loop (state object → `render()` →
delegated click/input listeners). Chosen for v1 to keep the app light and
dependency-free; the storage layer below is UI-agnostic, so porting the UI to
React/Vue later doesn't touch storage code at all.

**Models** — `Folder` and `MediaItem` are plain objects (documented at the
top of `index.html`). Media *metadata* (filename, type, dates, ordering)
lives separately from the media *bytes*.

**Storage — the important part**

This is built as a two-piece storage layer, both hidden behind one
`StorageProvider` class that the UI talks to:

1. **MetadataStore** (IndexedDB) — folders and media *metadata only*: titles,
   formats, dates, ordering, cover references. No binary data.

2. **BlobStore** (private app storage) — the actual photo/video bytes. This
   uses the **Origin Private File System (OPFS)** via
   `navigator.storage.getDirectory()` when the browser supports it — a real
   sandboxed, private storage area for this PWA that is *not* the user's
   Photos/camera roll and not a visible downloads folder. If OPFS isn't
   available (some older WebViews), it falls back to a dedicated
   IndexedDB object store used *only* for blobs, so metadata and binary data
   stay logically separate either way.

Why this matters for what's next: because every screen only ever calls
`storage.getFolders()`, `storage.addMedia()`, `storage.getObjectURL()`, etc.,
swapping to cloud storage later means writing new classes with the *same
method names* — nothing in the UI changes:

```js
// today
const blobStore = new BlobStore(db);          // OPFS / IndexedDB

// later (drop-in replacements)
const blobStore = new FirebaseBlobStore(app);  // Firebase Storage
const blobStore = new SupabaseBlobStore(sb);   // Supabase Storage

const metadataStore = new FirestoreMetadataStore(app); // or SupabaseMetadataStore
```

`StorageProvider` is composed from these two pieces, so cloud sync becomes a
storage-layer swap, not an app rewrite — exactly as specced.

**Services** — `StorageProvider`'s methods already encode the app-level
rules (e.g. deleting a folder cascades to delete its media and blobs;
uploading the first item to a folder auto-sets it as the cover). This keeps
business logic out of the UI layer.

## Feature checklist (v1)

- Simple sign-in: name only, no password, stored in IndexedDB so it
  survives closing the app and reopening it later (Settings → Log out
  returns to the sign-in screen without deleting any data)
- Home screen: scrollable folder cards (cover, format, title, file count, date)
- Floating "+" action button → Create New Content (format + title only)
- Edit any folder's title/format later (pencil icon in the folder view)
- Folder view: media grid, upload from device, capture directly, delete,
  drag-to-reorder (press and drag any tile), choose cover, fullscreen viewer
- Everything you delete (a folder or a single photo/video) goes to
  **Recently Deleted** first, with an instant "Undo" on the toast — nothing
  is ever destroyed immediately. See "Undo & Recently Deleted" below.
- Share/export any photo/video (or a whole folder) straight to your
  device's native share sheet — the bridge into Instagram/CapCut, see
  "Getting content into Instagram or CapCut" below
- Search: by title, format, or date, instant filtering
- Sort: newest, oldest, alphabetical, format
- Bottom nav: Home / Search / Settings
- Settings: signed-in account + log out, Recently Deleted, app version,
  storage used, storage mode, export/import placeholders, "Cloud sync
  coming soon"
- 100% offline, no accounts server, no network calls

### About the sign-in

This is deliberately the simplest possible version: a name, no password, no
server. It exists so the app doesn't feel like a blank slate every time it's
reopened, and so there's a real "logged in / logged out" state to build real
accounts on top of later (Firebase Auth / Supabase Auth would slot in here
the same way the storage layer is designed to be swapped). Today all folders
are shared across whoever signs in on this device — full per-account data
separation is part of the "Accounts" future-expansion item.

The session record lives in IndexedDB (not `localStorage`), the same durable
on-device store the rest of your content lives in, so it survives closing
the app fully and reopening it days later — as long as you're opening it
from the *same URL, on the same device, in the same browser*.

### Undo & Recently Deleted

Every delete is a soft delete:

- Deleting a folder or a photo/video moves it to **Recently Deleted**
  (Settings → Recently Deleted) instead of erasing it.
- The toast that appears right after has an **Undo** button, active for a
  few seconds, for instantly reversing the action.
- Anything in Recently Deleted can be **Restored** at any time, or
  **Deleted forever** if you're sure.
- Items left in Recently Deleted are permanently removed automatically
  after 30 days (checked once each time the app opens).

The one exception is "Delete forever" inside Recently Deleted itself —
that's the one truly irreversible action in the app, and it's guarded by
its own confirmation step.

All confirmations (log out, delete, delete forever) use an in-app
confirmation sheet rather than the browser's native `confirm()` dialog,
which is unreliable or silently blocked inside some embedded/installed-app
browser contexts.

### Getting content into Instagram or CapCut

Every photo/video (and every folder, all at once) has a **Share** action
that hands the full-quality original file to your phone's native share
sheet via the Web Share API — the same sheet you'd see sharing a photo from
the Photos app.

Worth understanding upfront: a website or installed PWA **cannot** register
itself as a source *inside* another app's own upload picker (e.g. the
"select from library" screen you see when you start a new post in
Instagram, or importing media in CapCut/Edits). That kind of file-provider
integration is an OS-level capability that only true native apps can
register for — no PWA, on iOS or Android, can do it. That's a platform
limitation, not something specific to this app.

The practical, reliable bridge instead is:

1. In Socialset, open the photo/video (or the whole folder) → **Share**.
2. Your phone's native share sheet opens. Choose **Save to Photos** (iOS)
   or **Save to device / Save image/video** (Android) — or, if Instagram or
   CapCut appear directly as share targets on your phone, tap straight into
   them.
3. The file is now in your camera roll (or was sent directly), so it shows
   up normally the next time you tap "select from library" inside
   Instagram or CapCut.

It's one extra tap versus a native in-app picker, but it keeps your camera
roll clean the rest of the time — media only touches Photos at the moment
you're actually about to use it.

### Making it "remember you" reliably on your phone

IndexedDB and OPFS are tied to the exact origin (URL) the app is served
from, not to the app in the abstract — so for your sign-in and your folders
to reliably persist between visits:

1. **Deploy it once to a real URL** — GitHub Pages, Netlify, Vercel, etc.
   (see "Deploying" above). Opening a one-off preview link each time is not
   the same origin twice in a row and can behave inconsistently.
2. **Add it to your Home Screen from that URL** (Share → Add to Home
   Screen on iPhone). Opening it from the home-screen icon launches the
   same installed origin every time.
3. From then on, your name, folders, and media are saved on that device, in
   that browser, tied to that URL — closing the app, restarting your phone,
   or going offline doesn't clear any of it. The only things that would
   clear it are uninstalling/removing site data for that URL, or Safari's
   rare "hasn't been opened in X weeks" storage cleanup (opening the app at
   least occasionally prevents that).



## Intentionally not in v1

Per the brief, these are reserved for later versions and are **not**
implemented, though the storage/service layering above is built so they can
be added without a rewrite: Firebase/Supabase sync, accounts, partner
collaboration, PDF support, voice notes, scripts, captions, content
pipeline, publishing workflow, AI tagging, calendar, cloud backup.

## Notes on the icon

`icons/icon.svg` is the editable source (a rounded folder mark in the app's
coral→violet gradient). The three PNGs are pre-exported at the sizes
`manifest.webmanifest` expects. Regenerate anytime with:

```bash
pip install cairosvg
python3 -c "import cairosvg; cairosvg.svg2png(url='icons/icon.svg', write_to='icons/icon-512.png', output_width=512, output_height=512)"
```
