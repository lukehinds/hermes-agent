# nono hermes

Sandbox profile, Hermes plugin, and Hermes skill for running [Hermes Agent](https://hermes-agent.nousresearch.com/) inside a [nono](https://nono.sh) security sandbox.

Install:

```bash
PYTHONDONTWRITEBYTECODE=1 nono run --profile hermes -- hermes
```

If the pack is not already installed, nono will prompt to pull it.

## What's in the pack

- **`policy.json`** — sandbox profile loaded as `--profile hermes`. Grants Hermes state under `~/.hermes`, nono user profile writes under `~/.config/nono/profiles`, read-only package metadata under `~/.config/nono/packages`, the Hermes launcher directory, and uv-managed Python runtimes under `~/.local/share/uv`. It does not grant access to nono's own audit or rollback state.
- **`policy.json` network controls** — ships reusable provider credential route definitions for OpenAI, Anthropic, Gemini, GitHub, and GitLab, but enables none by default. Users opt into the routes they need from an extending profile. Requires nono v0.51+ so enabled routes also apply to TLS CONNECT traffic through nono's interception path.
- **`plugin/nono-sandbox/`** — Hermes plugin. It registers a `nono_status` tool, `/nono-status` slash command, plugin-provenanced `nono-sandbox:nono-sandbox` skill, first-turn sandbox boundary context, redacted proxy/TLS trust context, denial remediation context, and metadata-only audit events under `~/.hermes/logs/nono-sandbox-audit.ndjson`.
- **`bin/nono-hermes-status.sh`** — small diagnostic script for checking Hermes, nono, current capabilities, proxy/TLS trust state, and sensitive Hermes file permissions.
- **`templates/config-hardening.yaml`** — YAML merge patch for enabling the plugin, fail-closed Tirith scanning, secret redaction, private URL blocking, and Hermes skill-write guarding.

## Activating the plugin

`nono pull lukehinds/hermes` symlinks the plugin into:

```text
~/.hermes/plugins/nono-sandbox
```

and merges this into `~/.hermes/config.yaml`:

```yaml
plugins:
  enabled:
    - nono-sandbox
```

Hermes loads the skill through the plugin, preserving its package provenance:

```text
skill_view("nono-sandbox:nono-sandbox")
```

The plugin also exposes `/nono-status` and the `nono_status` tool after Hermes reloads.

`PYTHONDONTWRITEBYTECODE=1` prevents Python from trying to create `__pycache__` under the signed, read-only package store while Hermes imports the plugin. Do not grant write access to `~/.config/nono/packages/.../plugin/nono-sandbox` to silence that cache write.

When checking capabilities from outside Hermes, include the profile context:

```bash
nono why --profile hermes --path ~/.config/nono/packages/<ns>/hermes/plugin/nono-sandbox/__init__.py --op read
```

Inside a running Hermes sandbox, use `nono why --self` so the query uses the live capability file.

## Credential and network filtering

The base `hermes` profile does not enable provider credentials by default. This avoids warnings for unused providers and prevents unused credential routes from becoming part of the session boundary.

Create your own profile that extends `hermes`:

```bash
nono pull lukehinds/hermes
nono profile init hermes-agent --extends hermes --full --force
$EDITOR ~/.config/nono/profiles/hermes-agent.json
```

Then set only the credential routes you need:

```json
"network": {
  "block": false,
  "allow_domain": [],
  "credentials": ["gemini"],
  "open_port": [],
  "listen_port": [],
  "custom_credentials": {}
}
```

The empty `custom_credentials` object in the child profile is fine: child profiles inherit the route definitions from `hermes`. Do not remove `custom_credentials` from the base pack profile if you rely on its route definitions.

Common route names:

```json
"credentials": ["gemini"]
"credentials": ["anthropic"]
"credentials": ["openai"]
"credentials": ["gemini", "github"]
```

For provider routes, nono injects a session-scoped phantom token into Hermes and swaps it for the real credential in the proxy. The profile uses env-var-shaped nono keychain account names:

- `OPENAI_API_KEY` -> `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY` -> `ANTHROPIC_API_KEY`
- `GOOGLE_API_KEY` -> `GOOGLE_API_KEY` for Gemini. Hermes' native Gemini documentation uses this as the primary environment variable, and Hermes also accepts `GEMINI_API_KEY` as an alias.
- `GITHUB_TOKEN` -> `GITHUB_TOKEN`
- `GITLAB_TOKEN` -> `GITLAB_TOKEN`

The OpenAI, Anthropic, and Gemini routes include method+path allowlists so model calls can proceed without opening arbitrary API endpoints.

Store keys under nono's keychain service with the same account names.

macOS Keychain:

```bash
security add-generic-password -U -s "nono" -a "GOOGLE_API_KEY" -w
security add-generic-password -U -s "nono" -a "GITHUB_TOKEN" -w
```

Keep `-w` last so macOS prompts for the secret instead of recording it in shell history.

Linux Secret Service:

```bash
# Debian/Ubuntu
sudo apt install libsecret-tools gnome-keyring

# Fedora
sudo dnf install libsecret gnome-keyring

# Arch
sudo pacman -S libsecret gnome-keyring

secret-tool store --label="nono: GOOGLE_API_KEY" \
  service nono username GOOGLE_API_KEY target default

secret-tool store --label="nono: GITHUB_TOKEN" \
  service nono username GITHUB_TOKEN target default
```

On Linux this requires a running Secret Service provider such as GNOME Keyring or KWallet. In SSH-only or headless environments, check the nono credential docs before choosing a storage backend.

For 1Password, Apple Passwords, file-backed secrets, environment references, or a non-default keyring service, use nono credential URI refs in your extending profile's `custom_credentials` entries. nono supports refs such as `op://vault/item/field`, `apple-password://server/account`, `file:///absolute/path`, `env://VAR_NAME`, and `keyring://service/account`. See the nono docs for the full credential model:

- https://docs.nono.sh/usage/secrets
- https://docs.nono.sh/usage/flags

Run Hermes with the child profile:

```bash
PYTHONDONTWRITEBYTECODE=1 nono run --profile hermes-agent -- hermes
```

With nono v0.51 or newer, enabled credential routes also cover normal HTTPS SDK traffic that uses `CONNECT` through the nono proxy. When a route has credentials or endpoint rules, nono creates a session-scoped interception CA under `~/.nono/sessions/...`, injects the relevant trust environment variables (`SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`, `CURL_CA_BUNDLE`, and `GIT_SSL_CAINFO`), terminates the eligible TLS tunnel, and applies the same L7 filtering and credential injection inside the proxy. If TLS interception cannot be prepared, nono blocks L7-bearing CONNECT routes instead of allowing a bypass.

Use `allow_domain` only for additional non-credentialed destinations the agent needs. Credential routes already constrain their own upstream service and method/path rules, so a Gemini-only profile does not need broad domains just to call Gemini.

The status helper redacts proxy URL userinfo because v0.51 proxy URLs can contain the session proxy token.

## Research notes

The pack maps directly to Hermes security features:

- Hermes approvals catch risky commands, but nono remains the outer OS boundary. The plugin therefore steers the agent toward `nono why` and profile changes instead of approval workarounds.
- Hermes plugin-bundled skills preserve provenance and avoid copying registry-managed content into the mutable user skill tree.
- Hermes plugin hooks are the right CLI+gateway surface for sandbox diagnostics and metadata-only audit. Gateway-only event hooks are useful for production monitoring, but this pack keeps them out of the default install to avoid surprise network callbacks.
- Hermes supports online skills registries, direct URL installs, GitHub taps, and well-known skill endpoints. `registry.nono.sh` could be valuable as a curated nono-specific skill and pack registry if it keeps signed provenance, review metadata, and security-scan results rather than becoming an unreviewed skill dump.

## Source

`https://github.com/lukehinds/nono-packs/tree/main/hermes`

Published via Sigstore-signed releases triggered by tags matching `hermes-v*`.
