# Pierre skills

Agent skills for the Pierre HTTP API, SDKs, and CLI.

## Install

```sh
npx skills add pierrecomputer/skills
```

Select `pierre`, then reload your agent. The skill reads the current [Pierre docs](https://code.storage/docs/llms.txt), which hold the examples for each language.

The skill covers Code Storage repositories through the HTTP API, the TypeScript, Python, and Go SDKs, and the CLI. It checks the installed package and CLI version before it applies the catalog naming rules.

To move code from the old Code Storage SDKs (`@pierre/storage`, `pierre-storage`, `code-storage-go`) or from deprecated HTTP routes, also select `pierre-migrate`. It holds the changes for the move, and the questions that the agent asks you.

## Develop

The skill is in `skills/pierre/SKILL.md`.

Keep API details and code examples in the docs. Keep only lookup instructions and mapping rules in the skill, so that it changes only when a mapping rule changes.

`skills/pierre-migrate/` holds the move from the old SDKs and routes. It holds the facts of that move itself, because it maps one old version to one new version. Its references are split by concern: `http.md`, `names.md`, `methods.md` (each old method), `sdk.md` (rules for every call), and one file for each language. Compile each example against the SDK that the skill targets before you publish a change.
