# FOSSA Integration Workflow & Architecture

## System Overview

Your compliance dashboard integrates with FOSSA through a dedicated **Scanning Microservice** that orchestrates scan triggers, result collection, and policy evaluation storage.

```mermaid
flowchart TB
    subgraph "Compliance Dashboard Platform"
        UI[Dashboard UI]
        API[Dashboard API Gateway]
        
        subgraph "Microservices"
            SCAN[Scanning Service<br/>FOSSA/Snyk]
            PROJ[Projects Service]
            POLICY[Policy Service]
        end
        
        DB[(Compliance DB)]
    end
    
    subgraph "External Scanners"
        FOSSA[FOSSA Cloud]
        SNYK[Snyk<br/>Future]
    end
    
    subgraph "Source Control"
        GH[GitHub/GitLab]
        CI[CI/CD Pipelines]
    end
    
    UI --> API
    API --> SCAN
    API --> PROJ
    API --> POLICY
    
    SCAN <--> FOSSA
    SCAN <-.-> SNYK
    
    CI -->|Webhook| SCAN
    GH -->|Repository Events| CI
    
    SCAN --> DB
    PROJ --> DB
    POLICY --> DB
```

---

## Scan Triggering Mechanisms

There are **three primary ways** to trigger FOSSA scans in your architecture:

```mermaid
flowchart LR
    subgraph "Trigger Sources"
        T1[🔄 CI/CD Pipeline<br/>On commit/PR]
        T2[📅 Scheduled Job<br/>Cron-based]
        T3[👆 Manual Request<br/>Dashboard UI]
    end
    
    subgraph "Scanning Service"
        Q[Scan Queue]
        W[Worker Pool]
    end
    
    subgraph "Execution"
        CLI[FOSSA CLI<br/>Container]
        API_CALL[FOSSA API<br/>Direct Call]
    end
    
    T1 --> Q
    T2 --> Q
    T3 --> Q
    
    Q --> W
    W --> CLI
    W --> API_CALL
```

### Trigger Type Comparison

| Trigger | Best For | Implementation |
|---------|----------|----------------|
| **CI/CD Webhook** | Real-time on every commit | GitHub Actions / GitLab CI calls your Scanning Service API |
| **Scheduled** | Nightly compliance reports | Kubernetes CronJob or internal scheduler |
| **On-Demand** | Release verification | Dashboard button → API → Scan Queue |

---

## Complete Scan Workflow Sequence

This diagram shows the full lifecycle from scan request to dashboard display:

```mermaid
sequenceDiagram
    autonumber
    participant User as Dashboard User
    participant UI as Dashboard UI
    participant API as API Gateway
    participant ScanSvc as Scanning Service
    participant Queue as Message Queue
    participant Worker as Scan Worker
    participant FOSSA as FOSSA Cloud
    participant DB as Compliance DB

    User->>UI: View Project Release
    UI->>API: GET /projects/{id}/compliance-status
    API->>DB: Query latest scan results
    
    alt No recent scan exists
        API-->>UI: Status: "Scan Required"
        User->>UI: Click "Run Scan"
        UI->>API: POST /scans/trigger
        API->>ScanSvc: Request scan for microservices
        
        loop For each microservice in project
            ScanSvc->>Queue: Enqueue scan job
        end
        
        ScanSvc-->>API: Scan jobs queued
        API-->>UI: Status: "Scanning..."
    end

    Note over Worker,FOSSA: Async Processing

    Worker->>Queue: Dequeue job
    Worker->>Worker: Clone repository
    Worker->>FOSSA: fossa analyze --project X --revision Y
    FOSSA-->>Worker: Analysis accepted (async)
    
    Worker->>FOSSA: fossa test --json --timeout 600
    
    loop Poll until complete
        FOSSA-->>Worker: Pending...
    end
    
    FOSSA-->>Worker: Results JSON
    
    Worker->>FOSSA: GET /api/v2/issues?scope=project
    FOSSA-->>Worker: Policy violations
    
    Worker->>DB: Store scan results
    Worker->>DB: Store issues/violations
    Worker->>Queue: Publish "scan.completed" event
    
    Queue-->>UI: WebSocket notification
    UI->>API: GET /projects/{id}/compliance-status
    API->>DB: Query updated results
    DB-->>API: Latest scan data
    API-->>UI: Full compliance report
    UI-->>User: Display results
```

---

## Detailed: How Analysis Happens

