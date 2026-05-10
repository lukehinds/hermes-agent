# nono hermes

Sandbox profile, Hermes plugin, and Hermes skill for running [Hermes Agent](https://hermes-agent.nousresearch.com/) inside a [nono](https://nono.sh) security sandbox.

Install:

```bash
nono run --profile hermes -- hermes
```

If the pack is not already installed, nono will prompt to pull it.

## What's in the pack

- **`policy.json`** — sandbox profile loaded as `--profile hermes`. Grants Hermes state under `~/.hermes`, nono user profile writes under `~/.config/nono/profiles`, read-only package metadata under `~/.config/nono/packages`, the Hermes launcher directory, uv-managed Python runtimes under `~/.local/share/uv`, and read-only access to nono audit history.
- **`policy.json` network controls** — activates the nono `enterprise` network profile, provider credential routes, and L7 endpoint rules for OpenAI, Anthropic, and Gemini routes.
- **`plugin/nono-sandbox/`** — Hermes plugin. It registers a `nono_status` tool, `/nono-status` slash command, plugin-provenanced `nono-sandbox:nono-sandbox` skill, first-turn sandbox boundary context, denial remediation context, and metadata-only audit events under `~/.hermes/logs/nono-sandbox-audit.ndjson`.
- **`bin/nono-hermes-status.sh`** — small diagnostic script for checking Hermes, nono, current capabilities, and sensitive Hermes file permissions.
- **`templates/config-hardening.yaml`** — YAML merge patch for enabling the plugin, fail-closed Tirith scanning, secret redaction, private URL blocking, and Hermes skill-write guarding.

## Activating the plugin

`nono pull always-further/hermes` symlinks the plugin into:

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

## Credential and network filtering

The `hermes` nono profile starts Hermes behind nono's proxy-only network mode unless you override it with `--allow-net`. The profile allows the `enterprise` network set and requests credential routes for `openai`, `anthropic`, `gemini`, `github`, and `gitlab`.

For provider routes, nono injects a session-scoped phantom token into Hermes and swaps it for the real credential in the proxy. Configure the backing credentials in nono before relying on this path:

```bash
export OPENAI_API_KEY=...
export ANTHROPIC_API_KEY=...
export GEMINI_API_KEY=...
export GITHUB_TOKEN=...
export GITLAB_TOKEN=...
```

The OpenAI, Anthropic, and Gemini routes include method+path allowlists so model calls can proceed without opening arbitrary API endpoints.

## Research notes

The pack maps directly to Hermes security features:

- Hermes approvals catch risky commands, but nono remains the outer OS boundary. The plugin therefore steers the agent toward `nono why` and profile changes instead of approval workarounds.
- Hermes plugin-bundled skills preserve provenance and avoid copying registry-managed content into the mutable user skill tree.
- Hermes plugin hooks are the right CLI+gateway surface for sandbox diagnostics and metadata-only audit. Gateway-only event hooks are useful for production monitoring, but this pack keeps them out of the default install to avoid surprise network callbacks.
- Hermes supports online skills registries, direct URL installs, GitHub taps, and well-known skill endpoints. `registry.nono.sh` could be valuable as a curated nono-specific skill and pack registry if it keeps signed provenance, review metadata, and security-scan results rather than becoming an unreviewed skill dump.

## Source

`https://github.com/always-further/nono-packs/tree/main/hermes`

Published via Sigstore-signed releases triggered by tags matching `hermes-v*`.
