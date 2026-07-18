# CLAUDE.md — Deepanjali Project Context

## What this is
"Deepanjali" is a private family heirloom register for the house **P1 Deepanjali**. Family members (including elderly parents) photograph antique furniture and artifacts on their iPhones; each object gets a catalog number (DPJ-001 format), a timestamp, a room assignment, and an AI-written description. Built as a single-file PWA that the family adds to their iPhone home screens. The owner (Risheet) is technical but wants a zero-command-line maintenance story for the family; Claude Code is now the exception where terminal work is allowed.

## Architecture
- **Frontend:** `index.html` — the entire app (HTML + CSS + JS module, no build step). Hosted on **GitHub Pages** from this repo (`https://p1deepanjali.github.io/deepanjali/`). Also in repo: `manifest.json`, `icon-180.png`, `icon-512.png`.
- **Backend data:** **Firebase** project `deepanjali` — Auth (email/password accounts created manually in console), Firestore (collections: `items`, `rooms`, `users`, `meta/counter` for the DPJ number sequence), Storage (`photos/` folder, client-compressed JPEGs ~1600px q0.82). Security rules: any signed-in user has full read/write.
- **AI endpoint:** a **Val.town** HTTP val (source mirrored in this repo as `valtown-deepanjali-ai.ts`). Flow: app sends `{idToken, photoURL}` → val verifies the Firebase ID token via identitytoolkit lookup → checks the URL host is Firebase Storage → calls Anthropic API (`claude-sonnet-4-6`) with the image as a URL source → returns `{title, description, period, materials}` JSON → app writes it onto the item doc. Env vars on Val.town: `ANTHROPIC_API_KEY`, `FIREBASE_WEB_API_KEY`.
- **Config:** the CONFIG block at the top of the `<script type="module">` in `index.html` holds the Firebase web config and `AI_ENDPOINT` (the val's URL). These are real values in the deployed repo. **Never overwrite them with placeholders; always preserve the existing CONFIG when editing.**

## Deploy flow
- App: edit `index.html` → commit → push to `main` → GitHub Pages serves it (~1 min). iOS caches home-screen PWAs aggressively; after significant changes, users may need Settings → Safari → Advanced → Website Data → delete site, then re-add to home screen.
- Val: paste updated `valtown-deepanjali-ai.ts` into the val at val.town (or use the Val.town API if credentials are provided). Keep the repo copy in sync with what's deployed.

## System status: WORKING (as of July 2026)
All core flows are verified live: sign-in, single capture, room tagging, Claude descriptions, editing, activity stats. One historical incident worth knowing: descriptions once failed silently because a re-pasted `index.html` shipped with the placeholder `PASTE_YOUR_VALTOWN_URL` still in `AI_ENDPOINT` — requests died client-side and never reached Val.town. This is why the CONFIG rule below is non-negotiable. (An earlier iteration also failed with Val.town "write EPIPE" errors from POSTing ~500KB base64 images; the fix, already in place, is sending `photoURL` and letting the Anthropic API fetch the image itself. Never revert to base64 image payloads through the val.)

Debug playbook if descriptions ever fail again: failed items store `aiStatus:"failed"` plus an `aiError` string shown in red on the item detail screen — read that first. A GET to the val URL should return `{"error":"POST only"}` and create a Val.town log entry; no log entry on app requests means the request is dying client-side (check `AI_ENDPOINT`, then PWA cache).

## Product decisions already made (do not regress)
- **No geolocation.** The location feature was deliberately removed; never reintroduce permission prompts for it.
- **Design system:** "Gallery" direction. Auto light/dark via `prefers-color-scheme` (no manual toggle). SF Pro system font stack. Near-monochrome chrome; single accent `--flame` (#FF9500 light / #FF9F0A dark) reserved for the flame mark, capture button, and small accents. Identity is a flame-over-diya-arc SVG mark (Deepanjali = "offering of lamps"). Photo-grid register (2-col 4/5 tiles), frosted-glass tab bar and collapsing header, shared-element photo transitions, sheet modals with iOS push-back (background scales to .93). Respect `prefers-reduced-motion`.
- **Elderly-first UX:** capture flow must stay at "tap camera → take photo → tap room, done." No required typing anywhere in the add flow.
- **Rooms** are Firestore docs (label + emoji + order), editable in-app; 15 seeded defaults. Items reference `roomId` so renames propagate.
- **Bulk add:** multi-select then rapid one-tap-per-photo room tagging; uploads concurrency 3, AI calls concurrency 2.
- **Export:** client-side ZIP (JSZip from cdnjs) with photos named `DPJ-### - Title.jpg` + `register.csv` + `register.json`. Requires one-time CORS config on the Storage bucket (gsutil `cors set` allowing GET from the Pages origin) for photo fetches to work.

## Conventions
- Single-file app; no build tooling, no frameworks. Firebase modular SDK v10 via gstatic ESM imports.
- Keep everything working on iOS Safari first; test viewport is iPhone.
- Costs matter modestly: Firebase Blaze with ~$0 usage, AI ~1¢/photo. Don't add paid services without asking.
- Ask before schema changes to Firestore; existing family data must survive all migrations.

## Likely next tasks (owner's roadmap)
1. Generate small thumbnails on upload so the home grid doesn't load full-size images (matters as the register passes ~70 items; a bulk import of ~70 photos is planned).
2. Backfill/verify the one-time Storage CORS config so the ZIP export includes photos (see Export above).
3. Possible future: multi-photo angles per item, one-page printable "instruction card" for elderly parents, scheduled export reminders.
