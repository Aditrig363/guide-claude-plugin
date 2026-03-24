---
name: sonatype-guide
description: >-
  Evaluates, audits, upgrades, and compares software dependencies for security using the
  Sonatype Guide MCP server. Analyzes vulnerabilities, Developer Trust Scores, license risks,
  and malicious package detection. Use when adding or installing a dependency, upgrading to a
  newer version, auditing a project for vulnerable dependencies, comparing alternative libraries,
  or when dependency security, supply chain security, or CVEs are mentioned.
---

# Sonatype Guide Workflows

## Critical Rules

### Never Recommend Downgrades

When a user provides their current version, **every recommendation must be >= that version**. The MCP returns versions ranked by Developer Trust Score, which can include older versions — you MUST filter these out.

**Only two exceptions exist:**
1. The user explicitly and repeatedly insists on a downgrade after being warned.
2. There is a catastrophic, unpatched vulnerability in all versions >= current (e.g., log4shell-severity with no forward fix), AND an older version is unaffected. Even then, present the downgrade only as a cautious side note, not the primary recommendation, and explain the trade-offs.

If neither exception applies, do not mention older versions at all.

### Never Recommend Malicious Components

If `malicious: true`, warn the user immediately. Never recommend, never suggest "with caution" — there is no safe use of a malicious package.

---

## PURL Format

All tools accept Package URLs. Format: `pkg:<type>/<namespace>/<name>@<version>`

| Ecosystem | Format | Example |
|---|---|---|
| Maven | `pkg:maven/<groupId>/<artifactId>@<version>` | `pkg:maven/org.apache.logging.log4j/log4j-core@2.17.1` |
| NPM | `pkg:npm/<name>@<version>` | `pkg:npm/express@4.18.2` |
| PyPI | `pkg:pypi/<name>@<version>` | `pkg:pypi/requests@2.31.0` |
| NuGet | `pkg:nuget/<name>@<version>` | `pkg:nuget/Newtonsoft.Json@13.0.3` |
| Go | `pkg:golang/<module>@<version>` | `pkg:golang/github.com/gin-gonic/gin@1.9.1` |

For scoped NPM packages, Cargo, RubyGems, and other ecosystems see [references/purl-ecosystems.md](references/purl-ecosystems.md).

---

## Tools

Three MCP tools available via `sonatype-guide`. All accept arrays of up to 20 PURLs.

- **sonatype-guide:getComponentVersion** — Analyze a specific version. Returns CVEs with CVSS scores, licenses, malicious flag, end-of-life status, and policy compliance results.
- **sonatype-guide:getLatestComponentVersion** — Find the newest version. Version in PURL is optional.
- **sonatype-guide:getRecommendedComponentVersions** — Ranked upgrade recommendations with Developer Trust Scores, breaking change counts, and vulnerable method signatures. Omit version for new component picks; include version for upgrade recommendations.

---

## Interpreting Results

### Developer Trust Score

Sonatype's proprietary quality metric (0-100) factoring security, license compliance, quality, and maintainability.

| Range | Action |
|---|---|
| 90+ | Safe for production |
| 80-89 | Generally safe, minor issues |
| 70-79 | Upgrade recommended |
| Below 70 | Upgrade urgently |

### Vulnerability Types

The MCP separates **direct** and **transitive** vulnerabilities. Always communicate this distinction:

- **`directVulnerabilities`** — CVEs in the package's own code. These are the most urgent — the user's code directly calls this library.
- **`transitiveVulnerabilities`** — CVEs in dependencies of the package. Still important, but may be mitigated by how intermediate libraries use the affected code. Note this nuance to users.

When presenting results, list direct vulnerabilities first and label them clearly. For transitive vulnerabilities, mention that upgrading the top-level package often resolves these (since it pulls in patched transitive deps).

### Policy Compliance

`getComponentVersion` returns a `policyCompliance` object indicating whether the component passes organizational policies:

```
policyCompliance: {
  compliant: true/false,
  conditions: [
    { conditionId: "cvss-threshold", conditionName: "CVSS < 7.0", passing: true/false },
    { conditionId: "license-threat-group", conditionName: "No Copyleft Licenses", passing: true/false },
    { conditionId: "malware", conditionName: "No Malware", passing: true/false }
  ]
}
```

Surface this in audit and evaluation workflows — it gives users a quick pass/fail for enterprise governance. When `compliant: false`, call out which specific conditions failed.

### Flags and Sentinels

- `malicious: true` — Supply chain attack. Do NOT use. Warn immediately.
- `endOfLife: true` — No longer maintained. Plan migration.
- `licenseThreatLevels` — Map of license to threat score. 0 = no concern. Higher = more restrictive.
- `catalogDate` — Epoch milliseconds when the version was cataloged. Use to note data freshness, especially for fast-moving packages (e.g., "cataloged 2 years ago" vs "cataloged last week").

### Null, Empty, and Zero Values

These have distinct meanings — do not conflate them:

| Field | `null` | `"0"` or `0` | `[]` empty array |
|---|---|---|---|
| `breakingChangesCount` | **Not analyzed** — unknown risk, do NOT say "no breaking changes" | Analyzed, confirmed no breaking changes | N/A |
| `vulnerableMethods` | Not checked | N/A | Checked, none found |
| `vulnerableMethods[].methodSignatures` | N/A | N/A | CVE confirmed but specific affected methods not yet mapped |

When `breakingChangesCount` is `null`, tell the user: "Breaking changes have not been analyzed for this upgrade path — review the changelog before upgrading."

---

## Version Recommendation Logic

