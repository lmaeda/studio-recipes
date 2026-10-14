# Snyk Plugin for Codex

Snyk security scanning, remediation, dependency health checks, and findings export for
[Codex](https://developers.openai.com/codex), powered by the local Snyk CLI MCP server.

| Component | What it does |
|---|---|
| `Snyk` MCP server | Runs `npx -y snyk@latest mcp -t stdio`: Snyk Code, Open Source, IaC, container, secrets, and SBOM scans, plus package health and upgrade-breakability checks |
| `snyk-fix` skill | Scan, fix, validate, and optionally open a PR for one issue or a batch |
| `secure-dependency-health-check` skill | Vet a package for vulnerabilities and maintenance health before adding it |
| `snyk-findings-export` skill | Run the scans you choose and write the findings, with native records and evidence, to a structured JSON file. Read-only. |

The plugin has **no lifecycle hooks**. For automatic Secure at Inception scanning while Codex
writes code, use the [Snyk Studio installer](../../installer/README.md), which configures Codex
hooks.

## Install

Prerequisites: Node.js (for `npx`) and a Snyk account.

```bash
codex plugin marketplace add snyk/studio-recipes
codex plugin add snyk@snyk-studio-recipes
```

Start a new Codex session after installing. The first session, and the first one after each new
Snyk CLI release, downloads the CLI through `npx` (about 275 MB), so the MCP server can take a
while to start. The plugin allows it 120 seconds instead of Codex's default 30. To install from a
local checkout instead, run
`codex plugin marketplace add <path-to-recipes>`. Codex reads the marketplace from
`.agents/plugins/marketplace.json`.

## Authenticate

Run `snyk auth` once. It opens a browser OAuth flow, and the Snyk CLI stores and refreshes the
token in its local configuration. The plugin's MCP server reads that configuration, so nothing
secret lives in the plugin or in prompts. You can also ask Codex to authenticate, which calls the
`snyk_auth` tool.

| Topic | Details |
|---|---|
| Scope | Scans run under the Snyk org configured in the CLI (`snyk config set org=<org>`), or the org passed to a tool. Results cover the local workspace, image, or SBOM you point a scan at. |
| Region | Tenants outside SNYK-US-01 must select their environment before `snyk auth`, for example `snyk config environment SNYK-EU-01` |
| Expiry and refresh | OAuth tokens refresh automatically. Run `snyk auth` again if a scan reports an authentication error. |
| Rotation and revocation | Run `snyk logout` to clear local credentials. Revoke tokens or service accounts in the Snyk web UI. A revoked credential makes scans fail with an authentication error; it never returns partial data. |

### Using `SNYK_TOKEN` (CI, service accounts)

Codex does not forward arbitrary environment variables to plugin MCP servers. Only `HOME`, `PATH`,
and similar basics reach them, so an exported `SNYK_TOKEN` is not visible to the plugin's server.
To use a token, define the server yourself in `~/.codex/config.toml`. A user-defined server with
the same name replaces the plugin's:

```toml
[mcp_servers.Snyk]
command = "npx"
args = ["-y", "snyk@latest", "mcp", "-t", "stdio"]
env_vars = ["SNYK_TOKEN"]
startup_timeout_sec = 120
```

## Using it alongside the Snyk Studio installer

If the installer already configured Codex:

- **MCP:** your `[mcp_servers.Snyk]` entry overrides the plugin's server, so you keep exactly one
  Snyk server.
- **Skills:** the installer also copies `snyk-fix` and `secure-dependency-health-check` into
  `~/.agents/skills/snyk/`. With the plugin enabled, Codex sees both copies. Keep one: either
  disable the plugin, or remove the installer's skill copies.

## Data handling

- Snyk Code scans send source files to Snyk for analysis, under your Snyk agreement. Open Source,
  container, IaC, and SBOM scans send dependency and configuration metadata.
- None of the plugin's scans create or update Snyk projects. `snyk-findings-export` never passes
  the IaC `report` option and never runs `snyk monitor`.
- `snyk-findings-export` writes its JSON file locally (default `snyk-findings.json` in the scanned
  directory) and never stores secret values or credentials found in repository URLs. The file
  contains source snippets from Code dataflow evidence, so treat it like source code.
- `snyk-fix` calls `snyk_send_feedback` to report counts of fixed and prevented issues.

## Versions

- **Plugin:** `0.1.0` (see `.codex-plugin/plugin.json`).
- **Snyk CLI:** resolved by `npx` as `snyk@latest` at server start. `snyk-findings-export`
  records the exact version in every file it writes.
- **Export format:** `snyk-findings-export/v1` (see
  `skills/snyk-findings-export/references/bundle-format.md`).

## Layout

The plugin uses the Codex compatibility layout: `.codex-plugin/plugin.json`, `.mcp.json`, and
`skills/`. It is the only layout where a plugin can raise its MCP server's startup timeout. The
portable `mcp.json` format has no field for it, and Codex ignores the setting in per-plugin user
config.
