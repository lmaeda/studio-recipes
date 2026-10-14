# Evidence Profiles: Snyk MCP Output to Bundle Fields

How each local Snyk MCP tool's output maps to bundle fields, what identifies a finding, and what the
tool does not return. "Not returned" fields get an availability state (see `bundle-format.md`). Do
not derive them from other fields.

| Tool | `originatingScanner` | `category` | Output shape |
|---|---|---|---|
| `snyk_sca_scan` | `snyk-open-source` | `dependency_vulnerability`, or `license` when `id` starts with `snyk:lic:` | Mapped JSON: `{success, issueCount, issues[]}` |
| `snyk_code_scan` | `snyk-code` | `code_weakness` | Mapped JSON: `{success, issueCount, issues[]}` |
| `snyk_iac_scan` | `snyk-iac` | `iac_misconfiguration` | Native `snyk iac test --json` |
| `snyk_secret_scan` | `snyk-secrets` | `secret` | Native `snyk secrets test` output |
| `snyk_container_scan` | `snyk-container` | `container_vulnerability`, or `license` for `snyk:lic:` IDs | Native `snyk container test --json` |
| `snyk_sbom_scan` | `snyk-open-source` | `dependency_vulnerability` | Native `snyk sbom test --json` |

The SCA and Code tools return a summarized projection, not the CLI's full JSON or SARIF. The other
tools return the CLI's native output, whose optional fields vary with CLI version and target. Map
what is present and mark the rest `unknown`.

## Severity

- **SCA, container, IaC, SBOM:** Snyk severity with the scale `low | medium | high | critical`, and authority `snyk`.
- **Code:** the MCP derives severity from the SARIF level: `error` becomes `high`, `warning` becomes `medium`, anything else becomes `low`. Snyk Code has no `critical`. Record the scale as `low | medium | high (from SARIF level)`.

## SCA (`snyk_sca_scan`)

The tool emits **one issue per vulnerability and dependency path**. The same `id` repeats when a
package arrives through several paths.

| Bundle field | Source |
|---|---|
| `rule.id` | `id` (Snyk vulnerability ID, for example `SNYK-JS-…`) |
| `rule.aliases.cve` / `.cwe` | `cves`, `cwes` |
| `title` | `title` |
| `asset.package` | `ecosystem` (the CLI's package manager), `packageName`, `version` |
| `asset.manifest` | `filePath` (absolute; store it relative to the target) |
| `asset.transitive` | `isTransitiveDependency` |
| `evidence.introducedThrough` | `introducedThrough`; absent means a direct dependency |
| `remediation` | `fixedIn`, `remediation` (advice text) |
| `status` | `isIgnored` maps to `ignored`, otherwise `open` |
| `description` | `message`, which the MCP **truncates to 200 characters**; mark it `truncated` |

- **Identity:** a composite, `sca:{id}|{ecosystem}:{packageName}@{version}|{manifest}|{introducedThrough joined by " > " or "direct"}`, scoped to the target repository. It stays stable while the package, version, manifest, and path are unchanged.
- **Version meaning:** the version the CLI resolved from the lockfile or installed environment. Declared ranges are not returned.
- **Not returned:** CVSS score and vector, exploit maturity, EPSS, upgrade path and the direct dependency to change (beyond the advice text), package URL (do not construct one), dependency scope (dev, build, or runtime), disclosure dates, Snyk org, project and snapshot IDs.

## Code (`snyk_code_scan`)

Security issues only. The MCP drops code-quality results.

| Bundle field | Source |
|---|---|
| `rule.id` | `id`, which is the **rule ID**, not a finding ID |
| `rule.aliases.cwe` | `cwes` |
| `title` / `description` | `title` (rule short description) and `message` |
| `asset.file` | `filePath` (absolute; store it relative to the target) |
| `asset.line` / `asset.column` | `line`, `column` (**1-based**) |
| `evidence.dataflow` | `dataflow[]`: `{position, filePath, flowRange, content}`. `flowRange` lines and characters are **0-based**. Copy the steps unchanged and set `evidence.dataflowLineBase: 0`. |
| `status` | `isIgnored` maps to `ignored`, otherwise `open` |

- **Identity:** `fingerPrint`, a native SARIF fingerprint scoped to the repository. If it is empty, use the composite `code:{id}|{relative filePath}:{line}:{column}`.
- **Dataflow caveat:** the MCP merges steps that share a file and start line.
- **Not returned:** language (do not infer it from the extension), end line and column of the primary location, rule version, priority score, fix suggestions, per-file analysis completeness, Snyk org.
- `dataflow[].content` contains lines of the user's own source code. Keep them; they are evidence.

## IaC (`snyk_iac_scan`)

The output is native CLI JSON: one object per scanned file, or an array of them. Typical fields:

- per file: `targetFile`, `projectType`, `meta.org`
- per issue in `infrastructureAsCodeIssues[]`: `id` / `publicId`, `title`, `severity`, `msg` (resource address), `path[]`, `lineNumber`, `issue`, `impact`, `resolve`, `isIgnored`

How to use them:

- **Identity:** `iac:{publicId or id}|{targetFile}|{msg}`.
- **Asset:** an `iac_resource` with file, line, and resource address (`msg` / `path`).
- **Org:** `meta.org` when present. It is `known` here, unlike in SCA and Code output.
- **Not returned:** rendered or plan-time evaluation, resolved variables, and mapping to a deployed cloud resource. Mark these `unsupported`.

## Secrets (`snyk_secret_scan`)

The native output format depends on the CLI version.

- Keep the record in `nativeRecord` with every matched value or snippet replaced by `[REDACTED]`.
- **Identity:** the native fingerprint if one is present. Otherwise use `secret:{rule}|{relative file}:{line}`.
- **Never** store the secret value, even partially or hashed.

## Container (`snyk_container_scan`)

The output is native CLI JSON. Typical fields:

- per vulnerability in `vulnerabilities[]`: `id`, `title`, `severity`, `packageName`, `version`, `from[]`, `fixedIn`, `identifiers`, `cvssScore`
- top level: `docker.baseImage`, `docker.baseImageRemediation`, `platform`, `packageManager`, `path` (the image reference), `org`

How to use them:

- **Identity:** `container:{id}|{image digest, or the reference if no digest}|{packageName}@{version}|{from joined by " > "}`.
- **Asset:** a `container_image` built from the image reference. Record a manifest digest **only if the output contains one**; otherwise mark it `unknown`, because tags are mutable labels.
- **Remediation:** `fixedIn` and base-image advice from `docker.baseImageRemediation`. Say whether the fix belongs in the application image or in its base image.

## SBOM (`snyk_sbom_scan`)

The output is native CLI JSON. Map vulnerabilities as for SCA where the fields exist.

- **Asset:** the SBOM file. Record its path relative to the target.
- **Dependency paths:** only what the SBOM itself declares.

## Limitations that apply to every bundle

Copy these into `collection.limitations`:

1. **Local snapshot only.** Findings describe the working tree at `target.revision`, plus uncommitted changes when `workingTreeDirty` is true. They do not describe Snyk platform projects. The bundle has no org or project inventory, pagination, incremental sync, or deletion markers.
2. **Org is mostly unknown.** The CLI scans under its configured Snyk org, which SCA and Code output does not report.
3. **Ignore metadata is missing.** `isIgnored` reflects the `.snyk` policy and org ignore rules the CLI applied. The ignore reason, actor, and expiry are not returned.
4. **Code results are scoped.** Code findings cover security rules only. Their completeness depends on the files Snyk Code supports and accepts.
