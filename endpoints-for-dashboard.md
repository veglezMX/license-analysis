# FOSSA API endpoints for compliance dashboard integration

Building a compliance dashboard requires **35+ distinct API endpoints** across six functional areas. All endpoints use `https://app.fossa.com/api` as the base URL, require Bearer token authentication, and return JSON responses. The critical "locator" format—`{fetcher}+{package}${version}`—is fundamental to nearly every API call.

## Authentication applies to all endpoints

Every FOSSA API request requires a Bearer token in the Authorization header. Two token types exist: **Full Access tokens** provide complete read/write permissions for dashboards, while **Push-Only tokens** only allow build uploads (suitable for CI/CD). Generate tokens at Organization Settings → Integrations → API.

```bash
Authorization: Bearer <FOSSA_API_TOKEN>
```

---

## Project endpoints power dashboard navigation

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/v2/projects` | List all projects with filtering |
| GET | `/api/projects/{locator}` | Get single project details |
| PUT | `/api/projects/{locator}` | Update project settings |
| PUT | `/api/v2/projects/policy` | Bulk update project policies |

### List all projects
```
GET https://app.fossa.com/api/v2/projects
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `title` | string | Filter by project name |
| `sort` | string | Sort field (e.g., `title`) |
| `count` | integer | Results per page |
| `offset` | integer | Pagination offset |

**Example Request:**
```bash
curl -G 'https://app.fossa.com/api/v2/projects' \
  -d 'title=my-project' \
  -d 'sort=title' \
  -d 'count=50' \
  -H 'Authorization: Bearer $TOKEN'
```

**Response includes:** Project `locator`, `last_analyzed_revision` (critical for fetching latest scan data), and `teamProjects` array.

### Get project details with branches
```
GET https://app.fossa.com/api/projects/{encoded-locator}?ref_type=branch
```

Returns revision references by branch name, enabling branch-specific compliance tracking.

---

## Revision endpoints track project versions

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/revisions?projectId={locator}` | List all revisions for a project |
| GET | `/api/revisions/{revision-locator}` | Get single revision details |

### List project revisions
```
GET https://app.fossa.com/api/revisions?projectId={url-encoded-project-locator}
```

**Example:**
```bash
curl -G 'https://app.fossa.com/api/revisions' \
  -d 'projectId=custom%2B27932%2FMyProject' \
  -H 'Authorization: Bearer $TOKEN'
```

### The locator format explained

FOSSA uses locators as universal identifiers. **Project locators** follow `{fetcher}+{org_id}/{project_name}` format (e.g., `custom+27932/DocsExample` or `git+github.com/org/repo`). **Revision locators** append a version: `{project-locator}${revision-id}` (e.g., `custom+27932/DocsExample$2023-03-29T15:40:33Z`).

When using locators in URLs, apply URL encoding: `/` → `%2F`, `+` → `%2B`, `$` → `%24`, `:` → `%3A`.

---

## Dependency endpoints enumerate project components

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/revisions/{locator}/dependencies` | Get all dependencies for revision |
| GET | `/api/v2/revisions/{locator}/dependencies` | Enhanced dependency listing |
| GET | `/api/v2/revisions/{locator}/dependencies/package-managers` | List package managers used |
| GET | `/api/revisions/{locator}/parent_projects` | Reverse lookup: find projects using dependency |
| POST | `/api/revisions/{locator}/list-dependencies` | Complex filtered dependency query |

### Get revision dependencies
```
GET https://app.fossa.com/api/revisions/{revision-locator}/dependencies
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `include_ignored` | boolean | Include ignored dependencies |
| `limit` | integer | Pagination limit (default ~750) |
| `offset` | integer | Pagination offset |

**Example:**
```bash
curl -G 'https://app.fossa.com/api/revisions/custom%2B27932%2FMyProject%242024-01-15T10%3A30%3A00Z/dependencies' \
  -H 'Authorization: Bearer $TOKEN'
```

**Response structure per dependency:**
```json
{
  "locator": "npm+package-name$1.0.0",
  "loc": {
    "fetcher": "npm",
    "package": "package-name",
    "revision": "1.0.0"
  },
  "licenses": [
    {
      "licenseId": "MIT",
      "title": "MIT License",
      "text": "...",
      "url": "..."
    }
  ]
}
```

### Find projects affected by a dependency
```
GET https://app.fossa.com/api/revisions/{dependency-locator}/parent_projects
```

Critical for impact analysis when vulnerabilities are discovered in a component.

---

## License endpoints reveal compliance status

License data is embedded in dependency responses and exposed through dedicated endpoints:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/v2/issues/license-list` | List all detected licenses |
| GET | `/api/v2/dependencies/custom-licenses` | Get organization's custom licenses |
| GET | `/api/v2/dependencies/{locator}` | Get global dependency license info |
| GET | `/api/project_group/{id}/release/{releaseId}/licenses` | Licenses for release group |

