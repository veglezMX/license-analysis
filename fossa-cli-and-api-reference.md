# FOSSA CLI and API integration guide for compliance dashboards

**FOSSA provides a mature toolset for programmatic license compliance and vulnerability scanning**, combining a powerful CLI for CI/CD pipelines with a comprehensive REST API for custom integrations. The CLI handles scan triggering and policy testing with exit codes designed for automation, while the API enables querying scan results, managing projects, and retrieving compliance status. For a compliance dashboard, the recommended pattern combines CLI-based scanning in pipelines with API polling for results, supplemented by webhooks for real-time notifications.

## CLI commands enable full scan automation

The FOSSA CLI offers four primary commands for compliance workflows: **`fossa analyze`** uploads dependency data to FOSSA for analysis, **`fossa test`** checks policy compliance and returns actionable exit codes, **`fossa report`** generates attribution documents, and **`fossa list-targets`** previews discoverable analysis targets.

### The `fossa analyze` command and key options

```bash
fossa analyze [OPTIONS] [DIRECTORY]
```

| Flag | Purpose |
|------|---------|
| `--output, -o` | Print results to stdout without uploading (dry-run mode) |
| `--project NAME` | Override project identifier |
| `--revision REV` | Set revision (typically commit hash) |
| `--branch BRANCH` | Set branch name for tracking |
| `--policy NAME` | Assign specific policy by name |
| `--policy-id ID` | Assign policy by ID |
| `--endpoint URL` | FOSSA server URL (default: app.fossa.com) |
| `--fossa-api-key KEY` | API key (alternative to `FOSSA_API_KEY` env var) |
| `--static-only-analysis` | Use only static analysis, skip build tools |
| `--detect-vendored` | Enable Vendored Source Identification for C/C++ |
| `--debug` | Generate verbose debug output |

**Typical CI usage:**
```bash
export FOSSA_API_KEY=$FOSSA_API_KEY
fossa analyze --project "my-service" --revision "$(git rev-parse HEAD)" --branch "main"
```

### The `fossa test` command for policy gates

This command is **critical for CI/CD integration**, polling FOSSA until scan completion and returning exit codes suitable for pipeline gates.

```bash
fossa test [OPTIONS]
```

| Flag | Purpose |
|------|---------|
| `--timeout SECONDS` | Maximum wait time (default: 3600s) |
| `--json` | Output issues as machine-parseable JSON |
| `--diff REVISION` | Only report NEW issues since base revision |

**Exit codes:** `0` indicates no policy violations (build passes); `1` indicates issues found (build fails). The `--diff` flag is particularly valuable for pull request workflows, failing only on newly introduced violations rather than pre-existing technical debt.

**Differential testing example:**
```bash
fossa test --diff $BASE_COMMIT --json --timeout 300
```

### Output formats for compliance reporting

The `fossa report attribution` command generates compliance documents in multiple formats:

```bash
fossa report attribution --format [json|spdx|cyclonedx-json|markdown|text]
```

FOSSA supports **CycloneDX** (JSON/XML with VDR/VEX vulnerability data), **SPDX** (JSON/tag-value), plus HTML, CSV, PDF, and Markdown. For dashboard integration, JSON and CycloneDX-JSON provide the most structured data for parsing.

## REST API endpoints for dashboard integration

The FOSSA API uses a base URL of `https://app.fossa.com/api` with v2 endpoints at `/api/v2/`. Authentication uses either Bearer tokens or the token keyword in the Authorization header.

### Authentication configuration

```bash
# Bearer token (recommended)
Authorization: Bearer <API_TOKEN>

# Alternative format
Authorization: token <API_TOKEN>
```

FOSSA offers **two token types**: Push-Only tokens (safe for CI, can only upload data) and Full Access tokens (complete read/write permissions for dashboard integrations). Tokens are generated at `Organization Settings > Integrations > API`.

### Core API endpoints for compliance data

**Projects and Revisions:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v2/projects` | GET | List all projects with pagination |
| `/api/revisions?projectId=<locator>` | GET | List revisions for a project |
| `/api/revisions/<locator>` | GET | Get revision details |
| `/api/revisions/<locator>/dependencies` | GET | Get dependency graph |

**Issues and Policy Results:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v2/issues` | GET | Query issues by category, status, severity |
| `/api/v2/issues/categories` | GET | Get issue counts by category |
| `/api/v2/issues/statuses` | GET | Get counts by status (active/ignored) |
| `/api/vulns/by-locator` | POST | Check vulnerabilities for specific packages |

