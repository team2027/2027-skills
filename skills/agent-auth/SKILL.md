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

1. **DON'T**: scrape token *values* to copy-paste (`cat ~/.config/.../config.json`) — none exist until login. (Checking whether `CI`/`BROWSER`/the provider's `*_TOKEN` are set is fine — those suppress the popup; if set, scope them off the *login command* — `CI= BROWSER= <cli> login` — not session-wide.) DON'T drive the browser (xdotool, screenshot the X root, CDP, browser-agent logs). DON'T heavy-install an SDK (`npm i <pkg>`, 1000+ pkgs / ~2 min) just to authenticate.
2. **Get the CLI light & current**: `npx -y <cli-package>@latest` — this also dodges a *stale* global/brew binary already on PATH, which can have a broken login (an old Homebrew Railway **v4.x** fails its callback). Sanity-check with `<cli> --version`; if a stale one shadows it, call the fresh one by full path or `brew upgrade`/reinstall.
3. **Run the default login ONCE and keep that one process alive until the human approves.** The login starts a local callback server (e.g. `localhost:4321`); if its process dies — **or another app already holds that port** — the human's click 404s with the *other app's* error (check it's free first: `lsof -i :<port>`). So run it **foreground with a long timeout**, or as a *persistent* background job (`run_in_background`, `nohup … & disown`) — **never** a bare `… &` that ends with the shell call (that kills the server), and don't pass popup-suppressing flags (`--no-open`, `BROWSER=none`):
   ```bash
   npx -y <pkg>@latest login        # popup opens immediately; call returns on success (timeout ~300s)
   ```
   Backgrounded (`run_in_background`) returns immediately with no inline output — poll its output file for the URL, then the success line / process-exit; don't foreground-`sleep` (this harness blocks it).
4. **The popup opens in the human's browser automatically.** Tell them: "approve the login in the window that just opened." If it didn't open (headless), surface the URL — **except for localhost-callback CLIs** (Sanity `:4321`, wrangler `:8976`, …): on a remote/headless host the human's browser can't reach *this* box's `localhost`, so the pasted URL never completes the callback. There, skip the popup and use the token/env path instead. For device-code flows (Vercel, Stripe, Auth0) the CLI prints a `user_code` — read it from the output and echo it to the human so they can confirm the match before approving.
5. **Wait for the success line — or just the process exiting (some CLIs print none) — then verify** (whoami/status).

**Run the login once and wait — don't fire it repeatedly.** It opens a local callback server and blocks until the human approves; spawning extra logins because you didn't see instant success just leaves competing servers behind. One login, keep it alive, wait for the success line.

**If the browser login fails, read the CLI's error and follow its hint — don't blindly retry.** Many CLIs offer a device-code mode (e.g. `railway login --browserless`): it prints a URL + pairing code instead of using a localhost callback (more robust, works headless). Device-code mode usually needs a real TTY, so drive it with `npx noninteractive <cli> login --browserless` and read the URL/code from the returned `urls` array. Token env vars are the last-resort no-popup path.

**Some logins gate the popup behind an interactive prompt** ("Press any key…", an auth-method picker, a device-name field) and *fatal* when backgrounded — Railway, Heroku, Daytona, Auth0, Supabase, Convex. Drive the gate in a PTY, two ways:

- **`noninteractive`** (also auto-opens the OAuth URL it finds): `npx noninteractive <pkg> login` for an npm CLI, or `npx noninteractive start <binary> login` for a binary/brew CLI (`start` runs the literal command, not `npx`). **Then send the keystroke** — `npx noninteractive send <sess> ' '` (or `''` for Enter) — and `read --wait` for the URL/success. Just starting it leaves the login parked at the gate (the #1 failure).
- **`expect`** bundles spawn + keystroke + success/error exit in one line:

```bash
expect -c 'spawn <cli> login; expect -re "(any key|Browser|Device)"; send "\r"; set timeout 180; expect -re "logged in" {exit 0} -re "error|denied|no code" {exit 1} timeout {exit 2}'
```

**Per-provider commands for 20 CLIs** (exact login command, TTY caveats, verify, token env): see [`references/providers.md`](references/providers.md). Sanity and Netlify are below.

## Sanity  (popup: auto-opens · login: automatic · verified)

- Install: `npx -y @sanity/cli@latest` (bin `sanity`; `@sanity/cli` is lighter than the `sanity` meta-pkg and has full `login`). In a Studio project, `npx sanity login` is instant.
- Login: `npx -y @sanity/cli@latest login --provider google` (swap `google`↔`github` for the human's account). **Pass `--provider`** — bare `sanity login` shows an interactive picker that hangs when backgrounded. Run it once, persistent; the popup opens (`Opening browser at <url>`) and the `localhost:4321` callback must stay reachable until approval.
- Success: exits 0 (callback server shuts down) and prints `Login successful` on current versions — but detect completion by the exit + `sanity debug` → `User: <email>` (authoritative), not by grepping the string. Verified live as `User: <email>`.
- Verify: `npx -y @sanity/cli@latest debug` → `User: <email>`. (`sanity whoami` doesn't exist.)
- CI / no popup: `export SANITY_AUTH_TOKEN=<token>` (read before config), or `sanity login --with-token < token.txt`.

## Netlify  (popup: auto-opens · login: automatic)

- Install: `npx -y netlify-cli` (NOT `npx netlify` — that's the JS API client). Bin `netlify`/`ntl`. `NETLIFY_TELEMETRY_DISABLED=1` skips the telemetry prompt.
- Login: `npx -y netlify-cli login`. Default opens the popup and polls Netlify's API (no localhost callback to collide), stores the token, prints `You are now logged into your Netlify account!`. If a token is already cached it short-circuits `Already logged in …`. Do NOT use `--request` (the copy-a-URL ticket flow) or `BROWSER=none`.
- Verify: `npx -y netlify-cli status` → "Current Netlify User" block.
- CI / no popup: `export NETLIFY_AUTH_TOKEN=<pat>` or `--auth <token>` on any command (PAT from app.netlify.com/user/applications).
