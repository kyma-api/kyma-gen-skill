# kyma-gen-skill

Public registry of Claude Code / Cursor / Codex skills that orchestrate the
[Kyma API](https://kymaapi.com) media generation surface via the
`@kyma-api/gen` CLI.

## Browse

The kymaapi.com search API is the recommended entrypoint:

```sh
curl https://kymaapi.com/skills?q=
```

Or fetch the manifest directly:

```sh
curl https://raw.githubusercontent.com/kyma-api/kyma-gen-skill/refs/heads/main/skills/index.json
```

## Install a skill

```sh
kyma-gen skills install <name>
```

The CLI fetches `skills/<name>/SKILL.md` plus any sibling files declared in
`index.json`, verifies sha256 (when published), and merges them into the
target adapter (Claude Code, Cursor, AGENTS.md).

## Layout

```
skills/
├── index.json                  ← required, registry manifest (version: 1)
└── <skill-name>/
    ├── SKILL.md                ← required, skill body with frontmatter
    └── ...                     ← optional supporting files
```

`index.json` shape per skill:

```json
{
  "version": 1,
  "skills": [
    {
      "name": "<skill-name>",
      "description": "<one-line>",
      "files": [
        { "path": "SKILL.md", "sha256": "<64-hex>", "bytes": 4382 }
      ]
    }
  ]
}
```

## Contributing

Open a PR with your skill folder under `skills/<name>/` and an updated
`index.json` entry. SHA-256 fields are soft-validated for v0–v1.0 (warning
if absent or malformed) and hard-validated for mismatch — wrong hash always
aborts install. v1.1 will hard-fail the missing-hash case too.

## License

Released under [MIT](./LICENSE). Each individual skill may carry its own
license declared in its SKILL.md frontmatter.
