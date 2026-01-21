# FOSSA API Glossary

A comprehensive reference for all terminology used in the FOSSA API and compliance dashboard integration.

---

## Core Identifiers

### Locator

The **universal identifier** used throughout FOSSA to reference any package, project, or dependency. Locators follow a structured format that encodes the source, name, and optionally the version of a component.

**Format:** `{fetcher}+{package}${revision}`

**Components:**
| Part | Description | Example |
|------|-------------|---------|
| `fetcher` | The package manager or source type | `npm`, `mvn`, `git`, `custom` |
| `package` | The package name or path | `lodash`, `org.apache.commons:commons-lang3` |
| `revision` | Version identifier (optional) | `4.17.21`, `2024-01-15T10:30:00Z` |

**Examples:**
```
npm+lodash$4.17.21
mvn+org.apache.logging.log4j:log4j-core$2.17.1
git+github.com/facebook/react$v18.2.0
custom+27932/MyProject$2024-01-15T10:30:00Z
```

**URL Encoding:** When used in API URLs, locators must be URL-encoded:
- `+` → `%2B`
- `/` → `%2F`
- `$` → `%24`
- `:` → `%3A`

---

### Project Locator

A locator that identifies a **FOSSA project** (your analyzed codebase). Projects are containers for revisions and their associated scans.

**Format:** `{fetcher}+{org_id}/{project_name}` or `{fetcher}+{repository_path}`

**Examples:**
```
custom+27932/ComplianceDashboard      # Custom project
git+github.com/myorg/my-repo          # Git-integrated project
```

---

### Revision Locator

A locator that identifies a **specific version/scan** of a project. Revisions represent point-in-time snapshots of your project's dependency tree.

**Format:** `{project_locator}${revision_id}`

**Examples:**
```
custom+27932/MyProject$2024-01-15T10:30:00Z
git+github.com/myorg/repo$abc123def456
```

---

### Dependency Locator

A locator that identifies a **third-party package** detected in your project's dependency tree.

**Format:** `{fetcher}+{package_name}${version}`

**Examples:**
```
npm+express$4.18.2
pip+requests$2.31.0
gem+rails$7.1.2
nuget+Newtonsoft.Json$13.0.3
```

---

## Fetcher Types

### Fetcher

The **package manager or source system** from which a dependency originates. FOSSA supports 20+ fetchers.

| Fetcher | Package Ecosystem | Example Locator |
|---------|-------------------|-----------------|
| `npm` | Node.js/JavaScript | `npm+react$18.2.0` |
| `pip` | Python PyPI | `pip+django$4.2.0` |
| `mvn` | Java Maven | `mvn+com.google.guava:guava$32.1.0-jre` |
| `gem` | Ruby RubyGems | `gem+nokogiri$1.15.0` |
| `go` | Go modules | `go+github.com/gin-gonic/gin$v1.9.1` |
| `nuget` | .NET NuGet | `nuget+Serilog$3.1.1` |
| `cargo` | Rust Crates | `cargo+serde$1.0.193` |
| `composer` | PHP Packagist | `composer+laravel/framework$10.0.0` |
| `cocoapods` | iOS/macOS | `cocoapods+Alamofire$5.8.0` |
| `git` | Git repositories | `git+github.com/org/repo$v1.0.0` |
| `custom` | Manual/uploaded | `custom+27932/ProjectName$timestamp` |
| `archive` | Uploaded archives | `archive+27932/filename.tar.gz$hash` |

---

## Project Structure

### Project

A **container** representing a codebase or software component being analyzed. Projects hold multiple revisions and track compliance status over time.

**Properties:**
- `locator` - Unique project identifier
- `title` - Human-readable name
- `description` - Project description
- `url` - Source repository URL
- `public` - Visibility setting
- `policy_id` - Assigned compliance policy
- `last_analyzed_revision` - Most recent scan
- `teamProjects` - Team assignments

---

### Revision

A **point-in-time snapshot** of a project's dependency tree, typically corresponding to a build, commit, or release. Each scan creates a new revision.

**Properties:**
- `locator` - Unique revision identifier
- `project_locator` - Parent project
- `resolved` - Dependency resolution complete
- `dependency_count` - Number of dependencies
- `issue_count` - Number of detected issues
- `created_at` - Scan timestamp

**Relationship:** Project → has many → Revisions → has many → Dependencies

---

### Branch / Ref Type

A **version control reference** that groups related revisions. When querying projects with `ref_type=branch`, FOSSA returns revision references organized by branch name.

---

## Dependency Concepts

### Dependency

A **third-party package** that your project uses. Dependencies can be direct or transitive.

**Properties:**
- `locator` - Package identifier
- `loc` - Parsed locator object (`fetcher`, `package`, `revision`)
- `licenses` - Array of detected licenses
- `depth` - Direct or transitive
- `parents` - Dependencies that pull this in
- `unresolved_locators` - Dependencies that couldn't be resolved

---

### Direct Dependency

