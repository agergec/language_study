# French Study Web App — Technical Design

This document provides a low‑level technical design for the single‑page French listening/reading study app implemented in `index.html` (mirrored from `beta/index.html`). It describes behaviors, data structures, state flows, UI composition, persistence, and integrations.

## Goals and Scope
- Provide an offline‑friendly, installation‑light web app for studying words/phrases in several study modes.
- Work well on mobile and desktop (PWA‑like), with strong keyboard/touch support.
- Support voice playback via:
  - System speech synthesis (Web Speech API)
  - Optional OpenAI TTS (HTTP API)
  - Optional ElevenLabs TTS (HTTP API)
- Load user word lists from local `.txt` files and persist them for subsequent visits (IndexedDB).

---

## Architecture Overview
- Single static document: `index.html` contains HTML, CSS, and JavaScript.
- No external build system or framework; runtime constructs PWA manifest and registers a light service worker when possible.
- Data/state kept in memory (plain JS variables), with persistence to browser storage (localStorage + IndexedDB).
- UI composed of:
  - A responsive CSS grid container with: header, display area, vertical controls, sidebar history, footer status bar.
  - Multiple modals (welcome/help/settings/wordlist/history) + a special “menu” modal implementing a right‑drawer with push subpanels.
  - Footer status bar with inline, anchored quick menus for mode/voice/files, and a compact inline confirmation for shuffle.

---

## Layout and Styling
- Grid layout with named areas: `header`, `display`, `controls`, `history`, `footer`.
- Desktop vs mobile:
  - Desktop: visible left “Answers” sidebar, central display, right controls.
  - Mobile: vertical stacking; answers panel collapsed; controls below display.
- iOS/standalone safe‑area handling via `env(safe-area-inset-*)` and `is-ios` / `is-standalone` classes.
- Modals use a blurred backdrop and `display:flex` toggled by `.visible`.
- Menu modal (`#menuModal`) is a full viewport drawer with:
  - Main `section.panel` for grouped settings mockup (iOS‑style cards)
  - Subpanels (`.panel.subpanel`) as fixed, full‑screen stacks sliding over the panel (push navigation)
- Cross‑browser fix: avoid `100vw/100vh` in the drawer to eliminate overflow in Firefox/Chromium; use `width/height: 100%` within the modal.

---

## State and Data Structures
- Top‑level session variables (illustrative):
  - `words: string[]` — flattened active words for current session
  - `originalWordOrder: string[]` — snapshot for unshuffling
  - `currentIndex: number`, `currentWord: string`, `isSessionComplete: boolean`
  - `readSayStage: 0|1|2`, `listenRepeatStage: 0|1|2` — staged flows for specific modes
  - `shuffleEnabled: boolean`
  - `curLang: 'en'|'fr'` — current UI language
  - `currentVoice: string` — value from `#voiceSelect` (e.g., `system`, `openai:echo`, `eleven:VOICE_ID`, or system voice name)
  - `availableVoices: SpeechSynthesisVoice[]` — from `speechSynthesis.getVoices()`
- Word list registry (in‑memory):
  - `wordLists: Record<string, { words: string[]; enabled: boolean }>` keyed by filename.
- Article/prefix pools (for “le/la”, “un/une” modes):
  - `prefixWordsIndef`, `prefixWordsDef`: arrays of `{ full, prefix, base }` derived by regex from active `words`.

---

## Persistence
- Preferences via localStorage:
  - `selectedVoice`, `studyMode`, minor flags (e.g., `voiceDebug`), API keys (`eleven_api_key`, `openai_api_key`)
- Word lists via IndexedDB:
  - DB: `studyApp`, store: `wordlists`, keyPath: `filename`
  - Record: `{ filename, words: string[], enabled: boolean, updated: number }`
  - Consent flow: on first successful word load per session, a compact confirm asks to “Save word lists on this device”. If accepted, subsequent changes auto‑persist. Consent tracked by `persistConsent` in localStorage, session gating through `sessionStorage.persistPrompted`.
  - Restore on startup: loads all records and rebuilds in‑memory state; hides the welcome modal if data present.
  - Reset action clears SW caches, LS/SS, and IDB wordlists.
- Short‑term audio cache via localStorage:
  - Key: `tts_last_audio`, storing last clip as `{ engine, voice, text, dataUrl, mime, ts }`
  - Used to replay repeated words without re‑fetching (OpenAI/ElevenLabs).
  - Explicitly cleared on app start, study mode change, word list changes, shuffle changes, voice changes, and API key changes.

---

## File Loading and Parsing
- Sources:
  - Welcome modal button (file input)
  - Menu → Files subpanel → “Add Word List…”
  - Global drag‑and‑drop of `.txt`
