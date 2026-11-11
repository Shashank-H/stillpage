# Stillpage — MVP Product Requirements

**Status:** MVP specification  
**Product name:** Stillpage (working name; app-store/trademark clearance pending)  
**Platforms:** Android and iOS  
**Business model:** Free MVP; optional paid sync may be evaluated later  
**Architecture:** Local-first, on-device extraction and rendering, no required backend

## 1. Vision

A beautiful, distraction-free place to save publicly accessible articles, social posts, and videos from other apps and return to them in a consistent reading experience. Users should not need to open distracting social feeds just to consume something they saved.

## 2. Product principles

- Save instantly, even offline or when extraction is unavailable.
- No account, payment, or cloud connection required for core functionality.
- Fetch and extract on-device; do not operate a centralized scraping service.
- Prefer direct HTTP fetching and HTML parsing; use a hidden WebView only when necessary and while the app is active.
- Display successfully extracted content in native app UI, not a visible browser.
- Always provide an immediately accessible **Open original** action that launches the device's external browser.
- Do not promise universal extraction, background completion, or paywall access.

## 3. MVP scope

### 3.1 Capture links (P0)

- Receive URLs from Android and iOS share sheets, and accept manual paste.
- Persist the URL and creation timestamp to local SQLite **before** attempting network access or extraction.
- Return control to the originating app as quickly as possible.
- Support offline saves, duplicates handling, deletion, and basic metadata when available.
- If a share extension or background process is terminated, the saved URL must remain available.

### 3.2 Library (P0)

- Clean, visually attractive list of saved content cards with title, source, thumbnail when available, and saved date.
- Search saved links, sort newest first, mark read/unread, and delete.
- Show extraction states without blocking browsing.
- Do not require user signup.

### 3.3 Extraction pipeline (P0)

1. Immediately save the bookmark with `pending` extraction status.
2. When execution time permits, fetch public HTML directly from the device and parse metadata and meaningful content.
3. Validate extraction based on content type and completeness, not just text length.
4. If HTML extraction is inadequate **and the app is foregrounded**, optionally render the page in a hidden WebView and run an injected extractor.
5. Normalize and sanitize the extracted content and store it **locally** for quick reopening.
6. On opening a bookmark without usable saved content, retry extraction. If still unavailable, show a helpful failure state and the original link.
7. Keep the original external-link action available during loading, success, partial extraction, and failure.

Extraction adapters should be modular: generic article/blog, X, Instagram, LinkedIn, Reddit, YouTube. Public social content is best effort; supported embeds or external original links are acceptable where direct extraction is unavailable. No private-account session handling.

### 3.4 Background processing and OS behavior (P0)

**Required user experience:** Sharing a link must feel instant. Extraction should continue opportunistically when the operating system permits, but users must never be told it is guaranteed to complete while they are in another app.

**Android:**
- Schedule eligible direct-HTTP extraction jobs using native WorkManager integration where practical.
- Jobs may run after the user returns to X or another app, subject to system scheduling, connectivity, battery optimization, and execution limits.
- Do not rely on a hidden WebView running inside a background worker.
- Resume pending or interrupted jobs when the main app becomes active.

**iOS:**
- Save the URL promptly in the share extension, using an app-group-compatible persistence strategy where necessary.
- Perform only bounded work while the extension is active; do not rely on continued execution after dismissal.
- Background task APIs or background URLSession may be evaluated for supported transfers, but they do not guarantee immediate arbitrary HTML extraction.
- Defer unfinished parsing and WebView extraction until the main app is active.

**Both platforms:**
- Treat background work as best effort, not immediate or guaranteed.
- Persist job state and retry safely with bounded attempts and backoff.
- Do not hold up the share flow for extraction.
- Hidden WebView rendering is foreground-only.
- If extraction has not finished when a bookmark is opened, begin or resume it and show a loading state with **Open original** visible.

### 3.5 Reader and original link (P0)

- Render extracted text, headings, images, and supported media with consistent native UI.
- Include source attribution and the original URL.
- Offer **Open original** prominently at all times, including while extraction is in progress.
- **No visible in-app browser in MVP.** Opening the original always uses the device's external browser.
- If content is partial or unavailable, clearly label that state rather than implying completeness.

### 3.6 Themes and reading modes (P0)

- Themes: light, dark, sepia; follow system theme when selected.
- Reading preferences: font size, font family, line spacing, and content width; persist locally.
- Reading modes: **Standard** (formatted content with images/media) and **Focus** (text-first minimal UI).
- No Original WebView reading mode in MVP; use external browser action instead.

### 3.7 Public content and paywalls (P0)

- Attempt extraction from content ordinarily accessible through unauthenticated HTTP requests or public webpage rendering, even if the page includes subscription prompts.
- Do not bypass authentication, encryption, subscription APIs, or technical access restrictions.
- If a server returns only a preview, show only accessible material and link to the original.
- Source attribution and linking do not automatically grant rights to republish full copyrighted content; assess applicable content-use permissions before distributing the feature.