```mermaid
flowchart TB
    subgraph "1. Scan Initiation"
        REQ[Scan Request]
        VAL{Validate<br/>Request}
        REPO[Fetch Repo<br/>Metadata]
    end
    
    subgraph "2. FOSSA CLI Execution"
        CLONE[Clone Repository]
        ANALYZE[fossa analyze<br/>--project --revision]
        UPLOAD[Upload to<br/>FOSSA Cloud]
    end
    
    subgraph "3. FOSSA Processing"
        DEPS[Dependency<br/>Resolution]
        LIC[License<br/>Detection]
        VULN[Vulnerability<br/>Matching]
        POL[Policy<br/>Evaluation]
    end
    
    subgraph "4. Results Collection"
        TEST[fossa test --json]
        ISSUES[GET /api/v2/issues]
        SBOM[fossa report<br/>--format cyclonedx]
    end
    
    subgraph "5. Storage"
        STORE_SCAN[Store Scan Record]
        STORE_DEP[Store Dependencies]
        STORE_ISS[Store Issues]
        STORE_SBOM[Store SBOM]
    end
    
    REQ --> VAL
    VAL -->|Valid| REPO
    VAL -->|Invalid| REQ
    
    REPO --> CLONE
    CLONE --> ANALYZE
    ANALYZE --> UPLOAD
    
    UPLOAD --> DEPS
    DEPS --> LIC
    LIC --> VULN
    VULN --> POL
    
    POL --> TEST
    TEST --> ISSUES
    ISSUES --> SBOM
    
    SBOM --> STORE_SCAN
    STORE_SCAN --> STORE_DEP
    STORE_DEP --> STORE_ISS
    STORE_ISS --> STORE_SBOM
```

---

## Scan Worker Implementation Flow

```mermaid
stateDiagram-v2
    [*] --> Queued: Job Created
    
    Queued --> Initializing: Worker picks up
    
    Initializing --> CloneRepo: Fetch source
    CloneRepo --> Analyzing: Run fossa analyze
    
    Analyzing --> WaitingForFossa: Upload complete
    
    WaitingForFossa --> Polling: Check status
    Polling --> WaitingForFossa: Still processing
    Polling --> FetchingResults: Analysis complete
    
    FetchingResults --> EvaluatingPolicy: Get issues
    EvaluatingPolicy --> StoringResults: Compare policies
    
    StoringResults --> Completed: All saved
    StoringResults --> Failed: Storage error
    
    Analyzing --> Failed: CLI error
    CloneRepo --> Failed: Clone error
    Polling --> TimedOut: Exceeded timeout
    
    Completed --> [*]
    Failed --> [*]
    TimedOut --> [*]
```

---

## Entity-Relationship Diagram for Scanning Microservice

This schema supports **multiple scanner backends** (FOSSA now, Snyk later):

