# Security policy

## Reporting a vulnerability

Please report security problems **privately**: on this repository, open the **Security** tab and click **Report a vulnerability**. Do not open a public issue or PR.

Include what you found, how to reproduce it, and what an attacker could do with it. We aim to acknowledge reports within 7 days.

## In scope

- Credentials (client secret, access tokens) reaching anything other than the configured GBrain server, or reaching logs, `os.environ` or disk.
- One Hermes profile reading another profile's credentials, tokens or pointers.
- Message text or other data being sent anywhere other than the configured GBrain server.
- Prompt-injection paths that bypass the plugin's existing handling of GBrain results.
- Crashes or unbounded resource use triggered by a GBrain server response.

## Out of scope

- Vulnerabilities in GBrain or Hermes Agent themselves. Report those to [garrytan/gbrain](https://github.com/garrytan/gbrain) or [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent).
- A GBrain server you configured yourself returning misleading page titles. Titles are shown to the model as untrusted background context.

## Supported versions

Only the latest commit on `main`, and the commit pinned in the Hermes plugin catalog, receive fixes.
