## Setup
- Node ≥ 20, pnpm; `pnpm i`
- Test: `pnpm test`
- Lint/format: `pnpm lint && pnpm format`
- TypeScript strict; prefer pure functions; single quotes; no semicolons
- Do **not** enable network access unless allow-listed domains are required; default to offline. :contentReference[oaicite:2]{index=2}

## Verification
- CI must pass: tests + lint
- On each PR, include: scope, risks, test coverage summary

## PR Conventions
- Title: `[feat|fix|chore]: <scope> – <summary>`
- Keep commits small; prefer minimal diffs

## Security
- Secrets via env vars only; never hardcode
- If Cloud internet is needed, allow-list *only* required registries/hosts (e.g., npm registry, GitHub, Neon) and restrict HTTP verbs to GET/HEAD/OPTIONS. :contentReference[oaicite:3]{index=3}

---
