# FROSTLINE — Admin Dashboard

A single self-contained HTML file — the internal dashboard for reviewing submissions from the FROSTLINE client intake form, updating their status, and downloading attachments.

This is one half of the FROSTLINE order system. The other half, the client-facing form, lives in a separate project/repo (`frostline-client-form`). They're split into two projects on purpose — see **"Why two projects"** below.

> ⚠️ **There is no login on this dashboard.** See "Before you share this URL" below before deploying this for real client data.

## Deploy to Vercel

**Drag and drop (fastest):** go to [vercel.com/new](https://vercel.com/new) and drag this folder onto the page.

**CLI:**
```bash
npm i -g vercel
vercel
```

**GitHub:** push this folder to its own repo, then import it in the Vercel dashboard.

## Run it locally

Open `index.html` directly in a browser — no server required.

## Why two projects

Splitting the client form and this dashboard into separate Vercel projects/repos costs nothing technically — both files are fully self-contained with zero shared dependencies. The benefit: they get separate URLs, separate deploy history, and you can apply different access rules to each independently later (e.g. an IP allowlist or a reverse proxy in front of just this one) without touching the public form at all.

They both still talk to the **same Firebase project** — see below.

## Connecting to Firebase

1. Create a Firebase project at [console.firebase.google.com](https://console.firebase.google.com) (skip this if you already made one for the client form).
2. Enable **Firestore Database** (production mode). That's the only Firebase product this needs — see the note below on why Storage isn't required.
3. In Firebase Console → Project settings → Your apps, create a **Web app** and copy its config object.
4. Open `index.html` here, find the `firebaseConfig` block near the top of the inline `<script type="module">`, and paste your config in.
5. **Paste the exact same config into the client-form project too** — both frontends must point at the same Firebase project, or submissions won't show up here.
6. Deploy the included Firestore rules:
   ```bash
   firebase deploy --only firestore:rules
   ```
   (or paste `firestore.rules` into the Firebase Console's Rules editor).

Once `firebaseConfig` has real values, this dashboard automatically switches from demo mode (localStorage) to reading live from Firestore, with near-real-time updates via `onSnapshot`.

### Why no Firebase Storage / Blaze plan needed

As of February 2026, Firebase requires the pay-as-you-go **Blaze plan** (a linked billing account) to use Cloud Storage at all — even to stay within its free tier. To avoid that requirement, this app stores uploaded files as base64 data directly in Firestore (in a `files` subcollection under each submission) instead of Cloud Storage, which stays entirely within Firestore's free Spark-plan quota.

The trade-off: Firestore caps each document at ~1MB, so files are limited to roughly **500KB each** after compression. Images are compressed automatically at upload time; non-image files (PDFs, spreadsheets) need to be under that size as-is. `storage.rules` is included but unused for now — if you later upgrade to Blaze anyway, flip `USE_FIREBASE_STORAGE` to `true` in `firebase-config.js`, enable Storage in the Firebase Console, and deploy `storage.rules` too. No other code changes needed, and the file-size limit goes back up to 10MB.

## Before you share this URL

There's no admin sign-in by design (removed on request). That means:

- Anyone with this dashboard's URL can see every submission and its attachments.
- Anyone who calls the Firebase API directly with this project's public web config — which isn't a secret, it's visible in any browser's dev tools — can read and update submissions too. `firestore.rules` is the only real gate, and it's currently open (`allow read: if true`).

If this data is sensitive, protect the route itself rather than relying on the rules alone:

- **Vercel's native password protection** requires a Pro plan plus a $150/month add-on, or Enterprise — not available cheaply on the free Hobby plan for a production domain. Splitting into its own project doesn't unlock this for free; it just makes it *possible* to apply later without touching the client form.
- **The practical free option** is reintroducing a real Firebase Authentication gate on this dashboard (a proper sign-in, not the old local-only fake one) and switching `firestore.rules` / `storage.rules` back to requiring a signed-in user for reads/updates. Ask for this if you want it added back.
- A lightweight alternative: an IP allowlist or HTTP basic-auth proxy in front of just this Vercel project, if your team always connects from known locations.

## Data model

Collection: **`submissions`** (Firestore), or the `frostline_submissions_v1` key in `localStorage` in demo mode:

| Field                    | Type                                       | Notes                                                |
|--------------------------|---------------------------------------------|-------------------------------------------------------|
| `teamName`               | string                                       | required                                               |
| `whatToDesign`           | array of strings                             | required — checklist selections, at least one          |
| `deadline`               | string (`"YYYY-MM-DD"`)                      | optional                                               |
| `designInspo`            | string                                       | optional — free-text notes to supplement the files     |
| `inspoAttachments`       | array of `{ name, type, size, fileId }`      | required — at least one file, up to 5, ~500KB each      |
| `logoAttachments`        | array of `{ name, type, size, fileId }`      | optional, up to 2 files, ~500KB each                    |
| `masterlistAttachments`  | array of `{ name, type, size, fileId }`      | optional, up to 3 files, ~500KB each                    |
| `additionalNotes`        | string                                       | optional                                               |
| `status`                 | `"new"` \| `"in_review"` \| `"completed"`    | defaults to `"new"` on create                          |
| `createdAt` / `updatedAt`| timestamp (ms)                               | set on create; `updatedAt` also changes on status update |

Each attachment references a document in that submission's `files` subcollection (`submissions/{submissionId}/files/{fileId}`), which holds the actual base64 file data. The dashboard fetches a file's data on demand — only when you click Download — rather than loading every file's contents up front. (If `USE_FIREBASE_STORAGE` is `true`, attachments instead carry a plain `url` pointing at Cloud Storage — same shape everywhere else.)

## Security notes

- **Sanitize on write, not just on read.** The Firestore rules cap field lengths and required fields, but don't scan content. If this data is ever rendered outside this dashboard's plain-text bindings, escape/sanitize `teamName`, `whatToDesign`, `designInspo`, and `additionalNotes` again there.
- **File uploads are validated by declared MIME type**, not by inspecting file bytes. Add a Cloud Function that re-validates uploaded files after the fact if you need stronger guarantees.
- Consider [Firebase App Check](https://firebase.google.com/docs/app-check) on the client form to cut down on spam/bot submissions, since it has no login by design.
