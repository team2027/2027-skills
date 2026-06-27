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

1. **DON'T**: scrape token *values* to copy-paste (`cat ~/.config/.../config.json`) — none exist until login. (Checking whether `CI`/`BROWSER`/the provider's `*_TOKEN` are set is fine — those suppress the popup; if set, scope them off the *login command* — `CI= BROWSER= <cli> login` — not session-wide.) DON'T drive the browser (xdotool, screenshot the X root, CDP, browser-agent logs). DON'T heavy-install an SDK (`npm i <pkg>`, 1000+ pkgs / ~2 min) just to authenticate. DON'T reach for a provider's MCP auth tool (e.g. `mcp__sanity__authenticate`) — drive the CLI's own login.
2. **Get the CLI light & current**: `npx -y <cli-package>@latest` — this also dodges a *stale* global/brew binary already on PATH, which can have a broken login (an old Homebrew Railway **v4.x** fails its callback). **Binary-only CLIs** (flyctl, render, daytona, stripe, auth0 — no npm package) skip the `npx` step entirely: install the documented `brew`/`curl` binary instead, but still `--version`-check *that* binary and watch for a stale older copy (brew `render` ships v1.x while v2.x exists). Sanity-check `<cli> --version 2>&1 | grep -v 'npm warn'` — it MUST print a real version. Blank/error = a stale binary shadowing it (`brew upgrade`/full path) **or** the `npx` wrapper never fetched its native binary (`npx -y @railway/cli` does this → `npm i -g <cli> --foreground-scripts`, or brew). A blank `whoami`/`logout` means the same — NOT logged-out. **Never `sudo`** to install or fix a CLI — on shared Macs `brew` often fails (`/opt/homebrew/Cellar is not writable`) and sudo isn't available; fall back to a user-level install (the CLI's GitHub-release Darwin tarball run from the scratchpad, or a vendor `curl … | bash` that lands in `~/`), then invoke it **by full path** — a `~/.<tool>` install isn't on a bash call's PATH. Don't retry the same install with sudo.
3. **Run the default login ONCE and keep that one process alive until the human approves.** The login starts a local callback server (e.g. `localhost:4321`); if its process dies — **or another app already holds that port** — the human's click 404s with the *other app's* error (check it's free first: `lsof -i :<port>`). So run it **foreground with a long timeout**, or as a *persistent* background job (`run_in_background`, `nohup … & disown`) — for a **localhost-callback** CLI **never** use a bare `… &` that ends with the shell call (that kills the callback server). Device-code / remote-poll CLIs (Render, Fly, Netlify) have no local server, so a bare `&` is harmless there — but `run_in_background` is still better, since it lets you surface the device code before the human approves. Don't pass popup-suppressing flags (`--no-open`, `BROWSER=none`):
   ```bash
   npx -y <pkg>@latest login        # popup opens immediately; call returns on success (timeout ~300s)
   ```
   Backgrounded (`run_in_background`) returns immediately with no inline output — poll its output file (bounded loop) for the URL then the success line / process-exit. Only a *bare top-level* `sleep` is blocked here; `/bin/sleep` or a `sleep` *inside* a loop works (or the Monitor tool).
4. **The popup opens in the human's browser automatically.** Tell them: "approve the login in the window that just opened." If it didn't open (headless), surface the URL — **except for localhost-callback CLIs** (Sanity `:4321`, wrangler `:8976`, …): on a remote/headless host the human's browser can't reach *this* box's `localhost`, so the pasted URL never completes the callback. There, skip the popup and use the token/env path instead. For device-code flows (Vercel, Stripe, Auth0, Render, WorkOS, Convex) the CLI prints a `user_code` — read it from the output and echo it to the human so they can confirm the match before approving. **Spell out the human's exact action — never leave them guessing.** Say "approve in the window that just opened," and if a code is shown, state which kind it is: a **visual match** to confirm against the terminal then approve in the browser (device-code flows — Vercel/Stripe/WorkOS/Render: nothing to send back), or a **paste-back** where the browser prints a code the human must enter *into the CLI* (Supabase: "Enter this verification code in Supabase CLI") — there you ask the human to relay it and you `send` it into the session. Silently polling while the human wonders what to do is a failure. **Tell them to sign into the *right account* in the browser first** — a popup approved under the wrong account still "logs in" but stores an `Unauthorized` token. When you ask for a relayed code, ask for **"the code, or any error text on the page"** — they may be staring at an error, and **you can't see screenshots** (if they send an image, say so and ask them to type it).
5. **Wait for the success line — or just the process exiting (some CLIs print none) — then verify** (whoami/status). Don't blind-`tail` the log — the identity line is usually *first* (grep for it); spinner/device-code logs flood with ANSI frames, so trust the verify command's identity *output*, not a log grep. Don't trust the verify **exit code** alone either — some (`netlify status`) exit non-zero outside a linked project even when logged in; parse stdout for the identity block. **Verify the *identity*, not just exit-0 — and don't say "logged in" until the verify command prints it.** An `Unauthorized`/empty verify *right after* a success line almost always means the human approved in the **wrong account** (or the provider hit a token cap), NOT a stale token — ask which; don't dive into a keychain/config diagnostic loop.

**Run the login once and wait — don't fire it repeatedly.** It opens a local callback server and blocks until the human approves; spawning extra logins because you didn't see instant success just leaves competing servers behind. One login, keep it alive, wait for the success line.

**Stay until the login command actually returns.** A login — especially a *poll-to-complete* step (`stripe login --complete`, `supabase login`) — isn't done until the command exits success / prints its identity line. Don't fire it and end your turn (or background it and move on): the human then approves into a dead process and nothing is saved. Block in the foreground, or `run_in_background` **and keep actively polling its output until you see completion**, before you verify.

**Keep ONE session; restarting is the last resort.** Before you stop+restart a login, confirm the session is actually dead (`noninteractive read <sess>`) — a restart orphans the human's in-flight approval and burns the code's short TTL. If it's alive (sitting at a code or retry prompt), **reuse it** — ask for a fresh code in the tab already open. Only restart when it truly exited or was never created (`Could not create CLI login session`), and when you do, **tell the human first** ("that tab's dead, a new one will open"); on success, note any leftover error tabs are safe to close. And **never go silent** — say "got it, submitting…" when a code arrives, and "still waiting on the browser" if a poll drags, so the human isn't guessing through a 12-call silence.

**Logging out for a clean state** (when a task must start logged-out): prefer `<cli> logout`; if it errors `unknown command` (Render has none), delete the config file to force logged-out (`rm ~/.render/cli.yaml`). An already-clean result is SUCCESS, not failure — `No login credentials found` (Sanity), `no profiles found` (Daytona). A logout that exits 0 (or prints `Logged out successfully`) is itself proof of clean state — go straight to login; don't spend a call on whoami/status/debug to re-confirm logged-out (that verify is for *after* login).

**If the browser login fails, read the CLI's error and follow its hint — don't blindly retry.** Many CLIs offer a device-code mode (e.g. `railway login --browserless`): it prints a URL + pairing code instead of using a localhost callback (more robust, works headless). Device-code mode usually needs a real TTY, so drive it with `npx noninteractive <cli> login --browserless` and read the URL/code from the returned `urls` array. Token env vars are the last-resort no-popup path.

**Some logins gate the popup behind an interactive prompt** ("Press any key…", an auth-method picker, a device-name field) and *fatal* when backgrounded — Railway, Heroku, Daytona, Auth0, Supabase, Convex. Drive the gate in a PTY, two ways:

- **`noninteractive`** (also auto-opens the OAuth URL it finds): `npx noninteractive <pkg> login` for an npm CLI, or `npx noninteractive start <binary> login` for a binary/brew CLI (`start` runs the literal command, not `npx`). **Then send the keystroke** — `npx noninteractive send <sess> ' '` (or `''` for Enter) — and `read --wait` for the URL/success. Just starting it leaves the login parked at the gate (the #1 failure). Flags (`--timeout`/`--no-open`) go on the `send`/`read` calls, NOT trailing the login command (they fatal `unknown flag`). It auto-opens **every** URL it prints — the auth popup is the first `…authorize…`; a later release/docs link is a junk tab. Device-code CLIs (Render) need no keystroke (the popup opens on `start`).
- **`expect`** bundles spawn + keystroke + success/error exit in one line:

```bash
expect -c 'spawn <cli> login; set timeout 180; expect {
  -nocase -re "(press enter|any key|open the browser|open browser|\(y/n\)|Login with Browser)" {send "\r"; exp_continue}
  -nocase -re "(logged in|success|authenticated)" {exit 0}
  -nocase -re "(error|denied|no token|no code)" {exit 1}
  timeout {exit 2}
  eof {catch wait r; exit [lindex $r 3]} }'
```
On `eof`, propagate the child's **real exit code** (`catch wait r; exit [lindex $r 3]`), not a blanket `exit 0` — `eof` only means the child closed its fd, so a bare `exit 0` reports success even when the CLI died non-zero with an error your narrow regex missed (`unauthorized`, `connection refused`). A clean exit (0) still passes; a non-zero exit now correctly fails. Verify with whoami/status regardless. Prefer inline `expect -c '…'` over a `.exp` file (Write won't clobber an unread scratchpad file).

**A few CLIs auto-switch to a JSON/agent mode under non-TTY instead of gating** — don't wrap these in `expect`. Stripe v1.40.9+ is the case: `stripe login --non-interactive` emits a `browser_url`+`verification_code`+`next_step` JSON and **exits 0 — that's step 1, not success**; surface the code, run the `next_step` (`stripe login --complete '<url>'`), then verify.

**OS note — validated on macOS.** The login commands, the 🟢/🔵/⌨️/🔗/🔑 classes, and the `expect`/`noninteractive` recipes are OS-agnostic. What differs on **Linux**: **install** (use each provider's docs-URL method / system package manager / `curl` installer — not `brew` or Darwin-arm64 tarballs) and **paths** — user binaries live under `~/.local/bin`, config under `~/.config` (XDG), not `/opt/homebrew/bin` or `~/Library/…`. Invoke a CLI by its name on PATH (or `command -v <cli>`); don't hard-code `/opt/homebrew`.

**Per-provider commands for 20 CLIs** (exact login command, TTY caveats, verify, token env): see [`references/providers.md`](references/providers.md). Sanity and Netlify are below.

## Sanity  (popup: auto-opens · login: automatic · verified)

- Install: `npx -y @sanity/cli@latest` (bin `sanity`; `@sanity/cli` is lighter than the `sanity` meta-pkg and has full `login`). In a Studio project, `npx sanity login` is instant.
- Login: `npx -y @sanity/cli@latest login --provider google` (swap `google`↔`github` for the human's account). **Pass `--provider`** — bare `sanity login` shows an interactive picker that hangs when backgrounded. Run it once, persistent; the popup opens (`Opening browser at <url>`) and the `localhost:4321` callback must stay reachable until approval.
- Success: exits 0 (callback server shuts down) and prints `Login successful` on current versions — but detect completion by the exit + `sanity debug` (authoritative), not by grepping the string. `logout` prints `Logged out successfully`; `No login credentials found` only shows when *already* logged out (both are clean states).
- Verify: `npx -y @sanity/cli@latest debug` → a multiline `User:` block; grep **`Email:`** for the address (the `User:` line is just a section header and has no email on it). (`sanity whoami` doesn't exist.)
- CI / no popup: `export SANITY_AUTH_TOKEN=<token>` (read before config), or `sanity login --with-token < token.txt`.

## Netlify  (popup: auto-opens · login: automatic)

- Install: `npx -y netlify-cli` (NOT `npx netlify` — that's the JS API client). Bin `netlify`/`ntl`. `NETLIFY_TELEMETRY_DISABLED=1` skips the telemetry prompt.
- Login: `npx -y netlify-cli login`. Default opens the popup and polls Netlify's API (no localhost callback to collide), stores the token, prints `You are now logged into your Netlify account!`. If a token is already cached it short-circuits `Already logged in …`. Do NOT use `--request` (the copy-a-URL ticket flow) or `BROWSER=none`.
- Verify: `npx -y netlify-cli status` → "Current Netlify User" block. It **exits 1 outside a linked project folder** even when logged in, so parse stdout for the block — don't trust the exit code.
- CI / no popup: `export NETLIFY_AUTH_TOKEN=<pat>` or `--auth <token>` on any command (PAT from app.netlify.com/user/applications).
