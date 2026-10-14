# Bundle Format: `snyk-findings-export/v1`

One JSON document per export. Field names are this skill's own. Each finding's `nativeRecord`
keeps the scanner's original field names and values.

## Availability states

A value written inline is **known**: it came from a named scan or command in this bundle. When a
field applies to the finding's category but has no known value, leave it out of the finding and list
it under `unavailable` instead:

```json
"unavailable": {
  "cvss": { "state": "unsupported", "reason": "snyk_sca_scan does not return CVSS" }
}
```

| State | Meaning |
|---|---|
| `unknown` | The field applies, but this run did not establish it. Examples: a git command failed, or an optional native field was missing. |
| `unsupported` | The local Snyk MCP tool never returns this field (see `evidence-profiles.md`). |
| `truncated` | A value is present but cut short. Keep the partial value inline and also list the field here. |
| `redacted` | Removed on purpose. Use this for secret values and credentials. |
| `denied` | The scan failed on authentication or permissions, so nothing was observed. |

Fields that do not apply to a category, such as a package for a Code finding, are simply left out.
Every reason names the tool or command responsible.

## Top level

| Field | Content |
|---|---|
| `format` | `"snyk-findings-export/v1"` |
| `provider` | `"snyk"` |
| `collection.collector` | `{skill: "snyk-findings-export", pluginVersion}`. Take `pluginVersion` from the `version` field of `.codex-plugin/plugin.json` at the plugin root (two directories above this skill's folder), or use `"unknown"`. |
| `collection.scanner` | `{name: "snyk-cli", version, via: "snyk mcp -t stdio"}`. `version` comes from `snyk_version`. |
| `collection.startedAt` / `finishedAt` | UTC ISO-8601 timestamps, for example `2026-10-01T12:00:00Z` (see "Timestamps" in `SKILL.md`) |
| `collection.target` | `{path, repository, revision, workingTreeDirty, changedPaths}`. Use `"unknown"` and an `unavailable` entry when git is not available. |
| `collection.org` | A known org (from IaC or container native output), or `{state, reason}` |
| `collection.scans[]` | One entry per scan type, including those not requested: `{category, tool, arguments, startedAt, finishedAt, outcome, issueCount, error}`. `outcome` is `succeeded`, `failed`, `denied`, `unavailable` (the server does not offer the tool), or `not_requested`. |
| `collection.completeness` | `{collection, analysis}`. Set `collection` to `"complete"` only when every requested scan succeeded; otherwise `"partial"`. `analysis` is a short sentence pointing at the scan outcomes and limitations. |
| `collection.limitations` | The list from `evidence-profiles.md`, plus anything specific to this run |
| `findings[]` | One record per issue, ordered by scan, then by severity from critical to low |

## Finding record

| Field | Content |
|---|---|
| `id` | `{value, kind: "native" or "composite", basis, scope}`. `basis` names the source field or composite rule. |
| `category`, `originatingScanner` | From the table in `evidence-profiles.md` |
| `title` | The tool's title |
| `description` | `{text}`. Mark it `truncated` when it is a cut-off SCA message. |
| `rule` | `{id, aliases: {cve: [], cwe: []}}` |
| `severity` | `{value, scale, authority}` |
| `status` | `{value: "open" or "ignored", meaning, authority: "snyk-cli"}` |
| `asset` | The category-specific target (see the example). Paths are relative to `collection.target.path`. |
| `remediation` | `{fixedIn, advice}` when present. A fix being available is not a fix being applied. |
| `evidence` | `introducedThrough`, or `dataflow` together with `dataflowLineBase: 0` |
| `observation` | `{scan: <category>, observedAt: <that scan's finishedAt>}` |
| `unavailable` | Availability entries for this finding |
| `nativeRecord` | The tool's issue object, copied unchanged (secrets: values redacted) |

## Worked example (sanitized)

```json
{
  "format": "snyk-findings-export/v1",
  "provider": "snyk",
  "collection": {
    "collector": { "skill": "snyk-findings-export", "pluginVersion": "0.1.0" },
    "scanner": { "name": "snyk-cli", "version": "1.1302.1", "via": "snyk mcp -t stdio" },
    "startedAt": "2026-10-01T12:00:00Z",
    "finishedAt": "2026-10-01T12:03:10Z",
    "target": {
      "path": "/work/checkout-service",
      "repository": "https://github.com/acme/checkout-service.git",
      "revision": "0123456789abcdef0123456789abcdef01234567",
      "workingTreeDirty": false,
      "changedPaths": []
    },
    "org": { "state": "unknown", "reason": "snyk_sca_scan and snyk_code_scan do not report the Snyk org" },
    "scans": [
      { "category": "sca", "tool": "snyk_sca_scan", "arguments": { "path": "/work/checkout-service", "all_projects": true, "include_ignores": true }, "startedAt": "2026-10-01T12:00:02Z", "finishedAt": "2026-10-01T12:01:30Z", "outcome": "succeeded", "issueCount": 1, "error": null },
      { "category": "code", "tool": "snyk_code_scan", "arguments": { "path": "/work/checkout-service", "include_ignores": true }, "startedAt": "2026-10-01T12:01:31Z", "finishedAt": "2026-10-01T12:03:08Z", "outcome": "succeeded", "issueCount": 1, "error": null },
      { "category": "iac", "tool": "snyk_iac_scan", "arguments": {}, "startedAt": null, "finishedAt": null, "outcome": "not_requested", "issueCount": null, "error": null }
    ],
    "completeness": { "collection": "complete", "analysis": "SCA and Code succeeded; IaC, secrets, container and SBOM were not requested." },
    "limitations": [
      "Local snapshot only: findings describe the working tree at target.revision, not Snyk platform projects.",
      "isIgnored reflects policy the CLI applied; ignore reason, actor and expiry are not returned."
    ]
  },
  "findings": [
    {
      "id": { "value": "sca:SNYK-JS-DEMOPARSER-0000001|npm:demo-parser@1.2.0|package.json|web-framework@4.1.0 > demo-parser@1.2.0", "kind": "composite", "basis": "sca composite (evidence-profiles.md)", "scope": "collection.target.repository" },
      "category": "dependency_vulnerability",
      "originatingScanner": "snyk-open-source",
      "title": "Prototype Pollution",
      "description": { "text": "Prototype Pollution affecting package demo-parser. Upgrade web-framework to 4.2.0." },
      "rule": { "id": "SNYK-JS-DEMOPARSER-0000001", "aliases": { "cve": ["CVE-2026-00001"], "cwe": ["CWE-1321"] } },
      "severity": { "value": "high", "scale": "low | medium | high | critical", "authority": "snyk" },
      "status": { "value": "open", "meaning": "Reported by the scan at observedAt and not suppressed by policy", "authority": "snyk-cli" },
      "asset": { "type": "package_occurrence", "package": { "ecosystem": "npm", "name": "demo-parser", "version": "1.2.0" }, "manifest": "package.json", "transitive": true },
      "remediation": { "fixedIn": ["1.2.1"], "advice": "Upgrade web-framework to 4.2.0." },
      "evidence": { "introducedThrough": ["web-framework@4.1.0", "demo-parser@1.2.0"] },
      "observation": { "scan": "sca", "observedAt": "2026-10-01T12:01:30Z" },
      "unavailable": {
        "cvss": { "state": "unsupported", "reason": "snyk_sca_scan does not return CVSS" },
        "packageUrl": { "state": "unsupported", "reason": "snyk_sca_scan does not return a package URL" },
        "org": { "state": "unknown", "reason": "snyk_sca_scan does not report the Snyk org" }
      },
      "nativeRecord": { "id": "SNYK-JS-DEMOPARSER-0000001", "title": "Prototype Pollution", "severity": "high", "cwes": ["CWE-1321"], "cves": ["CVE-2026-00001"], "packageName": "demo-parser", "version": "1.2.0", "ecosystem": "npm", "fixedIn": ["1.2.1"], "remediation": "Upgrade web-framework to 4.2.0.", "filePath": "/work/checkout-service/package.json", "message": "Prototype Pollution affecting package demo-parser. Upgrade web-framework to 4.2.0.", "isTransitiveDependency": true, "introducedThrough": ["web-framework@4.1.0", "demo-parser@1.2.0"] }
    },
    {
      "id": { "value": "4f1c0d9e2a7b", "kind": "native", "basis": "fingerPrint", "scope": "collection.target.repository" },
      "category": "code_weakness",
      "originatingScanner": "snyk-code",
      "title": "SQL Injection",
      "description": { "text": "Unsanitized input from an HTTP parameter flows into a SQL query." },
      "rule": { "id": "javascript/Sqli", "aliases": { "cve": [], "cwe": ["CWE-89"] } },
      "severity": { "value": "high", "scale": "low | medium | high (from SARIF level)", "authority": "snyk" },
      "status": { "value": "open", "meaning": "Reported by the scan at observedAt and not suppressed by policy", "authority": "snyk-cli" },
      "asset": { "type": "source_location", "file": "src/reports.js", "line": 85, "column": 18 },
      "evidence": {
        "dataflowLineBase": 0,
        "dataflow": [
          { "position": 0, "filePath": "/work/checkout-service/src/reports.js", "flowRange": { "start": { "line": 79, "character": 20 }, "end": { "line": 79, "character": 35 } }, "content": "const id = req.query.reportId;" },
          { "position": 1, "filePath": "/work/checkout-service/src/reports.js", "flowRange": { "start": { "line": 84, "character": 17 }, "end": { "line": 84, "character": 60 } }, "content": "db.query(`SELECT * FROM reports WHERE id = ${id}`);" }
        ]
      },
      "observation": { "scan": "code", "observedAt": "2026-10-01T12:03:08Z" },
      "unavailable": {
        "language": { "state": "unsupported", "reason": "snyk_code_scan does not return the language" },
        "ruleVersion": { "state": "unsupported", "reason": "snyk_code_scan does not return a rule version" }
      },
      "nativeRecord": { "id": "javascript/Sqli", "title": "SQL Injection", "severity": "high", "cwes": ["CWE-89"], "filePath": "/work/checkout-service/src/reports.js", "line": 85, "column": 18, "message": "Unsanitized input from an HTTP parameter flows into a SQL query.", "fingerPrint": "4f1c0d9e2a7b", "dataflow": [ { "position": 0, "filePath": "/work/checkout-service/src/reports.js", "flowRange": { "start": { "line": 79, "character": 20 }, "end": { "line": 79, "character": 35 } }, "content": "const id = req.query.reportId;" }, { "position": 1, "filePath": "/work/checkout-service/src/reports.js", "flowRange": { "start": { "line": 84, "character": 17 }, "end": { "line": 84, "character": 60 } }, "content": "db.query(`SELECT * FROM reports WHERE id = ${id}`);" } ] }
    }
  ]
}
```
