# StudyBuddy 3D

**Upload. Understand. Practice. Remember.**

An AI-powered study assistant that runs entirely in the browser. Upload a text-based PDF, ask questions and get answers that cite the page they came from, take a 5-question exam built from your own notes, and revise what you missed with 3D flip flashcards on a spaced-repetition schedule.

Plain HTML, CSS and JavaScript. No build step, no framework, no backend. It is a set of static files, so it hosts on GitHub Pages as-is.

> **Read this first — three things that surprise people**
> 1. **The login is a demo.** There are no accounts and no server. It checks the *format* of the email and password and nothing else. Nothing is stored or sent. Use "Continue as guest" if you prefer.
> 2. **AI features need an Anthropic API key.** A static GitHub Pages site has no server to hide a key on, so you paste your own key into **Settings**. It lives in memory only and is gone when you close or reload the tab. See [API key and security](#api-key-and-security).
> 3. **Saved data needs `window.storage`.** Per the project spec the app does not use `localStorage` or `sessionStorage`. On an ordinary GitHub Pages site in an ordinary browser `window.storage` does not exist, so documents, exam history and flashcards last only until the tab is closed or reloaded. The app tells you this on screen. See [Storage](#storage).

---

## Files

```
index.html   page structure, icon sprite, accessible markup
style.css    design system, glassmorphism, 3D effects, responsive layout, reduced motion
script.js    all logic: PDF extraction, chunking, BM25, chat, exam, flashcards, 3D scene
README.md    this file
```

External resources, all loaded from CDNs with `defer`: Three.js r128, PDF.js 3.11.174 (cdnjs) and the Inter and Space Grotesk fonts (Google Fonts). If a CDN is blocked the app degrades visibly (see [Fallbacks](#fallbacks)).

## Features

| Area | What it does |
|---|---|
| **3D login** | Glass sign-in card with mouse tilt and a conic-gradient glow, 3D buttons, a Three.js neural-sphere scene behind it, and a card-flip transition into the app. Inline validation, loading state, show/hide password, guest entry. |
| **Dashboard** | Upload area, recent documents, activity counters (counted from this app only), weak topics and upcoming flashcard reviews. The sidebar becomes a fixed bottom bar on phones. |
| **PDF upload** | PDF.js extraction **page by page**, so every passage knows which pages it came from. Shows progress and clear errors for corrupt, empty, oversized or scanned PDFs. |
| **Chunking** | ~500-word passages with an 80-word overlap (both adjustable in Settings). Page markers are kept inside the text. |
| **Retrieval** | BM25 keyword ranking, top 4 passages. Tokenizer is Unicode-aware (Devanagari works) with English, Hinglish and Hindi stop-words and a light stemmer. |
| **Refusal guard** | If the best passage scores below the threshold, the app answers *"I couldn't find enough evidence in your uploaded notes to answer confidently."* and **makes no model call at all**. |
| **Grounded chat** | Claude answers only from the retrieved passages, in the language of the question (English, Hindi or Hinglish), with `[Page N]` citations. Citations are checked against the retrieved pages; a citation that does not match is flagged as unverified. Sources are shown under each answer. |
| **Exam mode** | Five multiple-choice questions, each built from a random passage. Every question is validated before you see it (see below). Answer by click, arrow keys or keys 1–4; confirm before submitting; per-question review with the explanation and a quote from your notes. |
| **Weak topics** | Wrong answers feed topic statistics. A topic is labelled **weak** only after at least 2 attempts with at least 50% missed; with one attempt it reads "Not enough data yet". Skipped questions score 0 and are not counted against a topic. |
| **Flashcards** | Each wrongly answered question becomes a card. Click, tap, Enter or Space flips it in 3D. Filter by Due, Upcoming, Known, Needs practice or topic. |
| **Spaced repetition** | First review 1 day after creation. "I knew it" moves a card to 3 days, then 7 days, then marks it mastered. "Needs practice" sends it back to 1 day. Cards can only be rated when due; otherwise they are preview-only. |
| **Settings** | API key (memory only), optional proxy endpoint, chunk size, overlap, refusal threshold, re-index, reduce motion, high contrast, erase all saved data. |
| **Accessibility** | Skip link, visible focus rings, ARIA roles for alerts/status/radio groups, keyboard-operable flashcards and exam, a reduce-motion switch (an OS "reduce motion" setting forces it on), a high-contrast switch. |

### How exam questions are kept honest

The model is asked for strict JSON and its answer is **rejected** unless:

- the JSON parses (code fences are stripped);
- there are exactly 4 distinct options and `correctIndex` is an integer from 0 to 3;
- the cited `page` is one of the pages the passage actually came from;
- the quoted `evidence` really appears in the passage;
- the question is not a near-duplicate (Jaccard similarity ≥ 0.7) of one already accepted.

Rejected responses are skipped and another passage is tried (at most 12 attempts). If fewer than 5 valid questions result, the app **says so** and offers *Try again* or, with at least 3, *Use these N questions*. It never invents a question to fill the gap. Option order is shuffled in the browser, not by the model.

## Run it locally

You need a static file server (opening `index.html` directly with `file://` works poorly with CDN scripts and workers).

```bash
cd studybuddy-3d
python3 -m http.server 8080
# open http://localhost:8080
```

Any static server works (`npx serve`, VS Code Live Server, and so on). To use the AI features locally, open **Settings**, paste your Anthropic API key and click **Use key for this session**.

## Deploy to GitHub Pages

1. Create an empty repository on GitHub (for example `studybuddy-3d`). Do not add a README there; this folder already has one.
2. In this folder run:

   ```bash
   git init
   git add .
   git commit -m "feat: initial StudyBuddy 3D"
   git branch -M main
   git remote add origin https://github.com/<your-username>/studybuddy-3d.git
   git push -u origin main
   ```

3. On GitHub open **Settings → Pages**. Under **Build and deployment** choose **Source: Deploy from a branch**, then **Branch: `main`** and **Folder: `/ (root)`**, and click **Save**.
4. Wait a minute or two. The page then shows your address: `https://<your-username>.github.io/studybuddy-3d/`.
5. Check it worked:
   - the login screen loads and the 3D scene appears (or the CSS fallback with a notice);
   - "Continue as guest" opens the dashboard;
   - uploading a text-based PDF shows "Ready · N pages · M chunks";
   - a question whose answer is not in the PDF gets the refusal message (this works without an API key);
   - the browser console (F12) shows no red errors.

All paths are relative and `index.html` is at the repository root, so no extra configuration is needed. If you rename files, keep `style.css` and `script.js` next to `index.html`.

## API key and security

**Nothing in this repository contains a key, and none should ever be committed.**

- The key you paste in **Settings** is held in a JavaScript variable only. It is never written to storage, never put in the URL, and is cleared when you sign out, close or reload the tab.
- It is sent **only** to `https://api.anthropic.com`, with the header `anthropic-dangerous-direct-browser-access: true` that Anthropic requires for browser-direct calls. If you configure a custom endpoint, the key is **not** sent to it.
- Error messages are scrubbed so a key never appears on screen.
- **Be honest about the risk:** anything running in your browser can in principle read a key typed into that browser (a malicious extension, for example). Use a key with a low spend limit, use it only on a device you trust, and revoke it afterwards. This is a convenience for local testing and personal use.

### For a public deployment: use a proxy

GitHub Pages is static, so it cannot keep a secret. If other people will use your site, put a small server-side proxy in front of the API and keep the key there (a Cloudflare Worker, a serverless function, and so on). Then in **Settings** set **Endpoint** to the proxy URL (`https://…`, or a path such as `/api/claude` if you host app and proxy together).

With an endpoint set, the app sends a `POST` with a JSON body `{ model, max_tokens, system, messages }` (the Anthropic Messages API shape) and expects an Anthropic-style response (`content[].text`). Your proxy must add the API key itself, apply rate limits, and allow your site's origin with CORS. If neither a key nor an endpoint is set, the app calls same-origin `/v1/messages`; on plain GitHub Pages that returns 404 and the app explains that no API is available.

The app uses model `claude-sonnet-4-6` with `max_tokens: 1000`. Both are constants at the top of `script.js`.

## Storage

The app persists documents, exam history, flashcards and settings through **`window.storage`** when the host page provides it, and keeps everything in memory otherwise. It deliberately does **not** use `localStorage` or `sessionStorage`.

- **With `window.storage`:** data survives reloads. Documents larger than about 4.5 MB of extracted text are not persisted and the app warns you.
- **Without it (the normal case on GitHub Pages):** data lasts until the tab is closed or reloaded. A warning toast, a banner on the Overview and a notice in Settings say so.
- The API key is never persisted in either mode.

If you want data to survive reloads on a plain website, the smallest change is a `localStorage`-backed `window.storage` shim. That goes against the current spec, so I have not added it. Ask first.

## PDF limitations

- **Text-based PDFs only.** OCR is not implemented. A scanned or image-only PDF is detected and rejected with an explanation.
- Limits: 25 MB, 400 pages, 400,000 words; at least 30 words of text are required.
- Page numbers are the PDF's **physical** page positions (page 1 is the first page of the file). They can differ from numbers printed on the pages.
- Extraction is as good as the PDF's text layer. Multi-column layouts, tables, equations and slide decks may extract in odd order, which lowers retrieval quality.
- Password-protected PDFs are rejected with a clear message.

## Calibrating the refusal threshold

Retrieval scores are BM25 scores, which have no natural 0–100 scale. The app divides the best passage's score by the maximum score a passage could get for that query, giving a **relevance estimate** from 0% to 100%. **It is an estimate, not a probability that the answer is correct.** The meter in chat is labelled accordingly.

The default refusal threshold is **25%**. On a small set of questions I wrote against a Python-documentation excerpt (18 answerable, 25 unanswerable) it kept 17 of 18 answerable questions and refused 24 of 25 unanswerable ones. That is a handful of hand-written questions on one document, **not a benchmark**. Your documents will behave differently.

- Lower the threshold (Settings) if good questions are being refused; raise it if off-topic questions slip through.
- Short documents give coarse scores because there are few passages to compare.
- Retrieval is keyword-based. A question in Devanagari script against an English document will usually find nothing and be refused; the refusal message suggests adding the document's English technical terms (Hinglish works well). There is no pre-retrieval model call, by design.

## Fallbacks

| Situation | What happens |
|---|---|
| Three.js fails to load, or WebGL is unavailable or lost | A CSS-only 3D cube replaces the scene and a notice explains why. Everything else works. |
| PDF.js fails to load | Uploading shows "The PDF engine could not be loaded" instead of failing silently. |
| `prefers-reduced-motion` or the in-app switch | Animations are removed or shortened, the 3D scene stops animating, and tilt and flip effects are neutralised. |
| No `window.storage` | In-memory mode with visible warnings. |
| API errors (auth, rate limit, network, server, empty, truncated reply) | A specific on-screen message with Retry where that makes sense. A failed question stays in the chat so you can retry it. |

## Performance and privacy

- The 3D scene pauses when the tab is hidden, drops to ~30 fps while you work in the dashboard, uses fewer particles on small screens, and frees its GPU resources when the page is closed.
- PDF text is processed in your browser. The only data that leaves it is what you send to the Anthropic API: the question plus the 4 retrieved passages for chat, or one passage for each exam question. Fonts and libraries are fetched from Google Fonts and cdnjs.

## What was tested, and what was not

Tested (about 330 automated checks):

- **Core logic in Node (72 checks):** tokenizer and stemmer, chunking and page tracking, BM25 and relevance, refusal threshold, MCQ validation, duplicate detection, spaced-repetition dates, topic statistics, and the API client with a mocked `fetch` (key sent only to the Anthropic host, error kinds, key scrubbing).
- **Real browser (headless Chromium via Playwright, about 260 checks):** login validation, upload of valid, corrupt, empty, scanned and non-PDF files, chat with refusal (zero model calls verified), grounded answers with citations, API failure modes with retry, the exam flow (malformed, truncated, invalid and partial model replies, keyboard answering, confirmation, review, retry, discard), flashcards and the 1 → 3 → 7 day schedule using a simulated clock, settings, persistence across reload with a test `window.storage`, erase-all, OS reduced motion, a 390 px phone layout with no horizontal scroll, sign-out clearing the key, and no console errors.

**Not tested — please check these yourself after deploying:**

- The **real Three.js r128** and **real PDF.js 3.11.174** from the CDN. My sandbox could not reach the CDN, so the 3D scene was run against a stub (this only proves the calls do not throw, not that they render correctly) and PDF extraction ran on a newer PDF.js build.
- The **real Anthropic API**. All model replies in my tests were mocked, so answer quality, and how well the model follows the JSON and citation instructions, are unmeasured.
- **Safari, Firefox and real phones.** Only Chromium was used.
- A screen reader. ARIA roles and keyboard paths were checked in code and by automated tests only.

## Known limitations

- Login is a demo and protects nothing.
- Each chat question is **stateless**: follow-ups such as "explain that again" do not remember earlier turns, and retrieval sees only the new question.
- Retrieval is keyword-based (BM25), so paraphrases with no shared words can be missed, and Devanagari-only questions against an English document usually fail.
- Topic labels on exam questions come from the model and can be inconsistent between questions ("Cell energy" and "Respiration"), which splits weak-topic statistics.
- With a small document there may be fewer than 5 sufficiently long passages, in which case passages are reused for different questions.
- Answers can be wrong even when cited. Always check the cited page.
- Large PDFs take time to process on a phone.

## Demo

*Add a screen recording here, for example `docs/demo.gif`:*

```md
![StudyBuddy 3D demo](docs/demo.gif)
```

## License

No license file is included. Add one (for example MIT) before publishing if you want others to reuse the code.
