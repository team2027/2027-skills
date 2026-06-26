---
name: agent-auth
description: >-
  Log a real human into a dev tool's CLI by opening the OAuth browser popup
  automatically and letting the login complete itself — no token copy-paste.
  Use when you need to log in to / authenticate a CLI, when a command like
  `sanity login` or `netlify login` opens a browser, or on any interactive
  login / OAuth popup. You make the popup appear and wait; the human just
  clicks Approve. You NEVER automate the browser or scrape tokens.
---

# Logging a human into a CLI

Goal: make the OAuth **browser popup open automatically**, let the human click Approve, and let the CLI **store the token itself**. You don't copy tokens; you don't drive the browser.

## Fast-path (do this every time)

1. **DON'T**: hunt for tokens (`env | grep`, `cat ~/.config/.../config.json`) — none exist until login. DON'T drive the browser (xdotool, screenshot the X root, CDP :9222, browser-agent logs). DON'T heavy-install an SDK (`npm i <pkg>`, 1000+ pkgs / ~2 min) just to authenticate.
2. **Get the CLI light**: `npx -y <cli-package>@latest` (no global install; warm runs ~2s).
3. **Run the DEFAULT login backgrounded.** Default = the mode that auto-opens the browser; do NOT pass popup-suppressing flags (`--no-open`, `BROWSER=none`). Keep the PID; capture stdout for a fallback URL:
   ```bash
   npx -y <pkg> login > /tmp/login.out 2>&1 & PID=$!
   for i in $(seq 1 30); do grep -qE 'https?://' /tmp/login.out && break; sleep 1; done
   ```
   Background with a bare `&` — not `( … & )`, which throws away `$!`.
4. **The popup opens in the human's browser automatically.** Tell them: "approve the login in the browser window that just opened." Only if it didn't open (headless / no desktop) do you paste the captured URL: `grep -oE 'https?://[^ ]+' /tmp/login.out | head -1`.
5. **`wait $PID`** — the CLI catches the OAuth callback, **stores the token itself**, and exits. Then verify with the CLI's whoami/status.

Two things make or break the popup: don't pass a flag that suppresses the open, and **pick a provider up-front** if the CLI would otherwise show an interactive picker — a picker needs a TTY and hangs a backgrounded login.

Fallbacks: (a) headless box, no browser → surface the URL (step 4) for the human to open elsewhere; (b) CI / unattended → use the token env var below, no popup. Only reach for `npx noninteractive <name> [args]` if a CLI won't open or even print its URL without a real TTY.

## Sanity  (popup: auto-opens · login: automatic)

- Install: `npx -y @sanity/cli@latest` (bin `sanity`; `@sanity/cli` is lighter than the `sanity` meta-pkg and has full `login`). In a Studio project, `npx sanity login` is instant.
- Login: `npx -y @sanity/cli@latest login --provider google > /tmp/sanity_login.out 2>&1 & SP=$!` (swap `google`↔`github` for the human's account). **Pass `--provider`** — bare `sanity login` shows an interactive picker that hangs when backgrounded. Default (no `--no-open`) prints `Opening browser at <url>` and opens it; on macOS `BROWSER=…` is ignored so the popup always opens.
- Auto-completes: the popup redirects to `http://localhost:4321/callback`, the CLI stores the token and the process EXITS 0 — there's no success text, so `wait $SP` is your signal. (Callback is on `localhost`, so it completes on the human's own machine.)
- Verify: `npx -y @sanity/cli@latest debug` → `User: <email>`. (`sanity whoami` doesn't exist.)
- CI / no popup: `export SANITY_AUTH_TOKEN=<token>` (read before config), or `sanity login --with-token < token.txt`.

## Netlify  (popup: auto-opens · login: automatic)

- Install: `npx -y netlify-cli` (NOT `npx netlify` — that's the JS API client). Bin `netlify`/`ntl`. `NETLIFY_TELEMETRY_DISABLED=1` skips the telemetry prompt.
- Login: `npx -y netlify-cli login > /tmp/ntl.out 2>&1 & NP=$!`. Default `netlify login` prints `Opening <url>`, opens the popup, polls until the human approves, **stores the token**, and prints `You are now logged into your Netlify account!` before exiting. Do NOT use `--request` (the no-browser ticket flow that makes the human copy a URL while you poll `--check`), and do NOT set `BROWSER=none`.
- Auto-completes: `wait $NP` returns once the token is stored. If a token is already cached it short-circuits with `Already logged in …` and exits (no popup needed).
- Verify: `npx -y netlify-cli status` → "Current Netlify User" block.
- CI / no popup: `export NETLIFY_AUTH_TOKEN=<pat>` or `--auth <token>` on any command (PAT from app.netlify.com/user/applications).
