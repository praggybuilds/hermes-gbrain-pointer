[MODEL: claude-opus-5-5] [AGENT: Murphy] RESPONDING ON BEHALF OF PRAGGY

## Add `gbrain-pointer` to the plugin catalog

Adds `plugin-catalog/gbrain-pointer.yaml`, which pins
[`praggybuilds/hermes-gbrain-pointer`](https://github.com/praggybuilds/hermes-gbrain-pointer) at
`<PIN_AFTER_PUSH>` (version `0.1.0`).

### What it does

`gbrain-pointer` is a read-only `memory.provider` for [GBrain](https://github.com/garrytan/gbrain).
After each non-trivial user turn it calls GBrain's MCP `search` tool on a background thread. On the
next turn `prefetch()` returns up to 3 lines like `[source:slug] title`, under a header that tells
the model to treat them as fallible background context, not instructions. It returns page titles
and slugs only, never page text, and it never writes to GBrain.

### Hermes surfaces used

- `MemoryProvider` ABC through `register(ctx)` → `ctx.register_memory_provider(...)`. The manifest
  is detected as `kind: exclusive` and selected with `memory.provider: gbrain-pointer`.
- `is_available`, `unavailable_reason`, `initialize`, `get_tool_schemas` (returns `[]`),
  `get_config_schema` (env-var fields only, so `save_config` keeps the default no-op), `prefetch`,
  `queue_prefetch`, `recall_status`, `sync_turn` (no-op), `on_session_switch` and `shutdown`.
- `agent.secret_scope.get_secret` for credentials, `agent.memory_provider.spawn_context_thread` for
  the background search, and `is_trivial_prompt` to skip greetings, acknowledgements and slash
  commands.
- No tools, hooks, middleware, CLI commands, skills, Desktop or dashboard parts, and no Python
  dependencies (standard library `urllib` only).

### Disclosures (rule 13)

- **Network calls.** Only to the GBrain server the user configures (`GBRAIN_MCP_URL`, default
  `http://127.0.0.1:3131/mcp`, the `gbrain serve --http` default):
  - `POST <origin>/token` with `grant_type=client_credentials`, the client id and the client
    secret. This happens on the first search and once more after a `401`.
  - `POST <GBRAIN_MCP_URL>` with a JSON-RPC `tools/call search` after each non-trivial user turn.
    It carries the first 500 characters of the user's message, the result limit,
    `snippet_chars: 0` and an optional `source_id`.
  - No assistant text, tool output, files or message history is sent. Plain `http` to a
    non-loopback host is refused unless the user sets `GBRAIN_POINTER_ALLOW_HTTP=1`.
- **Third-party services.** None. GBrain is self-hosted by the user; this plugin contacts nothing
  else.
- **Reads outside its own data.** None. It reads only its own `GBRAIN_*` variables, through the
  profile secret scope. It reads no other tool's token files or browser profiles.
- **Stored credentials (rule 11).** None written. The user's own GBrain OAuth client secret comes
  from the profile `.env`. Minted access tokens are kept in memory per (token URL, client id) and
  never written to `os.environ`, disk or logs. The plugin mints tokens only for its own client
  (`client_credentials`). It does not refresh or rotate another client's tokens or present itself
  as another vendor's client.
- **Shell commands / subprocesses.** None.
- **Background work.** One short-lived daemon thread per non-trivial turn (12 s HTTP timeout, at
  most one in flight per session). No long-running processes.
- **Telemetry.** None.
- **Logging.** The text of user messages is never logged. On failure it logs only the exception
  class name, at debug level.
- **Self-updating code (rule 3).** None.
- **Core overrides (rule 9).** None; `hermes plugins validate` reports `no core override`.
- **Unattended runs (rule 12).** It never waits for a person: no prompts and no browser OAuth. It
  fails open (no context) under cron, the gateway and subagents.

### Naming and lineage (rules 15, 16, Names)

- The catalog key, manifest `name:` and registered provider name are all `gbrain-pointer`. That
  does not clash with `gbrain-plugin` (a status command, Desktop pane and setup scripts; not a
  memory provider) or `gbrain-retrieval-reflex` (a `pre_llm_call` hook; not a memory provider).
  The name includes `-pointer` so the bare `gbrain` key stays free for upstream.
- Original work, not a fork of either listed GBrain entry. They are different designs: this one is
  a single-select `memory.provider` that injects pointers only, prefetches non-blockingly, scopes
  credentials per profile and never writes. Not affiliated with GBrain or Nous Research.
- Submitted by the repository owner (rule 5).

### Validation

`hermes plugins validate` against the pinned tree, with Hermes Agent v0.21.5 (git `bc1f2679`):

```
✓ manifest — plugin.yaml parses
✓ manifest fields — name, version, description present
✓ requires_hermes — spec '>=0.21.5' parses
✓ config schema — not declared
✓ requires_env — all entries UPPER_SNAKE
✓ loadable — entry: __init__.py
✓ python dependencies — none declared
✓ capability probe — register() ran in isolation
✓ declared tools — matches registrations
✓ declared hooks — matches registrations
✓ declared middleware — matches registrations
✓ built-in tool collisions — no tools to check
✓ security scan — safe
✓ no core override — no runtime rebinds of Hermes core
◆ Isolation — runs in the plugin host (plugins.isolation: host)

Validation passed.
```

The repository's offline test suite (`tests/`, 25 tests, a stub `/token` + `/mcp` server on
`127.0.0.1`) covers:

- unavailable without credentials
- available through the secret scope with `os.environ` empty
- fails closed under multiplexing with no scope
- pointers returned once per session
- exactly one re-mint after a `401`
- minted tokens never written to `os.environ`
- two profiles never share a token
- trivial prompts skipped
- malformed responses fail open
- the prompt is never logged
- `prefetch()` never blocks
- loading through Hermes' real memory-provider discovery

### Compatibility (rule 14)

`requires_hermes: ">=0.21.5"` is the release this was validated against. The plugin falls back
when the newer `RecallStatus` and `spawn_context_thread` APIs are missing, but older releases were
not tested, so the floor stays at the tested version.

### Screenshots

None. The plugin has no UI.
