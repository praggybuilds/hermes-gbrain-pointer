[MODEL: claude-opus-5-5] [AGENT: Murphy] RESPONDING ON BEHALF OF PRAGGY

## Add `gbrain-pointer` to the plugin catalog

Adds `plugin-catalog/gbrain-pointer.yaml`, which pins
[`praggybuilds/hermes-gbrain-pointer`](https://github.com/praggybuilds/hermes-gbrain-pointer) at
`<PIN_AFTER_PUSH>` (version `0.1.0`).

### What it does

`gbrain-pointer` is a read-only `memory.provider` for [GBrain](https://github.com/garrytan/gbrain).
After each non-trivial user turn it calls GBrain's MCP `search` tool on a background thread. On the
next turn `prefetch()` returns up to the configured limit (3 by default, capped at 10) of lines like
`[source:slug] title`, under a header that tells the model to treat them as fallible background
context, not instructions. The model gets page titles, slugs and source ids only, never page text,
and the plugin never writes to GBrain.

### Hermes surfaces used

- `MemoryProvider` ABC through `register(ctx)` → `ctx.register_memory_provider(...)`. The manifest
  is detected as `kind: exclusive` and selected with `memory.provider: gbrain-pointer`.
- `is_available`, `unavailable_reason`, `initialize`, `get_tool_schemas` (returns `[]`),
  `get_config_schema` (env-var fields only, so `save_config` keeps the default no-op), `prefetch`,
  `queue_prefetch`, `recall_status`, `sync_turn` (no-op), `on_session_switch` and `shutdown`.
- `agent.secret_scope.get_secret` for credentials; `agent.secret_scope.serves_routed_profile` and
  `current_secret_scope` to refuse a routed call that has no bound credential scope;
  `agent.secret_scope.current_secret_scope_home` and `hermes_constants.get_hermes_home_override`
  (with `hermes_constants.hermes_home_key`) to key in-memory state by profile;
  `agent.memory_provider.spawn_context_thread` for the background search, and `is_trivial_prompt`
  to skip greetings, acknowledgements and slash commands.
- No tools, hooks, middleware, CLI commands, skills, Desktop or dashboard parts, and no Python
  dependencies (standard library `urllib` only).

### Disclosures (rule 13)

- **Network calls.** Only to the GBrain server the user configures (`GBRAIN_MCP_URL`, default
  `http://127.0.0.1:3131/mcp`, the `gbrain serve --http` default):
  - `POST <origin>/token` with `grant_type=client_credentials`, the client id and the client
    secret. This happens on the first search and once more after a `401`, and only in
    client-credential mode (an explicit `GBRAIN_MCP_ACCESS_TOKEN` takes precedence and is never
    replaced by minting).
  - `POST <GBRAIN_MCP_URL>` with a JSON-RPC `tools/call search` after each non-trivial user turn.
    It carries the first 500 characters of the user's message, the result limit,
    `snippet_chars: 1`, `fields: "lean"` and an optional `source_id`.
  - **Received:** GBrain has no metadata-only search mode (`snippet_chars: 0` means full text), so
    the plugin requests the smallest cap. Each row is GBrain's lean row (`slug`, `title`,
    `source_id`, `id`, `type`, `score`, `effective_date`, `chunk_id`, `evidence`,
    `create_safety`, a 1-character `chunk_text` plus truncation marker, and any safety/provenance
    fields present). Only `slug`, `title` and `source_id` are kept; the rest is discarded in
    memory and never shown to the model, logged or stored. Rows GBrain marks
    `injection_suspected: true` are skipped entirely; clean rows after them still fill the limit.
  - Response bodies are read up to 16,384 bytes (`/token`) and 262,144 bytes (search). A larger
    body, or a larger declared `Content-Length`, is dropped before decoding or parsing, and the
    turn gets no pointers.
  - Redirects are never followed on either request, so credentials are not forwarded to another
    origin or downgraded to `http`.
  - No assistant text, tool output, files or message history is sent. Plain `http` to a
    non-loopback host is refused unless the user sets `GBRAIN_POINTER_ALLOW_HTTP=1`.
- **Third-party services.** None. GBrain is self-hosted by the user; this plugin contacts nothing
  else.
- **Reads outside its own data.** None. It reads only its own `GBRAIN_*` variables, through
  `agent.secret_scope.get_secret`. A call that serves a routed profile must have a bound secret
  scope, so the launch profile's process environment is never used for another profile. It reads no other tool's token files or browser profiles.
- **Stored credentials (rule 11).** None written. The user's own GBrain OAuth client secret comes
  from the profile `.env`. Minted access tokens are kept in memory per (profile home, token URL,
  client id, keyed hash of the secret) and never written to `os.environ`, disk or logs. Cached
  pointers are keyed by (profile home, session) and tagged with a keyed hash of the endpoint,
  source filter and credential, so another profile or a changed credential cannot consume them.
  The plugin mints tokens only for its own client
  (`client_credentials`). It does not refresh or rotate another client's tokens or present itself
  as another vendor's client.
- **Shell commands / subprocesses.** None.
- **Background work.** One short-lived daemon thread per non-trivial turn, with per-request HTTP
  timeouts of 3 s for `/token` and 12 s for search. The plugin tracks one current-generation search
  per profile and session. Reset and rewind start a new generation without stopping the old
  worker: an invalidated worker may keep running until its own timeout but cannot publish, so two
  workers for one session can briefly overlap. Shutdown likewise blocks publication and token
  caching; it does not cancel a request already sent. No long-running processes.
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

The repository's offline test suite (`tests/`, 61 tests, a stub `/token` + `/mcp` server
on `127.0.0.1`) covers:

- unavailable without credentials
- available through the secret scope with `os.environ` empty
- fails closed under multiplexing with no scope
- pointers returned once per session
- exactly one re-mint after a `401`
- minted tokens never written to `os.environ`
- two profiles never share a token, pointer or in-flight search, even with the same client id and
  session id; a changed or removed credential drops cached authority and pointers
- a routed task without a profile identity is refused, and so is a routed call (foreign home
  override) with no bound credential scope; stamped and override-backed scopes and single-profile
  process-env credentials keep working
- token and search bodies over their byte limit fail open, with exact, absent, understated or
  overstated `Content-Length` and chunked encoding; the reader never asks for more than limit+1 bytes
- rows flagged `injection_suspected` are skipped before dedup and counting; clean rows fill the limit
- lowering the limit caps a result that was already cached under a higher one
- redirects on `/token` and `/mcp` are refused and the redirect target receives nothing
- malformed `GBRAIN_MCP_URL` values make the provider unavailable without raising
- reset, rewind and shutdown while a search is in flight publish nothing
- an explicit access token takes precedence over client credentials and is never replaced by minting
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
