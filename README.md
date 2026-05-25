# infini-skill

AI Skill for Infini API integrations. Works with Claude Code, Cursor, and other assistants that support the `SKILL.md` format.

> Use natural language to generate, debug, and troubleshoot Infini merchant API integrations without repeatedly digging through the docs.

## What This Skill Does

- Generates secure Infini merchant API integration code for hosted checkout, payments, withdrawals, subscriptions, cards, and webhooks.
- Explains and implements Infini HMAC-SHA256 request signing and webhook signature verification.
- Helps troubleshoot API errors, status transitions, idempotency issues, webhook delivery, and checkout flows.
- Keeps security defaults front and center: backend-only secrets, unique `request_id`, IP allowlists, webhook idempotency, and decimal-safe amounts.

## Installation

### Claude Code

Project-level install:

```bash
git clone git@github.com:infini-money/infini-skill.git
mkdir -p .claude/skills/infini-api
cp -R infini-skill/skills/infini-api/. .claude/skills/infini-api/
```

User-level install:

```bash
git clone git@github.com:infini-money/infini-skill.git
mkdir -p ~/.claude/skills/infini-api
cp -R infini-skill/skills/infini-api/. ~/.claude/skills/infini-api/
```

### Cursor

Cursor supports the same `SKILL.md` format and can also read `.claude/skills/` for compatibility.

Project-level install:

```bash
git clone git@github.com:infini-money/infini-skill.git
mkdir -p .cursor/skills/infini-api
cp -R infini-skill/skills/infini-api/. .cursor/skills/infini-api/
```

User-level install:

```bash
git clone git@github.com:infini-money/infini-skill.git
mkdir -p ~/.cursor/skills/infini-api
cp -R infini-skill/skills/infini-api/. ~/.cursor/skills/infini-api/
```

## Example Prompts

- "Use Infini skill to set up my first hosted checkout order."
- "Generate Node.js code to sign Infini API requests."
- "Create a Python webhook handler for order.completed."
- "Query an Infini withdrawal by request_id."
- "Create and cancel an Infini subscription."
- "Explain why my Infini request returns Invalid HMAC signature."

## Resources

- Production API: `https://openapi.infini.money`
- Sandbox API: `https://openapi-sandbox.infini.money`
- Merchant API prefix: `/v1/acquiring`
- Support: `dev@infini.money`

## License

MIT
