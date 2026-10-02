# MINI — the simplest web browser

MINI is a complete web browser in **one HTML file**. There is no server to run, no build step, and no dependencies to install. You open `MINI.html`, type an address or a search, and the page loads inside MINI.

It has **no tabs, no toolbar, and no hotbar**. The interface is a single full-screen view. The only visible UI is a search box on the home screen, a thin loading line, and a small status toast.

Under the hood MINI is a feature-complete sibling of **JUNIOR**, a tabbed version of the same browser. It has the same network engine and page-handling logic, with the chrome stripped away.

---

## Table of contents

1. [How it works](#how-it-works)
2. [Quick start](#quick-start)
3. [Features](#features)
4. [Keyboard shortcuts](#keyboard-shortcuts)
5. [User interface](#user-interface)
6. [Architecture](#architecture)
7. [Navigation lifecycle](#navigation-lifecycle)
8. [The injected page script](#the-injected-page-script)
9. [Configuration](#configuration)
10. [MINI vs. JUNIOR vs. the original MINI](#mini-vs-junior-vs-the-original-mini)
11. [Limitations and known behavior](#limitations-and-known-behavior)
12. [Privacy and security notes](#privacy-and-security-notes)
13. [Troubleshooting](#troubleshooting)
14. [Customizing](#customizing)
15. [Credits](#credits)

---

## How it works

A normal web page cannot freely fetch other websites. The browser's same-origin policy and CORS rules block it. MINI gets around this by doing its networking outside the browser's normal HTTP stack:

1. **[libcurl.js](https://github.com/ading2210/libcurl.js)** is a WebAssembly build of `libcurl`. It is loaded from a CDN and runs inside the page.
2. libcurl.js sends its traffic over a **[Wisp](https://github.com/MercuryWorkshop/wisp-protocol) WebSocket gateway** (`wss://wisp.mercurywork.shop` by default). The gateway relays raw TCP connections, so MINI can make real requests to arbitrary sites.
3. MINI downloads the target page's HTML and rewrites it before display. It injects a `<base>` tag and a small control script, and strips headers and meta tags that would block embedding.
4. The rewritten HTML is rendered in a sandboxed `<iframe>` through `srcdoc`.
5. The injected script catches link clicks, form submissions, popups, and shortcut keys inside the page. It reports them to MINI with `postMessage`, and MINI performs the next fetch.

```
┌──────────────┐   postMessage    ┌───────────────────────────┐
│ sandboxed    │ ───────────────► │ MINI (parent page)        │
│ iframe       │ ◄─────────────── │  • libcurl.js (WASM)      │
│ (rewritten   │   srcdoc / blob  │  • history, rendering     │
│  page HTML)  │                  └─────────────┬─────────────┘
└──────────────┘                                │ WebSocket (Wisp)
                                                ▼
                                       wss://wisp.mercurywork.shop
                                                │ TCP / TLS
                                                ▼
                                           target website
```

---

## Quick start

1. Save `MINI.html` anywhere, or host it on any static host.
2. Open it in a modern desktop or mobile browser. You need internet access, because libcurl.js is loaded from jsDelivr and the Wisp gateway is remote.
3. Wait for the status line to change from *"Initializing WebAssembly runtime..."* to **"Ready!"**. The button changes from *Loading WASM...* to **Go**.
4. Type a URL or a search term and press **Enter** or click **Go**.

### Opening a page directly

MINI can start on a page without going through the home screen:

```
MINI.html?url=example.com
MINI.html#https://example.com
```

Both forms accept anything you could type in the search box. They also accept search terms, such as `MINI.html?url=cute%20cats`.

---

## Features

### Browsing

- **Smart address input.** The input is classified as follows:

  | You type | What MINI does |
  |---|---|
  | `https://example.com/page` | Loads it as given |
  | `//example.com` | Uses `https:` |
  | `example.com`, `sub.site.org/path?x=1` | Adds `https://` |
  | `localhost:3000`, `192.168.1.5` | Adds `http://` |
  | `how do magnets work` | Searches on DuckDuckGo's HTML page |
  | `javascript:`, `data:`, `blob:` and `about:` | Rejected |

- **Link navigation.** Clicking a link loads it in the same view. `target="_blank"` and `window.open()` also load in the same view, since there are no tabs. Middle-click behaves the same way.
- **Form support.** Both `GET` and `POST` forms work. MINI honors `formaction`, `formmethod`, and the name and value of the clicked submit button. This makes search boxes, logins, and other form flows usable.
- **Back, forward, reload.** MINI keeps a full session history. History entries store the method and body, so reloading a `POST` result re-submits it the way it was first sent.
- **Hash navigation.** In-page `#anchors` and `hashchange` events are tracked, so the stored URL stays accurate.

### Content handling

- **Charset detection.** MINI reads the charset from the `Content-Type` header, falls back to a `<meta charset>` sniff of the first 2 KB, and then to UTF-8. Legacy encodings render correctly.
- **Redirects.** HTTP redirects are followed automatically. When the final URL differs, the status toast says *"Redirected to …"*.
- **Refresh redirects.** Both the `Refresh:` HTTP header and `<meta http-equiv="refresh">` are followed, up to 5 hops (`MAX_META_REFRESH`) so loops cannot trap you.
- **Non-HTML content.** Images, video, audio, PDFs, JSON, XML, and plain text open directly in the view through a blob URL.
- **Downloads.** Responses with `Content-Disposition: attachment`, and file types the viewer cannot display, are saved to disk with the correct filename. Links that carry the `download` attribute work too.
- **Embedding fixes.** To make pages render inside the iframe, MINI removes `<meta>` Content-Security-Policy and X-Frame-Options tags, `integrity=` attributes, and `crossorigin` attributes. It also normalizes `<base>` tags.
- **Error page.** If a site cannot be reached, you get a readable *"Can't reach this page"* screen showing the address and error message. It is also added to history, so Reload retries it.

### Interface

- **A single full-screen view** with no tabs and no toolbar.
- **Home screen as address bar.** The same search box serves as the "address bar". It pre-fills with the current URL when you bring it up.
- **Loading line.** A 2px animated line at the very top of the screen while a request is in flight.
- **Status toast.** A small message at the bottom-left reports loading, redirects, HTTP errors, downloads, and failures. It fades after a few seconds.
- **Dark theme** matching JUNIOR (`#111827` background, emerald accent).
- **Page cloak.** The tab title is `Google` and the favicon is Google's, as in the original MINI.

---

## Keyboard shortcuts

Because there are no buttons, navigation is keyboard-driven. Shortcuts work **even when focus is inside the loaded page**. The injected script forwards them to MINI.

| Shortcut | Action |
|---|---|
| **Alt + ←** | Back |
| **Alt + →** | Forward |
| **Alt + R** | Reload the current page |
| **Ctrl/⌘ + L** or **Alt + L** | Open the address/search box (pre-filled with the current URL) |
| **Alt + Home** | Open the address/search box |
| **Esc** | Close the address/search box and return to the page |
| **Enter** | Go (inside the search box) |

> On touch-only devices there is no way to trigger back, forward, or reload yet. See [Customizing](#customizing) for ideas.

---

## User interface

| Element | Where | Purpose |
|---|---|---|
| Home screen | Full screen overlay | Search/address input, status line, shortcut hint |
| Content view | Full screen | The page you are browsing, in an iframe |
| Loading line | Top edge, 2px | Shows that a request is in progress |
| Status toast | Bottom-left | Transient status and error messages |

The home screen shows at startup and again whenever you press an address shortcut. When a page is loaded successfully, the overlay hides automatically.

---

## Architecture

MINI is organized into a few cooperating parts, all inside the single file.

### State

| Variable | Meaning |
|---|---|
| `ready` | libcurl.js is initialized and the gateway is set |
| `pending` | A navigation requested before WASM was ready; runs once ready |
| `history_` / `idx` | Session history array and the current position |
| `token` | Monotonic counter used to cancel stale in-flight loads |
| `blobUrl` | Current blob URL, revoked when replaced |
| `loading` | Whether a request is running |

### Main functions

| Function | Responsibility |
|---|---|
| `normalizeInput()` | Turns user input into a URL or a search URL |
| `navigate(target, opts)` | Fetches, decodes, rewrites, and displays a page; manages history |
| `showPage()` | Writes HTML to `srcdoc`, or a blob to `src` |
| `showError()` | Renders the error page and records it in history |
| `downloadUrl()` / `saveBlob()` | Fetches and saves files |
| `goBack()` / `goForward()` / `reload()` | History navigation (replays the stored method and body) |
| `handleKey()` | Shared keyboard shortcut handler (parent page and iframe) |
| `showHome()` | Shows or hides the address overlay |
| `INJECT` | The script string added to every proxied HTML page |

### Cancellation

Each navigation increments `token`. After every `await`, the loader checks that its token is still current. Starting a new navigation therefore silently abandons the old one, with no stale content flashing in.

---

## Navigation lifecycle

1. **Input.** The user submits the form, clicks a link, or triggers back/forward/reload.
2. **Guard.** If WASM is not ready yet, the request is queued in `pending`.
3. **Fetch.** `libcurl.fetch()` requests the URL with the browser's user agent and `redirect: 'follow'`.
4. **Header checks.** MINI looks for `Refresh` redirects and `Content-Disposition: attachment`.
5. **Classify by `Content-Type`:**
   - HTML: decode, then follow any meta refresh, then rewrite, then inject.
   - Viewable media or text: wrap in a blob.
   - Anything else: download.
6. **History.** Push a new entry for a normal navigation, or replace the current entry for back, forward, and reload.
7. **Render.** Write to the iframe, hide the home overlay, and update the status toast.
8. **Cleanup.** Clear the loading line, and revoke the old blob URL.

---

## The injected page script

Every proxied HTML page receives a small script immediately after `<head>`. It runs inside the sandboxed iframe and communicates with MINI via `postMessage`.

| Page event | What the script does |
|---|---|
| Click on `<a href>` or `<area href>` | Cancels default, sends `nav` with the resolved URL |
| Link with `download` attribute | Sends `download` instead |
| Middle-click on a link | Sends `nav` (same view) |
| Form submit | Builds URL-encoded data, sends `nav` as `GET` or `POST` |
| `window.open(url)` | Sends `nav` and returns `null` |
| `target` attributes | Removed on `DOMContentLoaded` |
| `hashchange` | Sends `hash` so the stored URL stays accurate |
| Alt/Ctrl/⌘ shortcuts and Esc | Sends `key` so shortcuts work inside pages |

Links beginning with `#`, `javascript:`, `mailto:`, `tel:`, `sms:`, `data:`, or `blob:` are left alone. MINI also verifies that messages come from its own iframe, and only accepts `http(s)` URLs.

---

## Configuration

All configuration is at the top of the `<script>` block:

```js
const GATEWAY = 'wss://wisp.mercurywork.shop';  // Wisp WebSocket server
const MAX_META_REFRESH = 5;                      // max chained refresh redirects
```

- **`GATEWAY`**: point this at a different public Wisp server, or at one you host yourself, if the default is slow or unavailable.
- **`MAX_META_REFRESH`**: raise or lower the redirect hop limit.
- **libcurl.js version**: the `<script src>` in `<head>` uses `@latest`. For reproducible behavior, pin a version, for example `libcurl.js@0.6.x`, instead of `@latest`.
- **Search engine**: change the URL in `normalizeInput()` (`https://html.duckduckgo.com/html/?q=`) to use another HTML-friendly search page.
- **Tab disguise**: edit `<title>` and the `<link rel="icon">` to change what the browser tab displays.

---

## MINI vs. JUNIOR vs. the original MINI

| Capability | Original MINI | JUNIOR | **MINI (this version)** |
|---|---|---|---|
| Tabs | No | Yes | **No** |
| Toolbar / buttons | No | Yes | **No** |
| Back / forward / reload | No | Yes (buttons + keys) | **Yes (keys only)** |
| Address input | Home only | Persistent bar | **Home overlay, summonable** |
| Smart URL / search handling | Basic | Yes | **Yes** |
| Form `POST` support | No | Yes | **Yes** |
| Rendering method | `document.write` | Sandboxed iframe | **Sandboxed iframe** |
| Charset detection | No | Yes | **Yes** |
| Refresh redirects | No | Yes | **Yes** |
| Downloads and non-HTML files | No | Yes | **Yes** |
| Error page | Alert box | Yes | **Yes** |
| Stale-load cancellation | No | Yes | **Yes** |
| Open in new tab (Ctrl-click) | No | Yes | **N/A: opens in same view** |

---

## Limitations and known behavior

- **Online-only.** It needs a working CDN for libcurl.js and a reachable Wisp gateway.
- **Gateway-dependent.** Speed and availability depend on the Wisp server you point to.
- **Page scripts are not proxied.** Only the main document, navigations, form posts, and downloads go through libcurl. A page's own JavaScript calls (`fetch`, XHR, WebSockets, some images and fonts) go straight from your browser and can fail because of CORS or mixed-content rules. Script-heavy single-page apps and sites with strict anti-embedding or anti-bot protections may not work fully.
- **Sandboxed iframe.** The sandbox allows scripts, forms, popups, and downloads, but not same-origin access. Pages therefore cannot use cookies, `localStorage`, or IndexedDB the way they would on their real origin, and logins that depend on them may not persist.
- **No cookie jar.** MINI does not manage cookies between requests.
- **No tabs or bookmarks** by design.
- **No touch gestures** for back, forward, or reload. Those actions are keyboard-only.
- **Same-view popups.** Anything that tries to open a new window navigates the current view instead.
- **Non-`http(s)` links** such as `mailto:` and `tel:` are ignored.
- **Large files.** Responses are buffered fully in memory before display or download.

---

## Privacy and security notes

- **The gateway operator is in the path.** All of MINI's traffic flows through the Wisp server you configure. Because libcurl.js speaks TLS inside WebAssembly, the gateway generally relays encrypted connections, but it still sees which hosts you connect to and when. Use a gateway you trust, or host your own.
- **MINI does not log or store** your history beyond the current page session. Reloading the page clears it.
- **Passwords and sensitive logins** are best avoided in any proxy-style browser. Treat it as a convenience tool, not a hardened privacy browser.
- **Content-security stripping.** MINI removes CSP and X-Frame-Options meta tags so pages can render in the iframe. Treat unfamiliar sites the same way you would in any browser, and keep the iframe sandbox in place.
- **Responsible use.** Follow the terms of the networks you use, the gateway's rules, and the terms of the sites you visit.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Button stuck on *Loading WASM...* | The CDN script did not load. Check your connection or an ad/script blocker, and try a pinned libcurl.js version. |
| *"WASM init failed"* / button says *Failed* | WebAssembly is blocked, or the CDN is unreachable. Try another browser or network. |
| *"Can't reach this page"* | The gateway is down or the site refused the connection. Press **Alt+R** to retry, or change `GATEWAY`. |
| Page loads but looks broken | The page depends on its own scripts, XHR, or cookies, which the sandbox restricts. See [Limitations](#limitations-and-known-behavior). |
| Nothing happens on shortcuts | Click the page once to give it focus, or check that your browser or OS does not reserve `Alt+←/→`. |
| Search shows an odd layout | MINI uses DuckDuckGo's lightweight HTML results. Change the search URL in `normalizeInput()` if you prefer another engine. |
| Download did not start | Your browser may be blocking automatic downloads. Allow downloads for the page. |

---

## Customizing

Ideas that fit the single-file design:

- **Touch gestures.** Add a swipe or long-press handler that calls `goBack()`, `goForward()`, and `showHome(true)`.
- **Persistent history.** Save `history_` to `localStorage` in the parent page and restore it on load.
- **Custom homepage.** Call `navigate()` on a default URL from the `load` handler.
- **Alternate gateways.** Let the user pick a Wisp server from a list, or fall back automatically when the first fails.
- **Ad or script filtering.** Strip `<script>` or `<iframe>` tags matching a block list inside the HTML-rewrite step in `navigate()`.

---

## Credits

- **[libcurl.js](https://github.com/ading2210/libcurl.js)** provides the WebAssembly networking layer.
- **[Wisp protocol](https://github.com/MercuryWorkshop/wisp-protocol)** and the public gateway at `wisp.mercurywork.shop` provide the TCP-over-WebSocket transport.
- **JUNIOR** is the tabbed sibling this version's feature set is based on.

MINI: *the simplest web browser.*