**Attribution Reports:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/revisions/<id>/attribution/download?format=HTML` | GET | Download HTML report |
| `/api/revisions/<id>/attribution/download?format=CSV` | GET | Download CSV report |

### Querying policy evaluation results

The Issues API provides policy violation data with rich filtering:

```bash
curl -G https://app.fossa.com/api/v2/issues \
  -d 'category=licensing' \
  -d 'scope[type]=project' \
  -d 'scope[id]=git%2Bgithub.com%2Forg%2Frepo' \
  -d 'filter[depths][0]=direct' \
  -d 'status=active' \
  -H 'Authorization: Bearer <TOKEN>'
```

**Filter parameters include:** `category` (licensing/quality/vulnerability), `scope[type]` (global/project), `filter[severity][]` (critical/high/medium/low), `filter[depths][]` (direct/deep for transitive), and `sort` options.

### Locator format for API requests

FOSSA identifies packages using locators in the format `<fetcher>+<package>$<version>`. Examples: `npm+lodash$4.17.21`, `maven+org.apache.commons:commons-lang3$3.12.0`. **URL encoding is required** for API calls (`+` → `%2B`, `/` → `%2F`, `$` → `%24`).

## Policy management enables customizable compliance rules

FOSSA policies define which licenses, vulnerabilities, and quality issues trigger violations. Four policy types exist: **Licensing** (acceptable license rules), **Security** (CVE severity filters, CWE exclusions), **Quality** (staleness, blocked packages), and **SBOM** (required fields for imports).

### Policy rule actions

Each rule applies one of three actions: **DENY** creates a blocking issue requiring removal, **FLAG for Review** creates an issue requiring manual approval, and **APPROVE** never creates issues for matching dependencies. Conditional rules can filter by code location, project naming patterns, or dependency linking type.

### Accessing policy results programmatically

Policy evaluation results surface through the Issues API rather than a dedicated policy endpoint. The response categorizes violations as `policy_conflict` (direct violation), `policy_flag` (flagged for review), or `unlicensed_dependency` (missing license). Quality issues include types like `outdated_dependency`, `blacklisted_dependency`, and `risk_abandonware`.

**Export options:** Issues can be exported to CSV via the API (`csv=true` parameter), and SBOM exports in CycloneDX/SPDX format capture the full dependency and license inventory suitable for external database synchronization.

## Webhook and polling patterns for real-time updates

For compliance dashboards, FOSSA supports both synchronous polling and asynchronous webhook notifications.

### Polling with `fossa test`

The CLI's `fossa test` command implements internal polling against the FOSSA backend, making it ideal for blocking CI gates. For dashboard integrations, you can replicate this by polling the revisions endpoint until the build task status indicates completion, then querying the Issues API.

### Available webhook integrations

FOSSA provides several webhook mechanisms: **VCS webhooks** trigger scans on commits/PRs (auto-configured for Quick Import projects), **CI webhooks** accept notifications from TravisCI and CircleCI, **Jira webhooks** enable bi-directional issue synchronization, and **Issue notification webhooks** send POST requests when new issues are detected.

The issue notification webhook payload includes:
```json
{
  "project_title": "service-name",
  "locator": "git+github.com/org/repo$revision",
  "new_issues": {
    "licensing": { "policy_conflict": 2 },
    "vulnerability": { "critical": 1, "high": 3 }
  }
}
```

**Setup note:** Issue notification webhooks currently require FOSSA customer success engagement for configuration—there is no self-service API for webhook management.

## Recommended integration architecture

For a compliance dashboard system integrating with FOSSA, a hybrid approach delivers the best results:

1. **Scan triggering:** Use `fossa analyze` in CI/CD pipelines with Push-Only tokens, passing project metadata (revision, branch) for traceability
2. **Policy gating:** Use `fossa test --json` for blocking pipeline gates with machine-parseable output
3. **Dashboard data sync:** Poll `/api/v2/issues` and `/api/v2/projects` with Full Access tokens to populate dashboard state
4. **Real-time alerts:** Configure Jira webhooks for issue tracking integration; request issue notification webhooks for dashboard push updates
5. **SBOM generation:** Use `fossa report attribution --format cyclonedx-json` to export dependency inventories for external compliance databases

**Rate limiting:** FOSSA does not publicly document specific rate limits. For enterprise deployments, contact FOSSA support for guidance on high-volume API usage patterns. The API is designed for enterprise customers and limits may vary by subscription tier.

## Conclusion

FOSSA's integration surface combines a robust CLI optimized for CI/CD automation with a REST API suitable for custom dashboard development. The CLI's exit code behavior and `--json` output enable straightforward pipeline integration, while the v2 Issues API provides the granular policy violation data needed for compliance dashboards. The main limitation for real-time architectures is that webhook configuration currently requires support engagement rather than API-based setup. For most compliance dashboard use cases, a polling-based approach using the Issues and Projects APIs, supplemented by CLI-generated SBOM exports, will provide comprehensive coverage of license compliance, vulnerability status, and policy evaluation results.