# Per-provider CLI login reference

Verified by probing each CLI (help + source + shadowed-`open` tests). Sanity was completed end-to-end live; the rest are mechanism-verified (no real login performed). Method is in `SKILL.md`; this is the lookup table.

**Legend**
- 🟢 default `login` auto-opens the popup **and** stores the token itself — the happy path.
- ⌨️ **needs a real TTY** — bare/backgrounded login hits an interactive prompt and *fatals*. Run it in the foreground of a real terminal, or drive it with `npx noninteractive <cli> <login-args>` and answer the prompt.
- 🔗 **prints the URL only** (no auto-open) — you must surface the URL to the human yourself.
- `token` = the env var that skips the popup entirely (CI / headless / no-popup fallback).

| Provider | Light install | Login command | Verify | Token env (CI) |
|---|---|---|---|---|
| Sanity 🟢 | `npx -y @sanity/cli@latest` | `sanity login --provider google` ⌨️*(picker → pass `--provider`)* | `sanity debug` → `User:` | `SANITY_AUTH_TOKEN` |
| Netlify 🟢 | `npx -y netlify-cli` | `netlify login` *(polls API, no localhost server)* | `netlify status` | `NETLIFY_AUTH_TOKEN` |
| Vercel 🟢 | `npx -y vercel` | `vercel login` *(unset `CI` — `CI=1` suppresses the popup)* | `vercel whoami` | `VERCEL_TOKEN` |
| Cloudflare 🟢 | `npx -y wrangler@latest` | `wrangler login` *(localhost:8976 callback)* | `wrangler whoami` | `CLOUDFLARE_API_TOKEN` (+`CLOUDFLARE_ACCOUNT_ID`) |
| Modal 🟢 | `pip install modal` | `modal setup` *(NOT `modal login`)* | `modal token info` | `MODAL_TOKEN_ID`+`MODAL_TOKEN_SECRET` |
| E2B 🟢 | `npx -y @e2b/cli` | `e2b auth login` *(short-circuits if already logged in)* | `e2b auth info` | `E2B_API_KEY` *(`E2B_ACCESS_TOKEN` is deprecated — retired Aug 2026)* |
| WorkOS 🟢 | `npx -y workos@latest` | `workos auth login` *(NOT `workos login`)* | `workos auth status` | `WORKOS_API_KEY` |
| Fly.io 🟢 | `brew install flyctl` / curl | `fly auth login` *(remote poll)* | `fly auth whoami` | `FLY_API_TOKEN` |
| Render 🟢 | `brew install render` / binary | `render login` *(remote poll, works headless)* | `render whoami` | `RENDER_API_KEY` |
| Neon 🟢 | `npx -y neonctl` | `neonctl auth` *(CI var blocks it unless `--force-auth`)* | `neonctl me` | `NEON_API_KEY` |
| Turso 🟢 | curl `get.tur.so` / tap binary | `turso auth login` *(unset `TURSO_API_TOKEN` first)* | `turso auth whoami` | `TURSO_API_TOKEN` |
| Clerk 🟢 | `npx -y clerk` | `clerk auth login` *(does NOT print URL — popup only)* | `clerk whoami` | `CLERK_SECRET_KEY` |
| Railway 🟢⌨️ | `npm i -g @railway/cli` | `railway login` *(prompts `Open browser? Y/n`; non-TTY errors)* | `railway whoami` | `RAILWAY_API_TOKEN` |
| Heroku 🟢⌨️ | `npm i -g heroku` | `heroku login` *(`Press any key…` raw-mode; unset `HEROKU_API_KEY`)* | `heroku whoami` | `HEROKU_API_KEY` |
| Daytona 🟢⌨️ | binary / `brew` *(no npm pkg)* | `daytona login` *(TUI auth-method picker → Enter on "Login with Browser")* | `daytona organization list` | `DAYTONA_API_KEY` (or `login --api-key`) |
| Convex 🟢⌨️ | `npx -y convex` | `convex login --device-name <name>` *(`--device-name` skips the prompt)* | `convex login status` | `CONVEX_DEPLOY_KEY` |
| Supabase ⌨️ | `npx -y supabase` / brew tap | `supabase login` *(opens browser to mint a token, then prompts `Enter your access token` to paste it back; `--token` / `SUPABASE_ACCESS_TOKEN` skips the prompt)* | `supabase projects list` | `SUPABASE_ACCESS_TOKEN` |
| Auth0 🔵⌨️ | `brew install auth0/auth0-cli/auth0` | `auth0 login` *(survey "As a user/machine" + `Press Enter`; match the user code)* | `auth0 tenants list` | — *(no env token; CI = machine login `auth0 login --client-id … --client-secret … --domain …`)* |
| Stripe 🔵 | `brew install stripe/stripe-cli/stripe` | `stripe login` *(opens browser; confirm the pairing code it displays)* | `stripe config --list` | `STRIPE_API_KEY` / `--api-key` |
| Stack Auth 🔗 | `npx -y @stackframe/stack-cli` | `stack login` *(prints URL, polls; NOT `@stackframe/init-stack`)* | `stack whoami` | `STACK_CLI_REFRESH_TOKEN` |

## The ⌨️ TTY-gated logins (where a bare/backgrounded run fatals)

These show an interactive prompt *before* the popup, so a non-TTY run errors out. Run them in a real terminal foreground, or via `npx noninteractive <cli> <args>` and send the keystroke:

- **Railway** — `? Open the browser? (Y/n)`; without a TTY: `Cannot login in non-interactive mode`. (`railway login --browserless` switches to a pairing-code copy flow but still needs a TTY.)
- **Heroku** — `Press any key to open up the browser to login or q to exit` (raw-mode `stdin`); piped stdin throws.
- **Daytona** — TUI `Select Authentication Method` → `Login with Browser` (press Enter).
- **Auth0** — survey `How would you like to authenticate? As a user / As a machine`, then `Press Enter to open the browser`.
- **Supabase** — `Press Enter to open browser`, then after auth `Enter your verification code:` (paste the code the browser shows).
- **Convex** — `Device name:` prompt (skip with `--device-name <name>`).
- **Sanity** — only the provider picker is interactive; `--provider google|github` skips it and the rest is hands-free.

## Common popup-suppressors to NOT set

`CI=1` (Vercel, WorkOS, Neon → URL prints but no popup) · `BROWSER=none` / bogus `$BROWSER` · `--no-open` / `--no-browser` / `--headless` / `--browserless` / `--non-interactive` · a token env var already set (most CLIs skip login when their `*_TOKEN`/`*_API_KEY` is present — unset it to force the popup).
