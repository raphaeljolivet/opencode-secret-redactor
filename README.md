# opencode-secret-redactor

An [OpenCode](https://opencode.ai) plugin that prevents secrets from leaking
into LLM context. Secrets detected in tool output are replaced with
`🔒label🔓` tokens before the model sees them, then transparently
restored when a tool needs the real value for execution.

> **Fork note:** the upstream package (0.5.1) does not load on opencode >= 1.18
> (`"Plugin export is not a function"` — see
> [opencode #31575](https://github.com/anomalyco/opencode/issues/31575)).
> This fork fixes the packaging. The fixed build is published on npm as
> **`opencode-secret-redactor-fixed`**.

## Detected secret types

AWS keys, GitHub/GitLab tokens, OpenAI/Anthropic keys, Google Cloud
credentials, Stripe keys, Slack tokens, JWTs, private keys, database
connection strings, and many more. See
[`src/patterns.ts`](src/patterns.ts) for the full list.

## Setup

Add the plugin to your `opencode.json`:

```json
{
  "plugin": ["opencode-secret-redactor-fixed"]
}
```

or for the original (broken-on-1.18) package:

```json
{
  "plugin": ["opencode-secret-redactor@0.5.1"]
}
```

The plugin hooks into tool execution automatically -- no configuration is
required to start.

## Configuration

Everything is optional. Without any config file the plugin scans with the
default pattern set, ignoring values that look like placeholders.

### Config files

Read in this order; later layers win:

1. `secret-redactor.json` in your opencode config directory
   (`~/.config/opencode`, or `$OPENCODE_CONFIG_DIR` if set)
2. `.opencode/secret-redactor.json` in the current project directory
3. The file pointed to by `$SECRET_REDACTOR_CONFIG` -- when set, it
   **replaces** layers 1 and 2, which keeps tests and one-off runs hermetic
4. The environment lists `$SECRET_REDACTOR_ENABLE` and
   `$SECRET_REDACTOR_DISABLE` (comma-separated pattern labels)

Every layer uses the same schema:

```json
{
  "enable": ["email"],
  "disable": ["jwt"],
  "minLength": 12,
  "extraPatterns": [
    { "label": "acme_token", "pattern": "acme_[a-f0-9]{32}", "flags": "g" }
  ]
}
```

- `enable` / `disable`: pattern labels from [`src/patterns.ts`](src/patterns.ts).
  `disable` wins when a label appears in both. Labels only matter for patterns
  that are off by default (such as `email`) or to drop an over-eager pattern.
- `minLength`: shortest value worth redacting (default `8`).
- `extraPatterns`: your own regular expressions, appended to the built-in set.
  A pattern that fails to compile is skipped with a warning.

Malformed JSON and wrong-typed fields are ignored with a warning rather than
failing the plugin.

### Off-by-default patterns

PII patterns ship disabled, because redacting them costs more than it saves.
The current one is `email`; enable it explicitly:

```json
{ "enable": ["email"] }
```

### Placeholders are never secrets

Values that are references to a secret rather than the secret itself are
skipped: `$PASS`, `${DB_PASSWORD}`, `{some_pass}`, `{{vault_password}}`,
`%DEPLOY_TOKEN%`, `<<ACCESS_KEY>>`, `$(PASS)`, `__SECRET__`,
`${{ secrets.FOO }}`. The check matches the **whole** value, so a real secret
that merely contains `$` or braces (a bcrypt hash, for example) is still
redacted.

### Programmatic use

The package also exports `loadConfig`, `resolveConfig`, `configure`,
`getConfig`, `detectSecrets`, and `isPlaceholder`, so secrets can be scanned
outside opencode:

```js
const { loadConfig, configure, detectSecrets } = require("opencode-secret-redactor-fixed");

configure(loadConfig());
const hits = detectSecrets(fs.readFileSync("settings.yml", "utf8"));
```

## How it works

1. **After** `bash` or `read` tool output, the plugin scans for secrets using
   pattern matching and stores any matches in an in-memory vault.
2. The output sent to the LLM contains only redacted placeholders.
3. **Before** `bash`, `write`, or `edit` tool execution, placeholders in the
   tool arguments are replaced with the original values so commands run
   correctly.

## What was fixed

- The default export is now the `{ id, server }` object form, which opencode's
  loader resolves via `readV1Plugin` instead of the legacy `getLegacyPlugins`
  path (the source of the load failure).
- A new `exports["./server"]` entry points opencode to the ESM build
  (`dist/plugin.js`), avoiding the CJS interop object that broke loading.

## License

[MIT](LICENSE)
