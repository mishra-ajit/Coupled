# AGENTS.md — operating guide for Coupled

> This file is the contract for any AI agent (or human) working on **Coupled**.
> Read it first, every time. It defines *what we're building*, *why*, *how to change it safely*, and *what must never break*. When in doubt, optimise for the North Star in §1 and never violate the Invariants in §6.

Live: **https://getcoupled.netlify.app/** · Repo: **https://github.com/mishra-ajit/Coupled** (public)
Deep context: see `DOCUMENTATION.md`. Product research + roadmap thinking lives there too.

---

## 1. North Star

**Coupled is a private daily ritual for two partners — not a dashboard of two individuals.**

Every change should make the app feel more like *opening a shared, intimate space together* and less like *logging personal metrics*. If a feature doesn't help two people **know each other**, **turn toward each other**, **appreciate each other**, or **share something new**, it probably belongs in the background (Nest/Profile), not the emotional foreground (Home).

Guiding research (evidence-backed — honour these when designing):
- **Love Maps + escalating self-disclosure** (Gottman; Aron's "36 questions") → depth comes from structured, reciprocal, deepening questions.
- **Turning toward bids / A.R.E.** (Gottman; Sue Johnson EFT) → responsiveness is the single highest-value behaviour; surface each other's signals (esp. mood) and make responding one tap.
- **Gratitude as a "booster shot"** (Algoe; Gable's capitalization) → expressing appreciation and celebrating each other's wins raises next-day satisfaction for *both*.
- **Novelty sustains passion** (Sternberg; Aron self-expansion) → nudge toward new shared experiences, not just logistics.
- **Rituals of connection** (Gottman) → recurring shared moments (e.g. the Friday Vault) are the spine of the app.

The winning consumer pattern is **one low-friction shared thing per day**. Bias toward that.

---

## 2. What this app is (architecture in one breath)

A single-file, offline-first PWA for exactly two people. The **entire app is `index.html`** (HTML + Tailwind-CDN CSS + vanilla JS). Two partners sign in with Google, one invites the other by email, and they share one **end-to-end-encrypted** space in real time.

```
Browser (index.html)
  ├─ UI: Tailwind CDN + custom theme tokens
  ├─ Logic: vanilla JS, no framework, no build step
  ├─ Web Crypto: AES-GCM + PBKDF2 (E2E, client-side)
  └─ Service worker (Blob URL): offline caching
        │
   Firebase Auth (Google)   Cloud Firestore (stores ONLY ciphertext)
        │
   Firebase AI Logic (Gemini gemini-2.5-flash) → weekly insight
        │
   Netlify (static host) ← auto-deploys from GitHub main
```

Stack: Firebase compat SDK 10.12.5 (Auth + Firestore), Firebase AI Logic modular SDK 11.10.0 (module app named `"ai"`), Chart.js 4.4.1, Tailwind via CDN. No bundler, no npm runtime deps in production.

---

## 3. Repository layout

| File | Role |
|---|---|
| `index.html` | The whole app. Source of truth. |
| `manifest.json` | PWA manifest (name "Coupled", cream theme, PNG icons). |
| `firestore.rules` | Per-space member isolation + invite join; wildcard `match /{sub}/{docId}` covers all subcollections. |
| `FIREBASE_SETUP.md` | One-time Firebase setup guide. |
| `README.md` | Public overview. |
| `DOCUMENTATION.md` | Full technical + product doc. |
| `icon-192.png` / `icon-512.png` / `apple-touch-icon.png` | House + heart line-art on cream. |
| `Code.gs`, `GOOGLE_LOGIN_SETUP.md` | **Deprecated** (Google Apps Script era). Do not extend. |

---

## 4. Data model (Firestore)

Every content doc is `{ uid, enc }` where `enc` = base64 AES-GCM ciphertext of `JSON.stringify(obj)`.

```
spaces/{spaceId}
  members[] · memberProfiles{} · invitedEmails[] · ownerUid
  encSalt · verifier · habitsSeeded
  └─ subcollections (each doc = { uid, enc }):
       logs (h-<uid>-<date>-<label>) · items · notes
       weights (w-<uid>-<date>) · moods (m-<uid>-<date>)
       ai · habits · files ({name,type,size,n,at,by}) · fileChunks
```

**Files are chunked** (Firestore ~1 MB doc limit): file → base64 → slices of `DOC_CHUNK = 600000` chars → each encrypted → `fileChunks` doc id `<fileId>-<i>` = `{fileId, idx, uid, enc}`; metadata in `files/{fileId}`. Reading: query by `fileId`, sort by `idx`, decrypt, concat, rebuild Blob.

Core helpers you'll reuse: `docId()`, `putEnc(coll,id,obj,uid)`, `encStr`/`decStr`, `abToB64()`, `b64ToBytes()`, `fmtSize()`, `fileIcon()`, `escapeHtml()`.

**A powerful, reusable pattern:** the Friday Vault's *"seal now, reveal to both only when a condition is met"* mechanic. New depth features (Daily Question, etc.) should reuse this "reveal when both have answered" idea rather than reinventing it.

---

## 5. Coding conventions

- **Single file.** Keep everything in `index.html`. No new files unless it's a repo artifact (docs, rules, icons).
- **Vanilla JS**, small helpers, no framework. Match the existing style (short functions, `el(id)`, `render*()` functions per feature, Firestore `onSnapshot` listeners pushed to `unsub`).
- **Always encrypt user content** before writing (`putEnc` / `encStr`). Never write plaintext user content to Firestore.
- **Always `escapeHtml()`** any user-supplied string before injecting into `innerHTML`.
- **Theme tokens only** (don't hardcode colours): `appbg #f2ece0`, `card #fbf7ef`, `ink #2d3e46`, `sub #7a8a90`, `accent #00897b`, `accentdark #00695c`, `line #e4dccb`, `partnerA #00897b`, `partnerB #ff7043`. Fonts: Quicksand (body), Pacifico (`font-logo`).
- **Mobile-first**, max-width ~`md`, bottom tab bar (Home · Nest · Plants · Vault). Rounded-2xl cards, gentle shadows, warm tone. Emoji as lightweight iconography.
- **Copy voice:** warm, present-tense, never guilt-inducing or failure-implying. Quotes/prompts are original (self-authored), never scraped.
- **Offline-friendly:** assume flaky network; rely on Firestore offline cache and pending-write state.

---

## 6. Invariants — DO NOT BREAK

1. **End-to-end encryption stays intact.** All user content encrypted client-side; server sees only ciphertext. Never add a code path that stores plaintext or ships the key off-device.
2. **Never lose data.** History/metrics are permanent. Don't add destructive migrations or overwrite historical docs. Habit logging is capped to once/day but must never delete past logs.
3. **Keep `docId()`** — used by `putEnc` for documents, weights, moods, habits, and habit logs. Removing it breaks writes.
4. **Zero server cost.** Stay on free tiers (Netlify + Firebase Spark). Do **not** introduce anything requiring the paid Blaze plan (e.g. Firebase Storage — that's why files use chunked Firestore). No new paid services.
5. **Two-person model.** A space is for a couple. Don't add flows that assume >2 members or public sharing.
6. **The Firebase web config is public by design** and safe in the repo. Security = Firestore rules + E2E, not secrecy. Don't "hide" the config or treat it as a leak.
7. **Host and sandbox copies of `index.html` must stay identical** (see §7).

---

## 7. Change workflow (follow exactly)

1. **Edit** `index.html` (host copy = source of truth).
2. **Mirror** to the sandbox copy so it can be verified/uploaded. Both must be byte-identical (`diff -q`).
   - Host (file tools): project folder `index.html`.
   - Sandbox (bash): `outputs/site/index.html`.
3. **Verify before deploy** (never skip):
   - Extract the largest inline **non-module** `<script>` and run `node --check` on it.
   - `grep` to confirm intended additions are present and removed code is gone.
   - For data/crypto changes, prove round-trips in Node (e.g. chunk → encrypt → decrypt → byte-identical).
4. **Deploy** (Netlify auto-deploys from `main`):
   - Upload the verified `index.html` at `https://github.com/mishra-ajit/Coupled/upload/main`.
   - Find the file input ("Choose your files") and upload; **do not** click file inputs blindly.
   - Commit via the submit button using `button.form.requestSubmit(button)` — **plain `.click()` on GitHub's commit button does not submit** (known gotcha).
   - Confirm the deploy against `https://raw.githubusercontent.com/mishra-ajit/Coupled/main/index.html?cb=<ts>`; hard-refresh the live site.
5. **Target inputs precisely** during browser automation. A past run renamed the repo by targeting the wrong field via a "last visible input" heuristic — never guess; match by exact label/selector.

Historical gotchas already solved (don't reintroduce): base64 stack-overflow on large files (fixed by chunking `toB64` to 0x8000 slices + `DOC_CHUNK=600000`); GitHub commit submission (use `requestSubmit`).

---

## 8. Product direction & roadmap (build order)

Reframe **Home** from a metrics dashboard into **"Us, today"**: a single rotating **hero action** that is always a *shared* thing, then a light pulse of shared life; move logistics (Documents) off Home into Nest/Profile; collapse heavy widgets (mood chart, AI insight) behind a tap.

Prioritised features (impact per effort; reuse the Vault reveal + mood mechanics):

**First:** Daily Question (escalating, reveal-when-both-answered) · Appreciation "one thing" (notifies partner) · "On this day" resurfacing.
**Next:** Mood attunement nudge (turn toward a partner's low day) · Share-a-win / celebrate · Novelty jar / date-idea draw.
**Later:** Weekly "how are *we* doing" check-in · Repair mode after conflict · time-capsule Vault (unlock on anniversary).

For any new feature, ask: *Which research lever (§1) does this serve, and does it belong on Home (shared/foreground) or in a tab (individual/utility)?*

---

## 9. Definition of done

- Serves a §1 lever (or clearly improves reliability/usability of something that does).
- Honours every §6 invariant.
- Encrypted, escaped, themed, mobile-first, warm copy.
- Verified per §7 (node check + grep + relevant round-trip), host/sandbox identical, deployed, and confirmed live.
- Docs updated (`DOCUMENTATION.md` change log; this file if direction/workflow changed).