## 4. Suggested implementation

| Area | MVP approach |
|---|---|
| Mobile | React Native + Expo development builds / native integrations as required |
| Local persistence | SQLite |
| Share support | Native Android share intent and iOS share extension |
| Fast extraction | Native device HTTP request + HTML parser / article extractor |
| Complex-page fallback | Hidden `react-native-webview`, foreground only |
| Background extraction | Android WorkManager for eligible HTTP jobs; limited iOS share-extension work and foreground resume |
| Reader | Native React Native components with sanitized structured content |
| Backend | None |
| Authentication | None |

### Bookmark schema (conceptual)

```typescript
type ExtractionStatus =
  | 'pending'
  | 'extracting'
  | 'ready'
  | 'partial'
  | 'failed';

type Bookmark = {
  id: string;
  url: string;
  canonicalUrl?: string;
  source?: string;
  type: 'article' | 'post' | 'video' | 'unknown';
  title?: string;
  description?: string;
  thumbnail?: string;
  author?: string;
  savedAt: string;
  updatedAt: string;
  deletedAt?: string;
  isRead: boolean;
  extractionStatus: ExtractionStatus;
  lastExtractionAttemptAt?: string;
  extractionError?: string;
  // Extracted text and content blocks stored locally, separately.
};
```

## 5. Performance and reliability

- Save URL to local storage without waiting for extraction; target under one second after receiving it on reference devices.
- Show locally extracted content immediately on reopening when available.
- Limit fetch duration, WebView render time, retries, memory use, and redirects.
- A failed extraction must never delete or invalidate the bookmark.
- Allow browsing, searching, and managing saved URLs offline; fresh remote content requires connectivity.
- Keep extracted text locally until bookmark deletion for MVP; large media may remain remote.
- Avoid unsafe URL schemes, privileged WebView bridges, untrusted scripts in the native reader, and unsafe HTML rendering.

## 6. Acceptance criteria

1. User shares an X or news URL, immediately returns to the source app, and finds the bookmark saved in Stillpage later.
2. On Android, eligible HTTP extraction may finish in a scheduled background job; the app behaves correctly if it is delayed or never runs.
3. On iOS, dismissing the share extension does not lose the URL, and incomplete extraction resumes when Stillpage is opened.
4. Direct HTML extraction is attempted before a hidden WebView.
5. Hidden WebView is not presented to the user and is never required to keep running after the app backgrounds.
6. User can tap **Open original** at any stage; it opens the device's external browser.
7. Previously extracted content loads locally, without refetching on every open.
8. Themes and reading modes are available and persist across restarts.
9. Unsupported public pages degrade gracefully; no mandatory login or server required.
10. Test 20–30 public URLs across news, blogs, and supported social platforms; measure success, completeness, latency, and fallback frequency.

## 7. MVP delivery sequence

1. **Technical spike:** Share-sheet persistence, direct HTML extraction, hidden WebView foreground fallback, Android WorkManager, and iOS extension lifecycle on real devices.
2. **Core:** SQLite library, manual save, share flows, search, read status, and deletion.
3. **Reader:** Normalized extraction, local persistence, themes, Standard/Focus modes, external-link action.
4. **Platform adapters:** Best-effort public X, Instagram, LinkedIn, Reddit, and YouTube handling.
5. **QA/release:** Device testing, reliability, performance, accessibility, store readiness.

## 8. Future scope — not MVP

### Optional accounts and sync

- Guest mode remains fully functional without login.
- Signed-in users may sync **URLs and lightweight bookmark metadata only** across devices; extracted bodies, rendered HTML, and media stay on-device.
- Preserve local bookmarks when guests sign up; merge and deduplicate safely.
- Local SQLite remains the immediate data source; sync is optional.
- Server handles authentication and bookmark synchronization only, never scraping or rendering.

### Potential monetization

- Evaluate optional paid cross-device synchronization/backup after product validation.
- Keep unlimited local saving, themes, reading modes, and on-device extraction free.
- No pricing or payment provider is selected for MVP.

### Other future ideas

- Configurable original-link opening behavior (external browser vs. visible in-app browser), **not in MVP**.
- Optional cloud backup, advanced organization, or offline media caching, subject to future validation.

## 9. Explicit MVP exclusions

- Mandatory accounts, sync, subscriptions, ads, AI, recommendations, and social networking.
- Visible in-app browser or browser-preference setting.
- Guaranteed background execution, especially hidden WebView rendering while backgrounded.
- Server-side scraping, permanent server-side article storage, or private social-content extraction.
- Circumvention of technical access restrictions.

**Product promise:** Save instantly. Prepare content when the device allows. Read in a beautiful native interface—or open the original link at any time.