### License data structure

When fetching dependencies, each includes a `licenses` array with these fields:

| Field | Type | Description |
|-------|------|-------------|
| `ID` | int64 | Internal identifier |
| `LicenseID` | string | SPDX ID (e.g., "MIT", "GPL-2.0") |
| `Title` | string | Human-readable name |
| `URL` | string | License reference URL |
| `Text` | string | Full license text |
| `Attribution` | string | Required attribution text |
| `Copyright` | string | Copyright notice |
| `Ignored` | boolean | Whether ignored by policy |

---

## Issues API v2 handles compliance violations and vulnerabilities

The Issues API is the **core endpoint for compliance dashboards**, providing unified access to licensing issues, security vulnerabilities, and policy violations.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/v2/issues` | List issues with comprehensive filtering |
| GET | `/api/v2/issues/categories` | Get issue counts by category |
| GET | `/api/v2/issues/statuses` | Get counts by status (active/ignored) |
| GET | `/api/v2/issues/{issueId}` | Get specific issue details |
| GET | `/api/v2/issues/{issueId}/affected-projects` | Find all affected projects |

### List issues with filters
```
GET https://app.fossa.com/api/v2/issues
```

**Required Parameters:**
| Parameter | Type | Values |
|-----------|------|--------|
| `category` | string | `licensing`, `vulnerability`, `quality` |

**Scope Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `scope[type]` | string | `global` or `project` |
| `scope[id]` | string | Project locator (if type=project) |
| `scope[revision]` | string | Specific revision (optional) |

**Filter Parameters by Category:**

For **licensing** issues:
- `filter[type][]`: `policy_conflict`, `policy_flag`, `unlicensed_dependency`
- `filter[depths][]`: `direct`, `deep`
- `filter[search]`: Search dependency names
- `filter[ticketed][]`: `has_ticket`, `no_ticket`

For **vulnerability** issues:
- `filter[severity][]`: `critical`, `high`, `medium`, `low`, `unknown`
- `filter[depths][]`: `direct`, `deep`
- `filter[search]`: Search dependency names or CVE IDs

For **quality** issues:
- `filter[type][]`: `outdated_dependency`, `blacklisted_dependency`, `risk_abandonware`, `risk_empty-package`, `risk_native-code`

**Pagination and Export:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number |
| `count` | integer | Items per page |
| `sort` | string | `severity_desc`, `created_at_desc`, `package_asc` |
| `status` | string | `active` (default), `ignored` |
| `csv` | boolean | Email CSV export when true |

### Example: Get critical vulnerabilities for a project
```bash
curl -G 'https://app.fossa.com/api/v2/issues' \
  -d 'category=vulnerability' \
  -d 'scope[type]=project' \
  -d 'scope[id]=git%2Bgithub.com%2Forg%2Frepo' \
  -d 'filter[severity][0]=critical' \
  -d 'filter[severity][1]=high' \
  -H 'Authorization: Bearer $TOKEN'
```

### Example: Get all licensing violations globally
```bash
curl -G 'https://app.fossa.com/api/v2/issues' \
  -d 'category=licensing' \
  -d 'scope[type]=global' \
  -d 'filter[type][0]=policy_conflict' \
  -H 'Authorization: Bearer $TOKEN'
```

### Get issue category counts
```
GET https://app.fossa.com/api/v2/issues/categories
```

**Response:**
```json
{
  "licensing": 12,
  "vulnerability": 45,
  "quality": 8
}
```

---

## Vulnerability-specific endpoints enable security scanning

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/vulns/by-locator` | Check vulnerabilities for packages |
| GET | `/api/vulns/cve-list` | Get list of CVEs |
| GET | `/api/vulns/{vulnId}/revisions/{revisionId}/remediation-guidance` | Get fix recommendations |

### Check vulnerabilities by package locator
```
POST https://app.fossa.com/api/vulns/by-locator
Content-Type: application/json
```

**Request Body:**
```json
{
  "locators": [
    "npm+lodash$4.17.20",
    "npm+express$4.17.1",
    "mvn+org.apache.logging.log4j:log4j-core$2.14.0"
  ]
}
```

**Example:**
```bash
curl -X POST 'https://app.fossa.com/api/vulns/by-locator' \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"locators": ["npm+lodash$4.17.20"]}'
```

---