A dependency **explicitly declared** in your project's manifest file (package.json, pom.xml, requirements.txt, etc.). You directly control these.

---

### Transitive Dependency (Deep Dependency)

A dependency **pulled in by another dependency**. You don't declare these directly—they come from your direct dependencies' requirements.

**Example:** Your project requires `express` (direct), which requires `body-parser` (transitive), which requires `raw-body` (also transitive).

---

### Dependency Depth

Classification of how a dependency enters your project:

| Depth | Description | Filter Value |
|-------|-------------|--------------|
| Direct | Explicitly declared in manifest | `direct` |
| Deep/Transitive | Required by another dependency | `deep` |

---

### Parent Projects

Projects that **depend on a specific package**. Used for impact analysis—when a vulnerability is found, you can identify all affected projects.

**API:** `GET /api/revisions/{dependency-locator}/parent_projects`

---

### Unresolved Dependency

A dependency that FOSSA **could not fully analyze**, typically due to missing version information, private registries, or unsupported package formats.

---

## License Concepts

### License

The **legal terms** under which a dependency is distributed. FOSSA detects licenses from package metadata, LICENSE files, and source code analysis.

**Properties:**
- `licenseId` / `LicenseID` - SPDX identifier (e.g., "MIT", "Apache-2.0")
- `title` - Human-readable name
- `text` - Full license text
- `url` - Reference URL
- `attribution` - Required attribution text
- `copyright` - Copyright notice

---

### SPDX License ID

A **standardized identifier** from the Software Package Data Exchange specification. Examples: `MIT`, `Apache-2.0`, `GPL-3.0-only`, `BSD-3-Clause`.

**Reference:** https://spdx.org/licenses/

---

### License Expression

A **compound license statement** using SPDX syntax for packages with multiple licensing options.

**Operators:**
- `AND` - Must comply with both licenses
- `OR` - Can choose either license
- `WITH` - License with exception

**Examples:**
```
MIT OR Apache-2.0          # Choice between MIT and Apache
GPL-2.0-only WITH Classpath-exception-2.0
(MIT AND BSD-3-Clause)     # Must comply with both
```

---

### Custom License

A **user-defined license** created in your FOSSA organization for proprietary or uncommon licenses not in the SPDX database.

---

### License Scan

FOSSA's **analysis of source files** to detect licenses embedded in code comments, headers, and LICENSE files, beyond package metadata.

---

## Issue Types

### Issue

A **compliance problem** detected by FOSSA that requires attention. Issues are categorized and can be actively managed or ignored.

**Categories:**
- `licensing` - License compliance violations
- `vulnerability` - Security vulnerabilities
- `quality` - Code quality concerns

---

### Licensing Issue

A compliance violation related to **license terms or policies**.

| Type | Description |
|------|-------------|
| `policy_conflict` | Dependency license conflicts with policy |
| `policy_flag` | License flagged for review by policy |
| `unlicensed_dependency` | No license detected |

---

### Vulnerability Issue

A **known security flaw** (CVE) in a dependency.

**Severity Levels:**
| Level | CVSS Score | Description |
|-------|------------|-------------|
| `critical` | 9.0-10.0 | Immediate action required |
| `high` | 7.0-8.9 | Serious risk |
| `medium` | 4.0-6.9 | Moderate risk |
| `low` | 0.1-3.9 | Minor risk |
| `unknown` | N/A | Not yet scored |

---

### Quality Issue

A **code quality or maintenance concern** in a dependency.

| Type | Description |
|------|-------------|
| `outdated_dependency` | Newer version available |
| `blacklisted_dependency` | Dependency on blocklist |
| `risk_abandonware` | No recent maintenance |
| `risk_empty-package` | Package has no content |
| `risk_native-code` | Contains native/compiled code |

---

### Issue Status

The **current state** of an issue in your workflow:

| Status | Description |
|--------|-------------|
| `active` | Requires attention |
| `ignored` | Acknowledged and suppressed |
| `resolved` | Fixed in newer revision |

---

## Policy Concepts

### Policy

A **set of rules** that define which licenses and dependencies are acceptable in your organization.

**Properties:**
- `id` - Policy identifier
- `name` - Policy name
- `rules` - License approval/denial rules
- `projects` - Assigned projects

---

### Policy Conflict

An issue raised when a dependency's license **violates policy rules**—for example, using a GPL-licensed package when your policy requires permissive licenses only.

---

### Policy Flag

An issue raised when a license **requires manual review** per policy configuration, without being explicitly denied.

---

## Report & Export Concepts

### Attribution Report

A **compliance document** listing all dependencies, their licenses, and required attribution notices. Used for legal compliance and license fulfillment.

**Formats:** HTML, Markdown, CSV, PDF, TXT

**Sections:**
- Project license
- License scan results
- Dependency summary
- Direct dependencies
- Deep dependencies
- License list
- Copyright list

---

### SBOM (Software Bill of Materials)