```mermaid
erDiagram
    SCANNER_PROVIDER {
        uuid id PK
        string name "FOSSA, Snyk, etc"
        string api_base_url
        string provider_type "LICENSE, VULNERABILITY, BOTH"
        boolean is_active
        timestamp created_at
    }
    
    SCANNER_CREDENTIAL {
        uuid id PK
        uuid provider_id FK
        string credential_type "API_KEY, OAUTH"
        string encrypted_value
        string scope "PUSH_ONLY, FULL_ACCESS"
        timestamp expires_at
        timestamp created_at
    }
    
    SCAN_TARGET {
        uuid id PK
        uuid microservice_id FK "From Projects Service"
        uuid provider_id FK
        string external_project_id "FOSSA project locator"
        string repository_url
        string default_branch
        jsonb provider_config "Custom FOSSA settings"
        timestamp created_at
    }
    
    SCAN_REQUEST {
        uuid id PK
        uuid scan_target_id FK
        string trigger_type "CI_CD, SCHEDULED, MANUAL"
        string triggered_by "user_id or system"
        string revision "git commit SHA"
        string branch
        string status "QUEUED, RUNNING, COMPLETED, FAILED"
        timestamp requested_at
        timestamp started_at
        timestamp completed_at
    }
    
    SCAN_RESULT {
        uuid id PK
        uuid scan_request_id FK
        string provider_scan_id "FOSSA revision locator"
        string overall_status "PASSED, FAILED, WARNING"
        integer total_dependencies
        integer direct_dependencies
        integer policy_violations
        integer vulnerabilities_critical
        integer vulnerabilities_high
        integer vulnerabilities_medium
        integer vulnerabilities_low
        jsonb raw_response "Full FOSSA JSON"
        timestamp analyzed_at
    }
    
    DEPENDENCY {
        uuid id PK
        uuid scan_result_id FK
        string package_manager "npm, pip, maven"
        string package_name
        string version
        string locator "FOSSA locator format"
        boolean is_direct
        string license_expression "MIT, Apache-2.0, etc"
        timestamp created_at
    }
    
    ISSUE {
        uuid id PK
        uuid scan_result_id FK
        uuid dependency_id FK "nullable"
        string issue_type "LICENSE, VULNERABILITY, QUALITY"
        string severity "CRITICAL, HIGH, MEDIUM, LOW"
        string category "policy_conflict, cve, outdated"
        string external_issue_id "FOSSA issue ID"
        string title
        text description
        string status "ACTIVE, IGNORED, RESOLVED"
        string resolution_reason "nullable"
        timestamp detected_at
        timestamp resolved_at
    }
    
    POLICY_SNAPSHOT {
        uuid id PK
        uuid scan_target_id FK
        string policy_name
        string policy_version
        jsonb policy_rules "Cached from FOSSA or internal"
        string source "FOSSA, INTERNAL"
        boolean is_active
        timestamp synced_at
    }
    
    SBOM_EXPORT {
        uuid id PK
        uuid scan_result_id FK
        string format "CYCLONEDX, SPDX, CSV"
        string file_path "S3 or storage path"
        integer file_size_bytes
        string checksum_sha256
        timestamp generated_at
    }

    SCANNER_PROVIDER ||--o{ SCANNER_CREDENTIAL : "has"
    SCANNER_PROVIDER ||--o{ SCAN_TARGET : "scans via"
    SCAN_TARGET ||--o{ SCAN_REQUEST : "triggers"
    SCAN_TARGET ||--o{ POLICY_SNAPSHOT : "uses"
    SCAN_REQUEST ||--|| SCAN_RESULT : "produces"
    SCAN_RESULT ||--o{ DEPENDENCY : "contains"
    SCAN_RESULT ||--o{ ISSUE : "reports"
    SCAN_RESULT ||--o{ SBOM_EXPORT : "generates"
    DEPENDENCY ||--o{ ISSUE : "has"
```

---

## Data Flow: From FOSSA to Dashboard

```mermaid
flowchart LR
    subgraph "FOSSA Cloud"
        F_PROJ[Projects]
        F_REV[Revisions]
        F_ISS[Issues]
        F_DEP[Dependencies]
    end
    
    subgraph "Scanning Service"
        SYNC[Sync Worker]
        TRANS[Transformer]
        VALID[Validator]
    end
    
    subgraph "Compliance DB"
        DB_SCAN[scan_results]
        DB_DEP[dependencies]
        DB_ISS[issues]
    end
    
    subgraph "Dashboard"
        PROJ_VIEW[Project View]
        REL_VIEW[Release View]
        COMP_VIEW[Compliance Report]
    end
    
    F_PROJ -->|GET /projects| SYNC
    F_REV -->|GET /revisions| SYNC
    F_ISS -->|GET /issues| SYNC
    F_DEP -->|GET /dependencies| SYNC
    
    SYNC --> TRANS
    TRANS --> VALID
    
    VALID --> DB_SCAN
    VALID --> DB_DEP
    VALID --> DB_ISS
    
    DB_SCAN --> PROJ_VIEW
    DB_DEP --> REL_VIEW
    DB_ISS --> COMP_VIEW
```

---

## API Contract: Scanning Service Endpoints

```mermaid
flowchart TB
    subgraph "Scanning Service API"
        subgraph "Scan Management"
            POST_SCAN[POST /scans<br/>Trigger new scan]
            GET_SCAN[GET /scans/:id<br/>Get scan status]
            GET_SCANS[GET /scans?target_id=X<br/>List scans for target]
        end
        
        subgraph "Results"
            GET_RESULT[GET /scans/:id/result<br/>Full scan result]
            GET_DEPS[GET /scans/:id/dependencies<br/>Dependency list]
            GET_ISSUES[GET /scans/:id/issues<br/>Policy violations]
            GET_SBOM[GET /scans/:id/sbom<br/>Download SBOM]
        end
        
        subgraph "Configuration"
            POST_TARGET[POST /targets<br/>Register scan target]
            PUT_TARGET[PUT /targets/:id<br/>Update target config]
            POST_PROVIDER[POST /providers<br/>Add scanner provider]
        end
    end
    
    subgraph "Consumers"
        DASH[Dashboard Service]
        CI[CI/CD Webhooks]
        CRON[Scheduler]
    end
    
    DASH --> GET_RESULT
    DASH --> GET_ISSUES
    CI --> POST_SCAN
    CRON --> POST_SCAN
```

