# Contributing to gbrain-pointer

Thanks for helping. This is a small plugin, so the rules are short.

## Before you start

- Search [open and closed issues and PRs](https://github.com/praggybuilds/hermes-gbrain-pointer/issues?q=) first.
- For anything bigger than a small fix, open an issue to discuss it before writing code.
- Security problems: do not open a public issue. See [SECURITY.md](SECURITY.md).

## Project rules

Every change must keep these true. A PR that breaks one is declined, however good the code.

1. **Read-only.** The plugin never writes to GBrain. `sync_turn` stays a no-op, and it adds no tools.
2. **No new network destinations.** It talks only to the configured GBrain server (`/token` and `/mcp`). No telemetry, no third-party services.
3. **Standard library only.** No new Python dependencies.
4. **Fail open.** Any error means the turn gets no pointers, never an exception into the turn.
5. **Nothing private in logs.** Never log message text, tokens or secrets.
6. **Credentials stay per profile.** Read them through Hermes's secret scope, never write them to `os.environ`, and keep cached state keyed by profile.
7. **Docs match the code.** If you change what is sent, received or stored, update the "What it sends where" and "Privacy and security" sections of the README in the same PR.

## Running the checks

Tests are offline: a stub GBrain server on `127.0.0.1`, no network or credentials.

```bash
hermes --run-module unittest discover -s tests -v
# or, from a Hermes checkout's environment:
HERMES_AGENT_SRC=/path/to/hermes-agent python -m unittest discover -s tests -v
hermes plugins validate .
```

CI runs both on every PR (Python 3.14, Hermes `main`).

## Tests

- **Every fix needs a test that fails without the fix.** Say in the PR that you checked this.
- Test behaviour through the provider's public methods (`queue_prefetch`, `prefetch`, `on_session_switch` and the provider's stop method), not private helpers, where you can.
- Don't rely on a sleep as the only proof of something. Use the event-gated helpers already in `tests/`.

## Pull requests

- One topic per PR. Keep it small.
- Fill in the PR template.
- **AI-assisted PRs are welcome.** Say which tool and model you used, and that you have read and tested the change yourself.

## How review works

1. CI must pass.
2. The maintainer agent (`@murphbuilds`, an AI agent) reviews first and posts findings. Larger changes also get an independent review from a second AI model.
3. The owner (`@praggybuilds`) approves and merges. Only the owner's approval satisfies the branch rule; nothing is auto-merged.
4. Merged changes reach Hermes users only when the [Hermes plugin catalog](https://github.com/NousResearch/hermes-agent/tree/main/plugin-catalog) entry is re-pinned, which is a separate, reviewed PR.

By contributing, you agree your contribution is licensed under the [MIT License](LICENSE).
