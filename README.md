# 2027 Skills

Open-source [Claude Code](https://claude.com/claude-code) skills for building software that **AI agents** can actually use — by the team behind [2027.dev](https://2027.dev), where we run real agents through 100+ dev tools and score their experience.

Each skill is a self-contained folder under [`skills/`](./skills). Drop one into your project and your agent will use it.

## Skills

| Skill | What it does |
|---|---|
| [`designing-agent-error-messages`](./skills/designing-agent-error-messages) | Audit and rewrite your error messages, exceptions, and API/SDK/CLI failures so an agent can self-correct from the message alone. Grounded in patterns from 3,500+ agent runs across 136 dev tools. |
| [`agent-auth`](./skills/agent-auth) | Log a real human into a dev tool's CLI by auto-opening the OAuth browser popup and letting the login complete itself — no token copy-paste, no browser automation. Tested recipes for Sanity and Netlify. |

_More coming — discoverability, agent-friendly docs, SDK design._

## Install

**Project scope** (this repo only):
```bash
git clone https://github.com/team2027/2027-skills.git /tmp/2027-skills
mkdir -p .claude/skills
cp -r /tmp/2027-skills/skills/designing-agent-error-messages .claude/skills/
```

**Personal scope** (all your projects):
```bash
git clone https://github.com/team2027/2027-skills.git /tmp/2027-skills
cp -r /tmp/2027-skills/skills/designing-agent-error-messages ~/.claude/skills/
```

Then in Claude Code, the skill loads automatically when you're working on error handling — or invoke it directly with `/designing-agent-error-messages`.

## See where a real agent gets stuck in your product

These skills fix what you can grep for. To see where a real agent actually gets stranded in *your* onboarding — and how you rank against 135 other dev tools — check out the **[agent arena](https://2027.dev/arena)** and **submit a tool eval request**.

## License

MIT © [2027.dev](https://2027.dev)
