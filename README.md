**AI development disclosure:** This project was developed with assistance from the free versions of ChatGPT, Grok, and Claude (LLMs), with the user providing ideas and testing while the AIs and user collaboratively suggested, generated, reviewed, and refined code and solutions.


# O'Reilly EPUB downloader

O'Reilly provides all of their books in EPUB format, but only through their own web reader.

This script downloads the individual files that make up a book you have access to and reassembles them into a normal, standalone `.epub` file, so you can read it in whatever app or device you prefer — including for accessibility reasons the built-in reader doesn't support well.

Before any usage, please read the [O'Reilly Terms of Service](https://learning.oreilly.com/terms/). This tool is meant for personal, offline access to books you already have a legitimate subscription to — not for redistribution.

## Features

- Rebuilds a complete, valid EPUB: every chapter, stylesheet, and image the API lists, with all internal links and references rewritten so the result works as a normal self-contained book, not a pile of loose files.
- Authenticates with your actual browser cookies, not just a short-lived token, so a session lasts as long as your real login does instead of expiring within the hour.
- Understands *why* a request failed instead of just reporting a bare error: it tells you whether your token has genuinely expired (checked locally, no network needed) or whether something else is blocking the request regardless of your login being fine.
- Can open a real, native browser window to log in directly when nothing else works — see [Using `--webview`](#using---webview) below.
- Retries transient failures (rate limiting, brief network blips) on its own; a handful of flaky files won't take down an entire large book.

## Requirements

- Python 3.9+
- [`uv`](https://docs.astral.sh/uv/) (recommended), or `pip`
- Optionally, the `pywebview` package if you want to use `--webview` (see below) — not needed for normal use

## Installation

The script declares its own dependencies inline, so with [`uv`](https://docs.astral.sh/uv/) installed you don't need a separate install step at all — `uv run` fetches them automatically the first time:

```
$ uv run oreilly_downloader.py 9781633437777 --cookies cookies.json
```

Without `uv`, install the dependencies yourself and run it with `python3`:

```
$ pip install aiohttp lxml yarl
$ python3 oreilly_downloader.py 9781633437777 --cookies cookies.json
```

Every example below uses `python3 oreilly_downloader.py ...`; substitute `uv run oreilly_downloader.py ...` if that's how you installed it.

## Quick start

1. Log into [learning.oreilly.com](https://learning.oreilly.com) in your browser.
2. Install a cookie-export extension — e.g. [Cookie-Editor](https://cookie-editor.com/) (Chrome, Firefox, Edge) or EditThisCookie — and export **all cookies for the site** to a file called `cookies.json`. You don't need to hand-pick which cookies to include; anything not scoped to `oreilly.com` is ignored automatically.
3. Find the book's ID: it's the string of digits in the book's learning.oreilly.com URL. For `https://learning.oreilly.com/library/view/some-book/9781633437777/`, the id is `9781633437777`.
4. Run:

   ```
   $ python3 oreilly_downloader.py 9781633437777 --cookies cookies.json
   Authentication successful.
     JWT valid for ~47m
   listing https://learning.oreilly.com/api/v2/epubs/urn:orm:book:9781633437777/files/
   downloading 342 files (concurrency=8)
   saved current cookies to cookies.json
   created 9781633437777.epub
   ```

That's it — `9781633437777.epub` is a complete, standalone book.

Re-run the exact same command whenever you want another book (just change the id). `--cookies` keeps itself up to date on every run (see below), so in practice you'll only need to re-export from the browser once in a while, not before every download.

## Command-line reference

```
python3 oreilly_downloader.py BOOK_ID [options]
```

| Argument | Description |
| --- | --- |
| `book_id` | *(required)* The numeric book id from the book's learning.oreilly.com URL. |
| `--cookies PATH` | Path to a JSON file of learning.oreilly.com's cookies. The recommended way to authenticate — see [Authentication](#authentication) below. |
| `--jwt VALUE` | Just the `orm-jwt` cookie's value, as a quick one-off alternative to `--cookies`. Expires within about an hour and can't be refreshed — fine for a single small book, not for anything that might outlive it. |
| `--concurrency N` | How many files to download in parallel. Default: `8`. Lower this if you see a lot of `403` failures on a large book — it usually means the API's rate limiting is kicking in. |
| `--webview` | If there's no session yet, or the one you gave fails outright, open a real browser window so you can log in directly. See [Using `--webview`](#using---webview). |
| `--webview-profile PATH` | Where `--webview` keeps its own persistent browser profile between runs. Defaults to a folder next to `--cookies`. |
| `-h`, `--help` | Show the built-in help (kept in sync with this document, and the source of truth if the two ever disagree). |

## Authentication

### Why `--cookies` instead of just a JWT

O'Reilly authenticates API requests with a short-lived token (the `orm-jwt` cookie), which on its own typically expires within an hour. `--jwt` gives you just that value, quickly, but once it expires there's no way to renew it — you have to go back to the browser for a new one.

`--cookies` instead loads your *entire* cookie set for the site. That matters for two reasons:

- Other cookies (notably `orm-rt`, a refresh token) may be what lets O'Reilly's own backend transparently hand back a fresh token on an ordinary request once the old one has expired — the same mechanism that keeps a real browser tab logged in without you noticing. Whether this actually happens depends on O'Reilly's server, not on this script; when it works, requests just keep succeeding with the token quietly renewed underneath.
- After the run, the script writes your **current** cookies — including anything the API rotated in along the way — back to the same file. Point `--cookies` at the same path every time, and each run continues from wherever the last one left off.

Every field in your exported cookies (`domain`, `path`, `secure`, `httpOnly`, `sameSite`, `expirationDate`, and so on) is read and genuinely used, not just the value — so a cookie is only ever sent where and while it's actually supposed to be valid, the same way a real browser handles it. A cookie that's already past its own expiry date is skipped at load time with a warning rather than sent anyway. The file the script writes back is in that same full shape, so it stays compatible with re-importing straight back into a browser extension if you ever want to.

You can combine `--cookies` with `--jwt` to override just the token while keeping the rest of the cookies from the file.

### Understanding a failed login

**"No cookies/JWT provided."** Neither `--cookies` nor `--jwt` was given. The download proceeds anyway, but will typically only get content O'Reilly serves to logged-out visitors — usually nothing, for a real book.

**"Authentication check failed: HTTP 401/403 - ..."** This does *not* necessarily mean your login expired. Learning.oreilly.com sits behind **Akamai Bot Manager** (recognizable by cookies named `bm_s`, `bm_sz`, `bm_so`, `bm_lso`, and `_abck`), which can reject a request that merely *looks* automated — an unexpected header, an unconvincing browser fingerprint — independently of whether your actual session is completely valid. It can also let an identical request through moments later.

To help tell these apart, the script decodes your `orm-jwt`'s own expiry claim locally — no network request needed — and prints it before attempting anything:

```
  orm-jwt's own expiry claim: valid for ~52m more
```

or

```
  orm-jwt's own expiry claim: expired 3600s ago
```

If a failure shows up *and* that claim already says "expired," the message tells you plainly to re-export cookies from your browser. If the claim still shows time remaining, the message instead explains that this looks like Akamai rejecting the request itself, and suggests just trying again shortly — or using `--webview` (below) to get past it directly.

A handful of `FAILED to download ...` lines at the very end, naming specific files, usually just means transient rate-limiting on a big book. Re-run the exact same command; already-downloaded files are re-fetched too (the script always rebuilds the `.epub` from scratch), but that's normally quick.

## Using `--webview`

Some situations can't be solved by better cookies alone:

- Akamai's own bot-detection cookies (`_abck`, `bm_sz`, ...) are populated by an obfuscated JavaScript sensor that only runs inside a real browser page — never in a script's HTTP requests, no matter how convincing the headers.
- `orm-jwt`/`orm-rt` are only ever issued after actually completing O'Reilly's login flow.

`--webview` opens a real, native browser window — using your OS's actual browser engine (WebKit on Linux/macOS, WebView2 on Windows), not a simulation — so you can log in exactly as normal, and Akamai's sensor gets to do whatever it needs to. Whatever cookies that produces, Akamai's included, are picked up automatically.

### Setup

`pywebview` is an optional dependency, installed separately (it isn't downloaded automatically, since most usage doesn't need it):

```
$ pip install pywebview --break-system-packages
```

It also needs a native GUI toolkit already present on your system:

- **Linux**: GTK + WebKit2 (e.g. `sudo apt install python3-gi gir1.2-webkit2-4.0` on Debian/Ubuntu — check [pywebview's docs](https://pywebview.flowrl.com/) if package names differ for your distro)
- **macOS**: nothing extra — it uses the built-in WebKit
- **Windows**: nothing extra — it uses the built-in WebView2 (bundled with Windows 10/11)

**`--webview` needs an actual display** (X11, Wayland, or a macOS/Windows desktop session). It will not work over a plain SSH terminal or inside a headless container.

### Usage

```
$ python3 oreilly_downloader.py 9781633437777 --cookies cookies.json --webview
```

`--webview` kicks in automatically in two situations:

- **No cookies at all yet** (e.g. `cookies.json` doesn't exist) — it opens the login window immediately, before attempting anything else.
- **The cookies you gave fail the authentication check** — it retries a few times first (in case it's just a transient block), and only opens the window if that still doesn't succeed.

A window titled *"Log in to O'Reilly — this window closes itself once you're in"* will appear. Log in there as you normally would. The script polls in the background, and as soon as it detects a live session, **the window closes itself automatically** — you don't need to close it manually. If nothing happens within 10 minutes, it gives up and the window closes on its own.

Once it succeeds, you'll see:

```
Authentication successful. (via webview login)
```

and the resulting cookies are saved back to `--cookies` right away, so a future run won't need the window again unless the whole session dies.

`--webview-profile PATH` controls where the embedded browser keeps its own persistent profile (separate from `--cookies`) between runs — by default, a `.oreilly_webview_profile` folder next to your cookies file. This means logging in via `--webview` is normally a one-time thing, not something that happens on every run.

## How it works, briefly

1. Lists every file in the book via O'Reilly's own API (paginating through all results before downloading anything).
2. Downloads each file concurrently (bounded by `--concurrency`), retrying rate-limited or transiently-failed requests with backoff.
3. Converts each HTML chapter to valid, self-contained XHTML: rewrites every internal link, image, and stylesheet reference from the API's absolute paths to correct relative ones, fixes up SVG-wrapped cover images, and patches over a few known HTML/XML interoperability quirks (embedded `<style>` content, XML namespace declarations) so the result renders correctly both in strict EPUB readers and in the more lenient parsers most real reading apps actually use.
4. Packages everything into a proper EPUB container (`mimetype`, `META-INF/container.xml`, and the book's own package document) and writes it out atomically, so an interrupted run never leaves a corrupt half-written file behind.

## Limitations

- This can't defeat Akamai's bot detection outright — `--webview` works around it by using a real browser, not by tricking the anti-bot system.
- A cookie export only lasts as long as the underlying login session does. Once that's genuinely dead (not just the short-lived token), you need to log in again — either back in your regular browser, or via `--webview`.
- Only works for books your account actually has access to; it downloads exactly what the API would already show you in the web reader; nothing more.

## Contributing

I am not really interested in adding any major features to this project. I will accept fixes, but nothing that adds a significant amount of new code.

If you feel like something is missing, feel free to fork. You may also look at rejected pull requests, maybe someone already worked on something similar.

## Similar projects

- [https://github.com/lorenzodifuccia/safaribooks](https://github.com/lorenzodifuccia/safaribooks) (python)
- [https://github.com/hurlenko/orly](https://github.com/hurlenko/orly) (rust)
- [https://github.com/jenni/obooks](https://github.com/jenni/obooks) (javascript)
- [https://github.com/rahulvramesh/oreilly-books-grabber](https://github.com/rahulvramesh/oreilly-books-grabber) (go)