---

## Integration Timeline: When Things Happen

```mermaid
gantt
    title Scan Lifecycle Timeline
    dateFormat HH:mm:ss
    axisFormat %H:%M:%S
    
    section Trigger
    Receive webhook/request     :t1, 00:00:00, 1s
    Validate & queue            :t2, after t1, 1s
    
    section Preparation
    Clone repository            :t3, after t2, 5s
    Install dependencies        :t4, after t3, 30s
    
    section FOSSA Analysis
    Run fossa analyze           :t5, after t4, 10s
    Upload to FOSSA             :t6, after t5, 5s
    FOSSA processing            :crit, t7, after t6, 120s
    
    section Results
    Poll for completion         :t8, after t6, 120s
    Fetch issues via API        :t9, after t7, 5s
    Generate SBOM               :t10, after t9, 3s
    
    section Storage
    Store results in DB         :t11, after t10, 2s
    Notify dashboard            :t12, after t11, 1s
```

---

## Putting It Together: Complete Request Flow

```mermaid
sequenceDiagram
    autonumber
    participant CI as CI/CD Pipeline
    participant GW as API Gateway
    participant SS as Scanning Service
    participant Q as Redis Queue
    participant W as Worker
    participant FOSSA as FOSSA CLI/API
    participant DB as PostgreSQL
    participant PS as Projects Service
    participant WS as WebSocket

    Note over CI,WS: Phase 1: Trigger Scan
    CI->>GW: POST /api/scans {microservice_id, revision}
    GW->>SS: Forward request
    SS->>PS: GET /microservices/{id}
    PS-->>SS: Microservice details + repo URL
    SS->>DB: INSERT scan_request (status=QUEUED)
    SS->>Q: LPUSH scan_queue {job_payload}
    SS-->>GW: {scan_id, status: "QUEUED"}
    GW-->>CI: 202 Accepted

    Note over W,FOSSA: Phase 2: Execute Scan
    W->>Q: BRPOP scan_queue
    W->>DB: UPDATE scan_request SET status=RUNNING
    W->>W: git clone {repo_url}
    W->>FOSSA: fossa analyze --project X --revision Y
    FOSSA-->>W: Upload successful
    
    W->>FOSSA: fossa test --json --timeout 600
    loop Until complete or timeout
        FOSSA-->>W: {status: "pending"}
        W->>W: sleep 10s
    end
    FOSSA-->>W: {issues: [...], passed: false}

    Note over W,DB: Phase 3: Collect & Store
    W->>FOSSA: GET /api/v2/issues?scope[id]=X
    FOSSA-->>W: {issues: [...]}
    W->>FOSSA: GET /api/revisions/X/dependencies
    FOSSA-->>W: {dependencies: [...]}
    
    W->>DB: INSERT scan_result
    W->>DB: INSERT dependencies (batch)
    W->>DB: INSERT issues (batch)
    W->>DB: UPDATE scan_request SET status=COMPLETED
    
    Note over W,WS: Phase 4: Notify
    W->>Q: PUBLISH scan.completed {scan_id}
    Q-->>WS: Event broadcast
    WS-->>CI: Notification (if subscribed)
```

---

## Summary: Key Integration Points

| Component | Responsibility | FOSSA Interaction |
|-----------|----------------|-------------------|
| **Scan Queue** | Job management | None (internal) |
| **Scan Worker** | Execution | CLI: `analyze`, `test`, `report` |
| **Results Collector** | Data fetching | API: `/issues`, `/dependencies` |
| **Policy Engine** | Rule comparison | API: Policy sync or internal DB |
| **Storage Layer** | Persistence | None (internal) |
| **Notification Service** | Real-time updates | Webhook receiver |

This architecture allows you to:
1. **Trigger scans** from CI/CD, schedules, or dashboard UI
2. **Isolate FOSSA logic** in a dedicated microservice
3. **Store results locally** for fast dashboard queries
4. **Add Snyk later** by implementing the same `ScannerProvider` interface
5. **Scale workers** independently based on scan volume