- Format: each non‑empty line is a word or phrase; trim, filter empty lines.
- After load: update `wordLists`, rebuild flattened `words` array and prefix pools, reset study session, update indicators, optionally persist to IDB.
- Files subpanel UI (iOS‑style):
  - Shows each file with word count, optional enable toggle (only if >1 file), and a leading “×” delete button.
  - Delete removes from memory and IDB and refreshes the session state.

---

## Study Modes and Flows
- Modes (selectable in Menu, quick‑switch in footer):
  - `listenRead`: display + speak once
  - `readThenSay`: display (student reads), then speak on next action
  - `listenThenRepeat`: speak (student repeats), then on advance reveal and speak again
  - `articlesIndef` (`un/une`) and `articlesDef` (`le/la`): show base (e.g., “... ami”), student selects prefix or skips; tracks separate pools
- Flow control:
  - Stage variables (`readSayStage`, `listenRepeatStage`) ensure multi‑step sequences are respected.
  - History tracking records revealed words or unanswered items as appropriate.
  - Progress bar reports by mode: per `words.length` or per prefix pool length.

---

## Audio Subsystem
- System TTS: Web Speech API (SpeechSynthesisUtterance)
  - Voice selection: `system` means “best available French voice”; otherwise, pick named voice.
  - Platform nuance: Chromium vs Safari/Firefox handling for voice resolution.
- OpenAI TTS: `POST /v1/audio/speech` (model: `gpt-4o-mini-tts`, format MP3)
  - Requires `openai_api_key` in localStorage.
  - Fetches ArrayBuffer → Blob(MP3) → plays via HTMLAudioElement.
  - Error fallback to system TTS with a toast.
- ElevenLabs TTS: `POST /v1/text-to-speech/{voice}` (non‑streaming, MP3)
  - Requires `eleven_api_key` in localStorage or window.
  - Same blob playback and fallback behavior.
- Short‑term audio cache:
  - On successful fetch, the MP3 blob is converted to a data URL and cached as last clip keyed by `{ engine, voice, text }`.
  - Before fetching, engines check the cache; Repeat uses cache aggressively to avoid re‑fetch.

---

## Modals and Menu System
- Generic modal pattern: container with `.modal` + `.visible`; `.modal-content` has `role=dialog` and `aria-modal=true`; the script focuses the content on open.
- Menu modal specifics:
  - Main panel: grouped settings (language, shuffle, study mode, voice, integrations, data & system)
  - Subpanels: Language, ElevenLabs, OpenAI, Files
  - Push navigation: `.subpanel[aria-hidden=false]` slides over; main header is hidden when a subpanel is open to prevent overlap.
  - Floating close overlay aligns with hamburger on open (mobile affordance).

---

## Footer Status Bar and Quick Menus
- Elements: `bottomShuffle`, `bottomFileStatus`, `bottomVoice`, `bottomMode`, `bottomVersion`.
- Quick menus:
  - Mode: anchored listbox mirroring `#studyModeSelect`. Disables unavailable modes (e.g., no article words yet).
  - Voice: anchored list of active voices (system + integrations). Selecting updates `#voiceSelect`.
  - Files: anchored list of loaded files with counts; includes “Add Word List…” action.
- Shuffle toggle: compact inline confirmation near the icon to flip shuffle without opening settings.

---

## Keyboard and Touch
- Shortcuts: `Space` → Next, `R` → Repeat, `S` → Show, `Esc` → close modals; `Alt+M` toggles the menu.
- Touch: all primary interactions are click/touch friendly; quick menus handle `touchend` with passive control.

---

## Internationalization (i18n)
- Inline `i18n` map for `en` and `fr`:
  - Includes UI strings, prompts, toasts.
  - `setLanguage(lang)` swaps active content, labels, and titles.

---

## PWA and Platform Enhancements
- iOS meta tags and icons; canvas‑generated PNG icons for manifest at runtime.
- Service Worker (registered on https/localhost): minimal cache‑first for core files.
- Standalone and iOS classes used to adjust safe‑area padding and scrolling.

---

## Accessibility
- `role="dialog"` and `aria-modal` for modals; `aria-hidden` state changes for panels/subpanels.
- Focus management on modal open; `aria-live` for important dynamic text (e.g., main display).
- Labels/titles on actionable icons and buttons.

---

## Error Handling and UX Guards
- TTS errors: show a toast and fall back to system TTS.
- Toasts suppressed when the menu is open to avoid clutter.
- Mini inline confirmation used for sensitive quick actions (shuffle) instead of big dialogs.

---

## Extensibility
- Adding a study mode:
  1. Add option to `#studyModeSelect` and i18n labels.
  2. Update `getSelectedStudyMode`, `updateStudyModeIndicator`.
  3. Extend `nextWord()` and related staged flows.
