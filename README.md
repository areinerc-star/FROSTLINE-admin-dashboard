# FROSTLINE — Admin Dashboard

A single self-contained HTML file — the internal dashboard for reviewing submissions from the FROSTLINE client intake form, updating their status, and downloading attachments.

This is one half of the FROSTLINE order system. The other half, the client-facing form, lives in a separate project/repo (`frostline-client-form`). They're split into two projects on purpose — see **"Why two projects"** below.

> **This dashboard requires a real Firebase sign-in** to view or manage submissions — see "Admin sign-in" below for how to create an account.

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
2. Enable **Firestore Database** (production mode) and **Authentication** (Email/Password sign-in method) — see the note below on why Storage isn't required.
3. In Firebase Console → Project settings → Your apps, create a **Web app** and copy its config object.
4. Open `index.html` here, find the `firebaseConfig` block near the top of the inline `<script type="module">`, and paste your config in.
5. **Paste the exact same config into the client-form project too** — both frontends must point at the same Firebase project, or submissions won't show up here.
6. Deploy the included Firestore rules:
   ```bash
   firebase deploy --only firestore:rules
   ```
   (or paste `firestore.rules` into the Firebase Console's Rules editor).
7. Create your admin account: Firebase Console → Authentication → Users → Add user (email + password). That's what you'll sign in with here.

Once `firebaseConfig` has real values, this dashboard automatically switches from demo mode (localStorage) to reading live from Firestore, with near-real-time updates via `onSnapshot` — and requires the sign-in you just created.

### Why no Firebase Storage / Blaze plan needed

As of February 2026, Firebase requires the pay-as-you-go **Blaze plan** (a linked billing account) to use Cloud Storage at all — even to stay within its free tier. To avoid that requirement, this app stores uploaded files as base64 data directly in Firestore (in a `files` subcollection under each submission) instead of Cloud Storage, which stays entirely within Firestore's free Spark-plan quota.

The trade-off: Firestore caps each document at ~1MB, so files are limited to roughly **500KB each** after compression. Images are compressed automatically at upload time; non-image files (PDFs, spreadsheets) need to be under that size as-is. `storage.rules` is included but unused for now — if you later upgrade to Blaze anyway, flip `USE_FIREBASE_STORAGE` to `true` in `firebase-config.js`, enable Storage in the Firebase Console, and deploy `storage.rules` too. No other code changes needed, and the file-size limit goes back up to 10MB.

## Admin sign-in

This dashboard requires a real Firebase Authentication account — reading submissions, changing status, and deleting are all blocked server-side (in `firestore.rules`) for anyone who isn't signed in, not just gated by the UI.

To create an admin account:

1. Firebase Console → Authentication. If you haven't used Authentication in this project yet, click **Get started** and enable the **Email/Password** sign-in method.
2. Go to the **Users** tab → **Add user** → enter an email and password. That's the login for this dashboard.
3. Repeat step 2 for anyone else on your team who needs access — each person should have their own account rather than sharing one.

Anyone signed in has full access (view, change status, delete). For different permission levels per person, switch to Firebase custom claims and adjust the checks in `firestore.rules` and `storage.rules` accordingly — they currently just check "signed in or not."

The client form never requires sign-in — this only applies to the dashboard.

## Before you share this URL

With sign-in enforced server-side, sharing the dashboard's URL alone doesn't expose anything — someone would also need valid admin credentials. Still worth knowing:

- Anyone who calls the Firebase API directly with this project's public web config (not a secret — visible in any browser's dev tools) is subject to the exact same `firestore.rules` checks as the dashboard UI. There's no separate, weaker path in.
- Every signed-in user currently has equal access, including delete. If you add teammates, they can all delete submissions — there's no view-only role out of the box (see "Admin sign-in" above for how to add one via custom claims).
- If you ever want an *extra* layer beyond Firebase Auth (e.g., restricting who can even attempt to sign in by network location), that's a hosting-layer concern — Vercel's native password protection requires a Pro plan plus a $150/month add-on, so an IP allowlist or reverse proxy is the more realistic free option.

## Data model

Collection: **`submissions`** (Firestore), or the `frostline_submissions_v1` key in `localStorage` in demo mode:

| Field                    | Type                                       | Notes                                                |
|--------------------------|---------------------------------------------|-------------------------------------------------------|
| `teamName`               | string                                       | required                                               |
| `whatToDesign`           | array of `{ item, qty }`                     | required — checklist selections with quantities, min 15 pcs each |
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