When recommending versions from `getRecommendedComponentVersions` results:

1. **Filter out downgrades.** Remove any `toVersion` where the version is lower than `fromVersion`.
2. **Prefer the user's current major version.** Default recommendation should be the highest-scoring version within the same major version line.
3. **Present major version upgrades separately.** If a higher major version scores better, mention it as a secondary option with clear warnings about breaking changes.
4. **When `breakingChangesCount` is `null` for a major version jump**, explicitly warn that breaking change analysis is unavailable and recommend reviewing the migration guide.

**Example prioritization** for a user on express@4.18.2:
- Primary: best 4.x version (e.g., 4.22.1 — same major, lower migration risk)
- Secondary: best 5.x version (e.g., 5.1.0 — higher major, may have better security posture but requires migration effort)

---

## Workflow 1: Evaluate Before Adding

**Trigger**: Adding a new dependency, asking "is X safe", or choosing a version.

1. Build the PURL (version optional).
2. Call `sonatype-guide:getRecommendedComponentVersions`.
3. If the top result scores 90+ with no CVEs, recommend it.
4. If it has issues, call `sonatype-guide:getComponentVersion` for full details including policy compliance, and present trade-offs.
5. Always check `malicious` and `endOfLife`.

**Output**:

```
| Version | Trust Score | Direct CVEs | Transitive CVEs | License | Policy |
|---------|-------------|-------------|-----------------|---------|--------|
| x.y.z   | 99          | 0           | 0               | MIT     | Pass   |

Recommendation: ...
```

---

## Workflow 2: Upgrade Advisor

**Trigger**: Upgrading a dependency, asking "should I upgrade X", or responding to a known vulnerability.

1. Build the PURL with the current version.
2. Call `sonatype-guide:getRecommendedComponentVersions`.
3. **Filter out any version older than the current one.**
4. Compare `fromVersion` (current) against remaining `toVersions`.
5. Group recommendations: same-major first, then cross-major.
6. For the top 2-3 recommendations, highlight Trust Score delta, CVEs resolved, breaking changes, and any new issues.
7. If `vulnerableMethods` data exists for the current version, mention affected methods to help assess exposure.

**Output**:

```
Current: <package>@<version> (Trust Score: X, Direct CVEs: N, Transitive CVEs: M)

Recommended (same major):
1. <version> (Trust Score: Y) — <rationale>

Also available (major upgrade):
2. <version> (Trust Score: Z) — <rationale, breaking change warning>

Breaking changes to review: N (or "not analyzed — review changelog")
```

---

## Workflow 3: Project Audit

**Trigger**: "Audit dependencies", "check for vulnerabilities", "scan for security issues", or dependency health report requests.

1. Find the project's dependency manifest. Prefer lock files for exact versions.
2. Parse dependencies and build PURLs.
3. Batch-query `sonatype-guide:getComponentVersion` (up to 20 per call). For larger projects, prioritize direct dependencies.
4. Sort by severity: malicious first, then policy non-compliant, then end-of-life, then CVEs by CVSS (direct before transitive), then license concerns.
5. For packages with issues, call `sonatype-guide:getRecommendedComponentVersions` to suggest fixes — **only recommend upgrades, never downgrades**.

**Output**:

```
## Dependency Audit: <project>

Scanned: N dependencies | Issues: X | Policy violations: Y

### Critical
- <package>@<version>: <issue> [DIRECT]
  Recommended upgrade: <version> (Trust Score: Y)

### Warnings
- <package>@<version>: <issue> [TRANSITIVE]
  Recommended upgrade: <version> (Trust Score: Y)

### Summary
| Metric | Count |
|--------|-------|
| Malicious | 0 |
| Policy non-compliant | 1 |
| End of Life | 1 |
| Critical/High CVEs (direct) | 2 |
| Critical/High CVEs (transitive) | 3 |
| Medium CVEs | 3 |
| License concerns | 1 |
```

---

## Workflow 4: Dependency Comparison

**Trigger**: Choosing between alternatives ("axios vs got", "which library for X"), or evaluating competing packages.

1. Build PURLs for each candidate. If no version specified, call `sonatype-guide:getLatestComponentVersion` first.
2. Call `sonatype-guide:getRecommendedComponentVersions` on each to get Trust Scores.
3. Call `sonatype-guide:getComponentVersion` on each for policy compliance details.
4. Compare: Trust Score, direct CVEs, transitive CVEs, license, policy compliance, end-of-life, malicious flag.

**Output**:

```
| Metric | lib A | lib B | lib C |
|--------|-------|-------|-------|
| Latest Version | x.y.z | a.b.c | d.e.f |
| Trust Score | 99 | 85 | 72 |
| Direct CVEs | 0 | 1 | 2 |
| Transitive CVEs | 0 | 0 | 1 |
| License | MIT | Apache-2.0 | GPL-3.0 |
| Policy Compliant | Yes | Yes | No |

Recommendation: <lib A> — <rationale>
```

---

## Behaviors

- Proactively check dependencies when the user mentions adding, installing, or updating one.
- Prefer `sonatype-guide:getRecommendedComponentVersions` for decision-making — richest data.
- **Always filter out versions older than the user's current version** from recommendations.
- Default to same-major-version recommendations; present major upgrades as a secondary option.
- If `malicious: true`, warn immediately. Never recommend a malicious component.
- If `policyCompliance.compliant` is `false`, call out failing conditions.
- Distinguish direct vs transitive vulnerabilities in all output.
- When `breakingChangesCount` is `null`, warn that analysis is unavailable.
- Batch PURLs to minimize API calls.
