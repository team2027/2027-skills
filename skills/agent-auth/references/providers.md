# Per-provider CLI login reference

Verified by probing each CLI (help + source + shadowed-`open` tests). Sanity was completed end-to-end live; the rest are mechanism-verified (no real login performed). Method is in `SKILL.md`; this is the lookup table.

**Legend**
- 🟢 default `login` auto-opens the popup **and** stores the token itself — the happy path.
- 🔵 **device-code** — shows a `user_code`/URL to confirm in the browser (no localhost callback); surface the code to the human before they approve.
- ⌨️ **needs a real TTY** — bare/backgrounded login hits an interactive prompt and *fatals*. Drive it in a real-terminal foreground, with `npx noninteractive` (`<pkg>` for npm CLIs, `start <binary>` for binaries — then `send` the keystroke), or `expect`.
- 🔗 **prints the URL only** (no auto-open) — you must surface the URL to the human yourself.
- `token` = the env var that skips the popup entirely (CI / headless / no-popup fallback).

| Provider | Light install | Login command | Verify | Token env (CI) |
|---|---|---|---|---|
| Sanity 🟢 | `npx -y @sanity/cli@latest` | `sanity login --provider google` ⌨️*(picker → pass `--provider`)* | `sanity debug` → grep `Email:` *(multiline block; `User:` is just a header)* | `SANITY_AUTH_TOKEN` |
| Netlify 🟢 | `npx -y netlify-cli` | `netlify login` *(polls API, no localhost server)* | `netlify status` *(exits 1 outside a linked project even when logged in — parse stdout; `--json` dodges the ANSI color codes in the plain output)* | `NETLIFY_AUTH_TOKEN` |
| Vercel 🔵 | `npx -y vercel` | `vercel login` *(device-code: auto-opens `…/oauth/device?user_code=XXXX`; show the human the code; unset `CI`)* | `vercel whoami` | `VERCEL_TOKEN` |
| Cloudflare 🟢 | `npx -y wrangler@latest` | `wrangler login` *(localhost:8976 callback; prints `Successfully logged in.`)* | `wrangler whoami` | `CLOUDFLARE_API_TOKEN` (+`CLOUDFLARE_ACCOUNT_ID`) |
| Modal 🟢 | `pip install modal` | `modal setup` *(NOT `modal login`)* | `modal token info` | `MODAL_TOKEN_ID`+`MODAL_TOKEN_SECRET` |
| E2B 🟢 | `npx -y @e2b/cli` | `e2b auth login` *(short-circuits if already logged in)* | `e2b auth info` | `E2B_API_KEY` *(`E2B_ACCESS_TOKEN` is deprecated — retired Aug 2026)* |
| WorkOS 🔵 | `npx -y workos@latest` | `workos auth login` *(NOT `workos login`; device-code: opens `signin.workos.com/device` AND prints `Enter code: XXXX-XXXX` — surface it; remote poll, no localhost callback)* | `workos auth status` | `WORKOS_API_KEY` |
| Fly.io 🟢 | `brew install flyctl` / curl *(no npm pkg)* | `fly auth login` *(remote poll, no localhost callback → skip the port-collision check; auto-opens browser)* | `fly auth whoami` *(every `fly` cmd prints `Warning: Metrics token unavailable … context canceled` — non-fatal noise; trust the exit code + `successfully logged in as <email>`, not output cleanliness)* | `FLY_API_TOKEN` |
| Render 🔵⌨️ | `brew install render` / binary *(no npm pkg. brew's `render-oss` tap is untrusted + stuck at v1.x: if `brew upgrade render` errors `Refusing to load formula … from untrusted tap`, run `brew trust render-oss/render`; if it still shows v1.x, download v2.x from github.com/render-oss/cli/releases/latest — asset `cli_<ver>_darwin_arm64.zip` (pattern `cli_<ver>_<os>_<arch>.zip`, NOT `render-darwin-arm64`), unzip + chmod +x. v1.x logs in fine but nags; v2.x is nag-free)* | `render login -o text` *(device-code: prints `Complete login … with code: XXXX-XXXX-XXXX-XXXX` + opens `dashboard.render.com/device-authorization/<code>` — surface the code. **v1.x** plain backgrounded `render login` PANICS on the post-login workspace picker (`could not open /dev/tty`) AFTER saving the token — `-o text` skips it; **v2.x auto-switches to text on non-TTY** so the panic is gone (`-o text` stays harmless). Never retry on the panic, just verify)* | `render whoami` | `RENDER_API_KEY` |
| Neon 🟢 | `npx -y neonctl` | `neonctl auth` *(CI var blocks it unless `--force-auth`; prints `INFO: Auth complete`)* | `neonctl me` | `NEON_API_KEY` |
| Turso 🟢 | `brew install tursodatabase/tap/turso` / `curl get.tur.so/install.sh` *(full tap formula; bare `brew tap` is incomplete)* | `turso auth login` *(unset `TURSO_API_TOKEN` first)* | `turso auth whoami` | `TURSO_API_TOKEN` |
| Clerk 🟢 | `npx -y clerk` | `clerk auth login` *(does NOT print URL — popup only)* | `clerk whoami` | — *(no env-only CLI login — `clerk auth login` stores a token in the OS keychain; `CLERK_SECRET_KEY` is the app's server-side key and does NOT authenticate the CLI)* |
| Railway 🟢⌨️ | `npm i -g @railway/cli` *(need **v5+** — a brew/global at v5+ is fine; reinstall `--foreground-scripts` if `railway --version` is blank/<5: postinstall fetches the binary, skipped under `ignore-scripts`. Old v4.x's browser callback is flaky)* | `railway login` *(TTY → `expect`); **if it returns `No token received`, retry `railway login --browserless`** (pairing code)* | `railway whoami` | `RAILWAY_API_TOKEN` |
| Heroku 🟢⌨️ | `npx -y heroku@latest` *(`npm i -g` lands off non-interactive PATH)* | `npx noninteractive heroku login` *(`Press any key…` raw-mode → send a key; unset `HEROKU_API_KEY`; prints `Logged in as <email>`)* | `heroku whoami` | `HEROKU_API_KEY` |
| Daytona 🟢⌨️ | `brew install daytonaio/cli/daytona` / binary *(full tap formula — bare `brew install daytona` fails; no npm pkg → `npx noninteractive start daytona …` or `expect`)* | `daytona login` *(TUI picker → Enter on "Login with Browser"; prints `Successfully logged in!`)* | `daytona organization list` | `DAYTONA_API_KEY` (or `login --api-key`) |
| Convex 🔵⌨️ | `npx -y convex` | `convex login --device-name <name>` *(device-code: prints `…/device?user_code=XXXX`. `--device-name` skips only the FIRST prompt; a second `Open the browser? (Y/n)` gate ALSO needs a TTY — `--device-name` alone still fatals non-interactive, so drive it with `expect`/`noninteractive`)* | `convex login status` *(shows teams, not email)* | `CONVEX_DEPLOY_KEY` |
| Supabase ⌨️ | `npx -y supabase` / `brew install supabase/tap/supabase` | `supabase login` *(opens browser to mint a token, then prompts `Enter your access token` to paste it back; `--token` / `SUPABASE_ACCESS_TOKEN` skips the prompt)* | `supabase projects list` | `SUPABASE_ACCESS_TOKEN` |
| Auth0 🔵⌨️ | `brew install auth0/auth0-cli/auth0` | `auth0 login` *(survey "As a user/machine" + `Press Enter`; match the user code)* | `auth0 tenants list` | — *(no env token; CI = machine login `auth0 login --client-id … --client-secret … --domain …`)* |
| Stripe 🔵⌨️ | `brew install stripe/stripe-cli/stripe` | `stripe login` *(prints a pairing code, then `Press Enter to open the browser` — drive the Enter in a PTY (`expect`/`noninteractive`); confirm the pairing code matches)* | `stripe get /v1/account` *(authenticated API call — `config --list` only echoes local config and passes even when logged-out)* | `STRIPE_API_KEY` / `--api-key` |
| Stack Auth 🔗 | `npx -y @stackframe/stack-cli` | `stack login` *(prints URL, polls; NOT `@stackframe/init-stack`)* | `stack whoami` | `STACK_CLI_REFRESH_TOKEN` |

## The ⌨️ TTY-gated logins (where a bare/backgrounded run fatals)

These show an interactive prompt *before* the popup, so a non-TTY run errors out. Run them in a real-terminal foreground, via `npx noninteractive` (`<pkg>` or `start <binary>`) or `expect`, sending the keystroke:

- **Railway** — needs a TTY (non-TTY: `Cannot login in non-interactive mode`); v5.x auto-opens under a PTY with no keystroke (old v4.x prompts `Open the browser? (Y/n)`). Its browser callback is **flaky** — if `railway login` ends with `No token received` (web page says "try `--browserless`"), retry `railway login --browserless` (pairing code; also needs a TTY).
- **Heroku** — `Press any key to open up the browser to login or q to exit` (raw-mode `stdin`); piped stdin throws.
- **Daytona** — TUI `Select Authentication Method` → `Login with Browser` (press Enter).
- **Render** *(v1.x only)* — default `--output interactive` runs a post-login workspace picker that fatals non-TTY (`panic: could not open /dev/tty`); the token is saved BEFORE the panic, so a backgrounded `render login` still succeeds — pass `-o text` to skip it, then confirm the printed device code. **v2.x auto-switches to text on non-TTY**, so no panic (the `-o text` flag stays harmless).
- **Auth0** — survey `How would you like to authenticate? As a user / As a machine`, then `Press Enter to open the browser`.
- **Stripe** — prints a pairing code, then `Press Enter to open the browser` gates the popup; device-code, so confirm the displayed pairing code before approving.
- **Supabase** — `Press Enter to open browser`, then `Enter your access token` (paste the token the browser page mints; or skip with `--token`/`SUPABASE_ACCESS_TOKEN`).
- **Convex** — TWO gates: a `Device name:` prompt (skip with `--device-name <name>`) **and** a following `Open the browser? (Y/n)` prompt that also needs a TTY, so `--device-name` alone still fatals non-interactive. Drive the `Open the browser?` answer with `expect`/`noninteractive` — the generic `expect` recipe's `open the browser` match already handles it.
- **Sanity** — only the provider picker is interactive; `--provider google|github` skips it and the rest is hands-free.

## Common popup-suppressors to NOT set

`CI=1` (Vercel, WorkOS, Neon → URL prints but no popup) · `BROWSER=none` / bogus `$BROWSER` · `--no-open` / `--no-browser` / `--headless` / `--browserless` / `--non-interactive` · a token env var already set (most CLIs skip login when their `*_TOKEN`/`*_API_KEY` is present — unset it to force the popup). Stored creds in the CLI's *config file* (not just env) can also short-circuit login or block verify (e.g. Daytona's `config.json` `api.key`) — clear those too if a login mysteriously no-ops. Daytona's config is `~/Library/Application Support/daytona/config.json` (macOS) / `~/.config/daytona/config.json`; token = the profile's `api.token` (NOT the keychain), and `daytona logout` clears the token but leaves a stale `api.url` — a leftover `http://localhost:3000` makes login *succeed* yet `daytona organization list` fail `connection refused` (delete the config to reset).

**Exception — Turso `--headless`:** the blanket `--headless` ban above is the rule for auto-open CLIs, but Turso *documents* `turso auth login --headless` as its WSL/remote/SSH fallback — it prints the auth URL to open manually instead of trying (and hanging on) a browser auto-open. In exactly the headless/agent contexts this skill targets, prefer it over fighting the auto-open.

## Install docs (fresh-machine links)

Verified official install pages — reach for these when a CLI isn't already on the box. The table's `npx`/`pip` forms install-on-run; the `brew`/`curl` binaries (Fly, Render, Turso, Daytona, Auth0, Stripe, Heroku, Supabase) need a real install first.

- Sanity — https://www.sanity.io/docs/apis-and-sdks/cli
- Netlify — https://docs.netlify.com/api-and-cli-guides/cli-guides/get-started-with-cli/
- Vercel — https://vercel.com/docs/cli
- Cloudflare — https://developers.cloudflare.com/workers/wrangler/install-and-update/
- Modal — https://modal.com/docs/guide
- E2B — https://e2b.dev/docs/cli
- WorkOS — https://workos.com/docs/authkit/cli-installer
- Fly.io — https://fly.io/docs/flyctl/install/
- Render — https://render.com/docs/cli
- Neon — https://neon.com/docs/reference/cli-install
- Turso — https://docs.turso.tech/cli/installation
- Clerk — https://clerk.com/docs/cli
- Railway — https://docs.railway.com/cli
- Heroku — https://devcenter.heroku.com/articles/heroku-cli
- Daytona — https://www.daytona.io/docs/en/tools/cli/
- Convex — https://docs.convex.dev/cli
- Supabase — https://supabase.com/docs/guides/local-development/cli/getting-started
- Auth0 — https://auth0.com/docs/deploy-monitor/auth0-cli
- Stripe — https://docs.stripe.com/stripe-cli/install
- Stack Auth — https://docs.stack-auth.com/docs/js/getting-started/setup