A **machine-readable inventory** of all components in your software, following standard formats for supply chain transparency.

**Supported Formats:**
- CycloneDX (v1.2-1.6, JSON/XML)
- SPDX (v2.2+, JSON/XML)

---

## Build & Scan Concepts

### Build

A **FOSSA analysis job** that processes your project and creates a new revision with dependency and license data.

**Statuses:**
| Status | Description |
|--------|-------------|
| `CREATED` | Build queued |
| `ASSIGNED` | Build assigned to worker |
| `RUNNING` | Analysis in progress |
| `SUCCEEDED` | Completed successfully |
| `FAILED` | Analysis failed |

---

### Analysis / Scan

The **process of examining** your project's dependencies, licenses, and potential vulnerabilities. Triggered by CLI upload, CI/CD integration, or manual scan.

---

## Authentication Concepts

### API Token

A **credential** for authenticating API requests. Two types exist:

| Type | Permissions | Use Case |
|------|-------------|----------|
| Full Access | Read + Write all data | Dashboards, integrations |
| Push-Only | Upload builds only | CI/CD pipelines |

**Location:** Organization Settings → Integrations → API

---

### Organization ID (Org ID)

A **numeric identifier** for your FOSSA organization, embedded in custom project locators.

**Example:** In `custom+27932/MyProject`, `27932` is the org ID.

---

## Scope Concepts

### Global Scope

Query across **all projects** in your organization.

**API Parameter:** `scope[type]=global`

---

### Project Scope

Query within a **single project**.

**API Parameters:**
```
scope[type]=project
scope[id]={project-locator}
scope[revision]={revision-id}  # Optional
```

---

## API Response Concepts

### Pagination

Breaking large result sets into **pages** for efficient retrieval.

**Parameters:**
- `count` / `limit` - Items per page
- `offset` - Starting position
- `page` - Page number (1-indexed)

---

### Sorting

**Ordering** API results by specific fields.

**Common Values:**
- `severity_desc` - Highest severity first
- `created_at_desc` - Newest first
- `package_asc` - Alphabetical by package
- `title` - Alphabetical by name

---

## Quick Reference: Locator Patterns

| Entity | Pattern | Example |
|--------|---------|---------|
| NPM Package | `npm+{name}${version}` | `npm+express$4.18.2` |
| Maven Artifact | `mvn+{group}:{artifact}${version}` | `mvn+com.google.guava:guava$32.0.0-jre` |
| Python Package | `pip+{name}${version}` | `pip+django$4.2.0` |
| Go Module | `go+{module}${version}` | `go+github.com/gin-gonic/gin$v1.9.1` |
| Git Repository | `git+{host}/{path}${ref}` | `git+github.com/org/repo$main` |
| Custom Project | `custom+{org_id}/{name}${timestamp}` | `custom+27932/MyApp$2024-01-15T10:30:00Z` |

---

## Visual: Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                         ORGANIZATION                            │
│                         (org_id: 27932)                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ Policy  │ │ Policy  │ │  Team   │
              │    A    │ │    B    │ │         │
              └─────────┘ └─────────┘ └─────────┘
                    │           │           │
                    └─────┬─────┘           │
                          ▼                 │
┌─────────────────────────────────────────────────────────────────┐
│                          PROJECT                                │
│  locator: custom+27932/MyApp                                    │
│  policy_id: A                                                   │
│  last_analyzed_revision: 2024-01-15T10:30:00Z                   │
└─────────────────────────────────────────────────────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │   REVISION   │    │   REVISION   │    │   REVISION   │
    │  $2024-01-15 │    │  $2024-01-10 │    │  $2024-01-05 │
    │  (latest)    │    │              │    │              │
    └──────────────┘    └──────────────┘    └──────────────┘
            │
            ├─────────────────────────────────────────┐
            ▼                                         ▼
    ┌──────────────┐                         ┌──────────────┐
    │  DEPENDENCY  │ ────(transitive)────▶   │  DEPENDENCY  │
    │  (direct)    │                         │  (deep)      │
    │ npm+express  │                         │ npm+raw-body │
    └──────────────┘                         └──────────────┘
            │                                         │
            ▼                                         ▼
    ┌──────────────┐                         ┌──────────────┐
    │   LICENSE    │                         │   LICENSE    │
    │     MIT      │                         │     MIT      │
    └──────────────┘                         └──────────────┘
            │
            ▼
    ┌──────────────┐
    │    ISSUE     │
    │ (if policy   │
    │  conflict)   │
    └──────────────┘
```

---

## Common API Workflow

```
1. List Projects          GET /api/v2/projects
        │
        ▼
2. Get Latest Revision    (use last_analyzed_revision from project)
        │
        ▼
3. Fetch Dependencies     GET /api/revisions/{rev-locator}/dependencies
        │
        ▼
4. Check Issues           GET /api/v2/issues?scope[type]=project&scope[id]={locator}
        │
        ▼
5. Generate Report        GET /api/revisions/{rev-locator}/attribution/download
```