## Report generation endpoints support compliance documentation

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/revisions/{locator}/attribution/download` | Download report (HTML, MD, CSV) |
| GET | `/api/revisions/{locator}/attribution/email` | Email report (TXT, PDF) |
| GET | `/api/revisions/{locator}/attribution/json` | Get JSON attribution data |

### Download attribution report
```
GET https://app.fossa.com/api/revisions/{revision-locator}/attribution/download
```

**Query Parameters:**
| Parameter | Values | Description |
|-----------|--------|-------------|
| `download` | `true` | Required for direct download |
| `format` | `HTML`, `MD`, `CSV` | Output format |
| `includeProjectLicense` | boolean | Include project's own license |
| `includeLicenseScan` | boolean | Include scan results |
| `includeDependencySummary` | boolean | Include summary table |
| `includeDirectDependencies` | boolean | Include direct deps |
| `includeDeepDependencies` | boolean | Include transitive deps |
| `includeLicenseList` | boolean | Include license inventory |
| `includeCopyrightList` | boolean | Include copyright notices |

**dependencyInfoOptions array values:**
`Library`, `License`, `CustomTextLicense`, `OtherLicenses`, `Authors`, `Description`, `FullTextLicense`, `Source`, `ProjectUrl`, `PackageDownloadUrl`, `DependencyPaths`, `IssueNotes`

### Email PDF/TXT reports
```
GET https://app.fossa.com/api/revisions/{revision-locator}/attribution/email?format=PDF
```

Reports are delivered to the authenticated user's email address.

---

## SBOM import and export enables standards compliance

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/components/signed_url` | Get upload URL for SBOM |
| POST | `/api/components/build?fileType=sbom` | Trigger SBOM processing |

### Import SBOM (Step 1: Get signed URL)
```
GET https://app.fossa.com/api/components/signed_url?packageSpec={name}&revision={version}&fileType=sbom
```

### Import SBOM (Step 2: Upload and trigger build)
```
POST https://app.fossa.com/api/components/build?fileType=sbom
Content-Type: application/json
```

**Request Body:**
```json
{
  "selectedTeams": [],
  "archives": [{
    "packageSpec": "Project Name",
    "revision": "1.0.0",
    "fileType": "sbom"
  }]
}
```

**Supported formats:** CycloneDX v1.2-1.6 (JSON/XML), SPDX 2.2+ (JSON/XML)

---

## Scan and build status endpoints track analysis progress

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/cli/{locator}/latest_build` | Get latest build status |
| POST | `/api/builds` | Upload build/analysis data |

### Check build status
```
GET https://app.fossa.com/api/cli/{project-locator}/latest_build
```

**Response:**
```json
{
  "ID": 12345,
  "Error": null,
  "Task": {
    "Status": "SUCCEEDED"
  }
}
```

Status values: `CREATED`, `ASSIGNED`, `RUNNING`, `SUCCEEDED`, `FAILED`

---

## Complete endpoint reference table

| Category | Method | Endpoint | Auth Required |
|----------|--------|----------|---------------|
| **Projects** | GET | `/api/v2/projects` | Full Access |
| | GET | `/api/projects/{locator}` | Full Access |
| | PUT | `/api/projects/{locator}` | Full Access |
| **Revisions** | GET | `/api/revisions?projectId={locator}` | Full Access |
| | GET | `/api/revisions/{revision-locator}` | Full Access |
| **Dependencies** | GET | `/api/revisions/{locator}/dependencies` | Full Access |
| | GET | `/api/revisions/{locator}/parent_projects` | Full Access |
| **Issues** | GET | `/api/v2/issues` | Full Access |
| | GET | `/api/v2/issues/categories` | Full Access |
| | GET | `/api/v2/issues/statuses` | Full Access |
| **Vulnerabilities** | POST | `/api/vulns/by-locator` | Full Access |
| | GET | `/api/vulns/{id}/revisions/{rev}/remediation-guidance` | Full Access |
| **Reports** | GET | `/api/revisions/{locator}/attribution/download` | Full Access |
| | GET | `/api/revisions/{locator}/attribution/email` | Full Access |
| | GET | `/api/revisions/{locator}/attribution/json` | Full Access |
| **SBOM** | GET | `/api/components/signed_url` | Full Access |
| | POST | `/api/components/build?fileType=sbom` | Full Access |
| **Build Status** | GET | `/api/cli/{locator}/latest_build` | Full Access |

## Implementation notes for dashboard builders

The **recommended data flow** for a compliance dashboard: First, call `/api/v2/projects` to list all projects. For each project, use the `last_analyzed_revision` field to call `/api/revisions/{locator}/dependencies` for the component inventory. Aggregate compliance status using `/api/v2/issues/categories` at global scope for the overview, then drill into `/api/v2/issues` with project scope for details. Generate reports on-demand via the attribution endpoints.

**Rate limiting** is not explicitly documented, but enterprise customers should contact FOSSA support for guidance on high-volume API usage. The **SwaggerHub OpenAPI spec** at `https://app.swaggerhub.com/apis-docs/FOSSA1/App/0.3.8` provides additional endpoint details, though access may be restricted.