- Adding a new TTS engine:
  1. Add options to `#voiceSelect` builder.
  2. Add engine dispatch in `speakWord`.
  3. Implement `speakWith<Engine>` with the same blob→audio pattern + short‑term cache hooks.
- Storage model allows per‑file metadata; extend IDB schema in an `onupgradeneeded` bump as needed.

---

## Known Limitations
- Audio cache: stores only the last clip; encoded as data URL in localStorage (size‑limited). Intended as a short‑term convenience, not long‑term storage.
- OpenAI/ElevenLabs require network and valid keys; CORS/network issues fall back to system TTS.
- iOS/WebKit autoplay policies can affect playback; code speaks in direct user gesture contexts where practical.
- localStorage/IndexedDB can be cleared by the browser at any time (private mode, storage pressure, user action).

---

## Build, Test, and Deploy
- Static hosting: serve `index.html` with assets; no build step required.
- Local testing: open over `http://localhost` or `https://` to enable SW; test on desktop and mobile browsers.
- Release process: copy `beta/index.html` → `index.html` (already in place) and publish.

### CI/CD with GitHub Actions (GitHub Pages)

This repo includes CI + deployment workflows under `.github/workflows/`:

- `ci.yml` — runs on pushes and pull requests to `main`. Uses Lychee to check links in HTML and README.
- `pages.yml` — builds and deploys to GitHub Pages on push to `main` (and via manual dispatch).
- `beta-deploy.yml` — builds and deploys the beta variant (from `beta/`) to a separate repository on push to the `beta` branch (publishes to that repo’s `gh-pages` branch).

Steps to enable:
- In your repository: Settings → Pages → “Build and deployment” → Source: GitHub Actions.
- Optional: Protect `main` with required status checks and require the “CI” workflow to pass before merging.

What it deploys:
- Copies `index.html` and, if present, the `beta/` folder into a `dist/` directory and publishes that as the site root. You can add other asset folders named `assets/`, `static/`, or `public/` and they’ll be included automatically.

Customizing tests:
- You can add more checks (HTML validation, ESLint, Prettier, etc.). Add a `package.json` with the tools and extend `ci.yml` to install and run them.
- To tighten link checks, edit the `args` in `ci.yml` (e.g., remove `--accept 403,429`, add `--exclude` patterns, or restrict to local links only).

### HTML Linting

CI runs `HTMLHint` against `index.html` to catch malformed markup. Rules are in `.htmlhintrc`. To lint all HTML (including `beta/`) once it’s stable, change the CI step to:

```
npx --yes htmlhint --config .htmlhintrc "**/*.html"
```

### Option B — Separate Beta Repository

Use a second repository for your beta site, published independently of production.

1) Create a new repo (example): `yourname/language_study_beta`
   - In the beta repo: Settings → Pages → “Build and deployment” → Source: Deploy from a branch → Branch: `gh-pages` / folder: `/ (root)`.

2) In THIS repo, add two secrets (Settings → Secrets and variables → Actions → New repository secret):
   - `BETA_REPOSITORY`: set to `yourname/language_study_beta` (owner/repo)
   - `BETA_PAGES_TOKEN`: a Personal Access Token (classic) with `repo` scope from your account that has push access to the beta repo

3) Workflow behavior (`.github/workflows/beta-deploy.yml`):
   - Triggers on push to the `beta` branch (and manual dispatch)
   - Builds a `dist/` that uses `beta/index.html` as the site root
   - Pushes `dist/` to the external repo’s `gh-pages` branch using `peaceiris/actions-gh-pages`

4) Usage:
   - Branching model: develop beta changes on the `beta` branch here; pushing updates the beta site.
   - Production: merge to `main`; Pages workflow deploys the production site from `index.html` here.

Notes:
- This repo is configured to publish beta to the external repo’s `gh-pages` branch by default. Ensure the beta repo’s Pages source is set to `gh-pages` (as above).
- Keep your PAT safe; you can rotate it anytime and update the secret.


---

## File Map (key runtime components)
- `index.html`
  - CSS: responsive layout, modals, iOS grouped settings styles, quick menu styles, mini‑confirm styles
  - JS: word list management, IndexedDB persistence, TTS engines + cache, study modes, UI updates, quick menus, modals, PWA manifest/SW, i18n
- `beta/index.html`: staging iteration before release (source of truth during development)

---

## Security and Privacy Notes
- API keys are stored locally on device (localStorage). Users can remove them via the integrations pages.
- No backend; all processing and storage occur in the browser.
- Word lists are user‑provided and stored locally in IndexedDB upon consent.

---

## Future Improvements
- Multi‑entry audio cache with LRU and size accounting.
- Download/export and import of word list bundles.
- Additional languages and TTS voices.
- More granular accessibility testing and focus trapping within modals.
- Optional compression for large word lists before saving to IDB.
