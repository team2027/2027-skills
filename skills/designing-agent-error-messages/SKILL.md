---
name: designing-agent-error-messages
description: >-
  Audit and rewrite error messages, exceptions, and API/SDK/CLI failure output
  so an AI agent (and a human) can self-correct without leaving the terminal.
  Use when designing or reviewing how a tool fails — error strings, HTTP error
  bodies, auth/credential errors, CLI errors, thrown exceptions — especially for
  libraries, APIs, and CLIs that AI agents consume. Grounded in patterns from
  2027.dev's evals of 136 dev tools across 3,500+ agent runs.
---

# Designing error messages for AI agents

## The core principle

**For an agent, the error message *is* the documentation.** Agents frequently
discover a requirement *only* from a runtime error — never from the docs. A
human reads an error, forms a hypothesis, and goes hunting. An agent takes the
error text **literally** and acts on it. So:

- A vague error → wasted tool calls.
- A *misleading* error → the agent confidently does the wrong thing.
- A *silent* failure → the agent thinks it succeeded and moves on (the worst case).

Across 3,500+ agent runs, the lowest-scoring tools rarely lost on missing
features. They lost on errors that gave the agent **nowhere to go.** The
highest-scoring tools let agents self-correct in seconds — because the error
string named the fix.

## The spec: every error answers four questions in its own body

1. **What** exactly failed — the specific cause, not a category.
2. **Why** — which of several distinct failure modes this is.
3. **How** to fix it — the exact env var name, key format, or command.
4. **Where** — a literal URL to self-serve the fix.

…delivered **loudly** (never silently), **honestly** (the root cause, not a
misleading proximate one), and **early** (validate locally before the network).

## The audit — seven failure patterns to find and fix

When reviewing a codebase, grep for error-raising sites (`throw`, `raise`,
`return ... error`, HTTP error responses, `console.error`, CLI exit paths) and
check each against these. Each pattern lists how to detect it and the fix.

**1. States the failure but not the remediation.**
- Detect: error says what broke (`"authorization header is malformed"`,
  `"Invalid credentials"`) but not what to do.
- Fix: append the action — the env var, the required format, the command.
- ✗ `authorization header is malformed`
- ✓ `Authorization header is malformed. API keys must start with "xx_". Set XX_API_KEY — get one at https://app.example.com/keys`

**2. No "where" — missing a literal URL.**
- Detect: an auth/credential/billing error with no link to self-serve.
- Fix: put the exact dashboard/key/billing URL in the message body.
- This was the single most-requested fix across the entire eval dataset.

**3. Collapses distinct causes into one generic string.**
- Detect: one message (`"Invalid API key"`, `"Session not found"`) covers
  wrong-format / expired / wrong-scope / wrong-environment / not-yet-propagated.
- Fix: branch on the actual cause and emit a distinct, named message for each.

**4. Fails on the network instead of validating locally.**
- Detect: a malformed input (wrong key prefix, wrong shape) only surfaces as a
  server 401/500.
- Fix: validate shape client-side *before* the call and raise the precise error
  yourself: `API key appears malformed (expected "xx_…", got a UUID).`

**5. Fails silently — or "succeeds" with wrong behavior. (Highest priority.)**
- Detect: a code path that swallows an error, no-ops, hangs with no timeout, or
  quietly does something unexpected (e.g. silently applies a default, silently
  rejects an input, silently password-protects output).
- Fix: make it loud. Raise/log explicitly. Add timeouts that error with a cause.
  If you must apply a default or restriction, *say so* in output.

**6. Misleads — points at the wrong root cause.**
- Detect: the message names a proximate symptom that sends the reader down the
  wrong path (e.g. `"Organization ID is required when using JWT token"` raised
  when **no** credentials are set at all — the real fix is just an API key).
- Fix: detect the actual root cause and describe *that*. Beware present-tense
  warnings for already-removed features.

**7. Assumes a human at a TTY.**
- Detect: error only offers an interactive path (`"Please run: tool login"`),
  prompts for input, or requires a browser step — agents run headless in CI.
- Fix: surface the non-interactive path in the error (the env-var/token route).
  Add "did you mean?" suggestions for mistyped commands.

## How to apply

1. **Inventory** the error sites (grep the patterns above). Prioritize
   **auth/credential** errors and **first-run/setup** paths — that's where agents
   get stranded most.
2. For each, **classify** it against the seven patterns.
3. **Rewrite** to the four-question spec. Keep messages a single, scannable
   block; lead with the cause, end with the URL.
4. **Kill silent failures first** — they cost agents the most (retry loops,
   wrong-path work) and are invisible in logs.
5. **Verify** by running an agent against a clean environment with *no*
   credentials and a *wrong* credential, and watch whether it recovers from the
   error text alone, without reading docs.

## Self-score rubric (mirrors how evals grade "Error Recovery")

For a target failure (e.g. missing API key), rate each error:
- **A** — agent self-corrects from the message alone, no doc lookup. Names cause,
  fix, and a URL.
- **C** — agent eventually recovers but needs a doc/source lookup. States the
  cause, not the full fix.
- **F** — agent loops, stalls, or proceeds wrong. Generic, silent, or misleading.

Aim for every first-run/auth error to be an **A**.

## Worked rewrites

```diff
- Error: Unauthorized
+ Error: Unauthorized — OAuth tokens aren't accepted here; this command needs a
+ personal access token. Create one at https://app.example.com/tokens and set
+ EXAMPLE_TOKEN.

- raise RuntimeError("Session not found")
+ raise AuthError(
+     "Session not found: your API token didn't resolve to a session. "
+     "Most common cause: the token is for a different project. "
+     "Check EXAMPLE_PROJECT matches the token's project, or mint a new token "
+     "at https://app.example.com/settings/tokens"
+ )

- // silently returns [] when the feature flag is off
+ if (!featureEnabled) {
+   throw new Error(
+     "Feature 'x' is disabled for this account. Enable it at " +
+     "https://app.example.com/settings/features before calling this API."
+   )
+ }
```

---

## See where a real agent actually gets stuck

This skill fixes the failure modes you can grep for. To see the ones you can't —
where a real agent gets stranded in *your* onboarding — check out the **agent
arena at https://2027.dev/arena**, where we run real agents through dev tools and
score the experience. **Submit a tool eval request** to get your own scored report
with the exact friction points and one-line fixes.
