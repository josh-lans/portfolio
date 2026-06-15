# MonLite — Enterprise Infrastructure Observability Platform

> **Note**: This is a portfolio showcase of architecture, methodology, and design decisions. The full source code is proprietary.

## What is MonLite?

MonLite is an enterprise-grade infrastructure monitoring platform I designed and built from scratch. It provides real-time observability across SAP systems, databases, Linux/Windows hosts, and web services — all from a single pane of glass with AI-powered diagnostics. Now with **High Availability** (automatic failover in 5-30 seconds) and **one-command deployment** for both single-server and distributed HA setups.

**Built in 6 weeks** using AI-assisted development (Claude Code + structured session methodology). Currently at **v3.1.0** with 170+ development sessions.

## Demo

[**Quick 2-minute overview**](https://www.youtube.com/watch?v=8yCcPyOEzzk) · [**Full 7-minute walkthrough**](https://www.youtube.com/watch?v=Qudo0nftRvQ)

## Screenshots

| Dashboard | Connections & Hierarchy |
|-----------|------------------------|
| ![Dashboard](screenshots/dashboard.png) | ![Connections](screenshots/connections.png) |

| Alarm Management | Threshold Configuration |
|-----------------|------------------------|
| ![Alarms](screenshots/alarms.png) | ![Thresholds](screenshots/thresholds.png) |

| Visualizations | Runbooks |
|---------------|----------|
| ![Visualizations](screenshots/visualizations.png) | ![Runbooks](screenshots/runbooks.png) |

---

## Platform Capabilities

### Server Monitoring
- **Linux** (SSH) and **Windows** (WinRM/SNMP) host monitoring
- CPU, memory, disk, swap, load average, network, process-level metrics
- Filesystem capacity projections with days-to-full prediction
- Auto-discovery of filesystems and drives
- ICMP/TCP availability with configurable ping intervals

### Database Monitoring
Seven production-ready database connectors with real-time metrics, backup tracking, auto-discovery, and per-engine threshold profiles:

| Engine | Metrics |
|--------|---------|
| **Microsoft SQL Server** | CPU, memory, I/O, PLE, deadlocks, connections, batch requests, log space, backup age/status |
| **Oracle** | Tablespaces, SGA/PGA, sessions, wait events, redo log, datafile I/O |
| **SAP HANA** | Sessions, blocking, deadlocks, backup age, memory/CPU/storage, cache hit ratio, replication lag (3-strategy connection cascade for Docker/NAT) |
| **IBM DB2** | Bufferpool hit ratio, lock waits, log utilization, connections, tablespace usage |
| **SAP/Sybase ASE** | CPU, memory, I/O, deadlocks, connections, tempdb usage |
| **SAP MaxDB** | Cache hit ratio, sessions, lock waits, data/log area, backup status |
| **PostgreSQL** | Connections, cache hit ratio, transaction rates, table bloat, replication lag |

### SAP Application Monitoring
Two connector types with a Unified mode that auto-selects the best check source:

**SAPControl (SOAP API):**
- Process health (disp+work, icman, gwrd, enserver)
- Work process utilization by type (DIA, BGD, UPD, SPO, ENQ)
- Dispatcher queue depth, enqueue locks/errors, ICM threads
- Parameter compliance (security baseline monitoring)
- Auto-discovery via SSH/WinRM from managed hosts

**ABAP RFC (19 Surveillance Categories):**

| Category | SAP Source | Key Details |
|----------|-----------|-------------|
| Work Processes | TH_WPINFO | Per-type thresholds, runtime tracking, PRIV detection |
| Jobs | TBTCO | Duration/delay/error detection, top offenders |
| Short Dumps | /SDF/GET_DUMP_LOG | Error ID classification, aggregate + individual alerting |
| Locks | ENQUEUE_READ | Lock age tracking, orphan detection, table-level filtering |
| Transports | TPALOG | Return code filtering (RC>=8), time-windowed import history |
| IDocs | EDIDC | Error/waiting status, direction-aware (inbound/outbound) |
| RFC Queues | TRFC_QIN/QOUT_OVERVIEW | Point-in-time queue snapshots, error detection |
| Spool | TSP01 | Error status monitoring, time-windowed |
| Certificates | SSF_ALERT_CERTEXPIRE | PSE certificate expiry via INST_EXECUTE_REPORT |
| Change Settings | TMW + T000 | SE06/SCC4 drift detection with full audit history trail |
| Parameter Compliance | PAHI | Security baseline monitoring (login/*, rdisp/*, icf/*) |
| SAPconnect | SOOS | Email/fax delivery status (SCOT/SOST equivalent) |
| Batch Input | APQI | Session status monitoring |
| Application Logs | BALHDR | SLG1 equivalent with severity filtering |
| Audit Logs | RSAU_READ_LOG | SM20 equivalent, security event detection |
| Process Chains (RDA) | RSPCLOGCHAIN | BW/BI chain execution with auto-detection |
| RFC Destinations | RFCDES + RFC_PING | Destination availability testing |
| Update Requests | VHDR | SM13 equivalent, error detection |
| Dispatcher Queues | TH_DPINFO | Queue depth monitoring |

### Web Service Monitoring
- HTTP/HTTPS endpoint availability and response time
- 6 authentication types (Basic, Bearer, OAuth2, API Key, custom headers, none)
- SSL certificate expiry tracking
- Routed through collectors for full network reach behind firewalls

### AI Diagnostics — Luna
- Air-gapped LLM assistant using Ollama/Qwen locally (no data leaves the firewall)
- Cloud LLM fallback (Gemini, OpenAI, Anthropic) for enhanced analysis
- RAG pipeline indexing 27 help documentation pages
- Direct database context queries (real-time SAP status, job history, change audit)
- Natural-language infrastructure queries: *"Is NPL open for changes?"* *"What failed jobs ran on PRD today?"*
- Anti-hallucination validation against live metrics

### High Availability & Deployment
- **PostgreSQL advisory lock leader election** — automatic failover in 5-30 seconds
- **Worker gating architecture** — 13 background workers run on all nodes, check `is_leader()` each cycle
- **Zero-downtime cascade updates** — one-click rolling upgrade across CP nodes
- **Install scripts** — single-server (`install.sh`), HA database (`db-install.sh`), HA control plane (`cp-install.sh`), load balancer (`lb-install.sh`)
- **Docker support** — single-server Compose, HA profiles, Docker Swarm multi-host
- **Release package builder** — `tools/build-release.sh` generates customer-ready `.tar.gz` for air-gapped deployments

### Enterprise Controls
- **Check enable/disable hierarchy** — Global → Profile → Company → Platform → System → Host → Connector
- **CLI management** — `monlitectl` for Docker management, HA cluster control (`ha status`, `ha failover`), offline updates, and release packaging

---

## Architecture

### Single Server
```
┌─ CONTROL PLANE ────────────────────────────────────────────┐
│  Browser UI ──HTTPS──► FastAPI + React 19 + Luna AI        │
│                        Alarm engine + Scheduler             │
│                        PostgreSQL 16 + Ollama (optional)    │
└────────────────────────────────────────────────────────────┘
```

### HA Deployment
```
                    ┌─────────────────────┐
                    │   PostgreSQL (DB)    │
                    └──────────┬──────────┘
                ┌──────────────┴──────────────┐
        ┌───────▼───────┐           ┌────────▼────────┐
        │  CP1 (LEADER)  │◄─ lock ──►│  CP2 (STANDBY)  │
        │  13 workers    │  failover │  13 workers     │
        │  ACTIVE        │   5-30s   │  GATED (idle)   │
        └───────┬────────┘           └────────┬────────┘
                └──────────────┬───────────────┘
                        ┌──────▼──────┐
                        │  nginx LB   │
                        └──────┬──────┘
                    ┌──────────▼──────────┐
                    │  Collector Agents    │
                    └─────────────────────┘
```

### Collector Communication
```
  ┌──────────────┐    HTTPS/9443    ┌──────────────────────┐
  │  Collector    │ ─────────────► │  /api/collector/*      │
  │  Agents       │  push metrics,  │  heartbeat, metrics,  │
  │  (Docker or   │  WS checks,     │  system info, pair,   │
  │   systemd)    │  DB checks,     │  ws-checks, db-checks │
  │               │  SAP RFC/SOAP   │  results              │
  └──────────────┘                  └──────────────────────┘
```

## Technical Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | Python 3.12, FastAPI, SQLAlchemy 2.0, Pydantic v2, Uvicorn |
| **Frontend** | React 19, Vite 7, Tailwind CSS 4, ApexCharts, D3.js |
| **Database** | PostgreSQL 16 (TIMESTAMPTZ, enforced SSL in production) |
| **AI** | Ollama/Qwen (local), Gemini, OpenAI, Anthropic (configurable) |
| **Auth** | Microsoft Entra ID / Okta SSO (OIDC + PKCE), LDAP, Argon2, RBAC |
| **Security** | Fernet encryption, HMAC audit logs, CSP headers, CSRF, DOMPurify XSS |
| **Integrations** | ServiceNow (CMDB + Event Mgmt), Elasticsearch/Kibana, Webhooks |
| **Deployment** | Docker Compose or bare-metal (systemd), CLI tools (monlitectl) |

---

## Enterprise Security & Compliance

MonLite has undergone an **8-phase security hardening** and **independent multi-AI verification audits** using Claude, OpenAI Codex, Gemini, and GitHub Copilot — providing cross-model validation of security posture:

- **OWASP Top 10 coverage**: CSP headers, DOMPurify XSS sanitization, CORS hardening, SSRF fail-closed, SQL injection prevention via SQLAlchemy ORM
- **Session security**: HttpOnly cookies (no localStorage tokens), CSRF double-submit, configurable inactivity timeout, concurrent session limits, session write throttling
- **Data protection**: Fernet encryption (AES-128-CBC + HMAC) for credentials at rest, HMAC-signed audit logs, enforced PostgreSQL SSL in production
- **Access control**: RBAC (System Admin / Engineer Admin / Engineer / Viewer), company-level multi-tenant isolation, team-based role elevation, ITAR-compatible
- **Authentication**: Local accounts + LDAP + OIDC SSO (Okta, Entra ID) with JIT provisioning, group-to-role mapping, Force SSO mode, IdP-managed MFA
- **Rate limiting**: Fail-closed in production on login, password reset, SSO, and API key generation endpoints
- **Audit trail**: Every action logged with HMAC integrity verification, expandable detail, Luna session review

---

## Scalability

Designed for **3,000+ monitored endpoints** with **200+ concurrent users**:

- ETag/304 response caching across all API endpoints
- SQL subquery RBAC filters (no post-fetch filtering)
- Paginated responses with configurable limits
- 90-day metric retention with hourly rollups (nightly cleanup)
- Distributed collector agents with parallel ThreadPoolExecutor (10 DB + 15 WS workers)
- Remote Push Update from the UI (no SSH into collectors)
- Time-windowed SAP RFC queries to prevent WP exhaustion on production systems

---

## AI-Assisted Development Methodology

One of the most distinctive aspects of this project is the development methodology. I used multiple AI coding assistants — primarily **Claude Code** (Anthropic) across 160+ structured sessions, with **OpenAI Codex**, **Gemini**, and **GitHub Copilot** for independent security and architecture verification.

### The `.claude-docs` System

I developed a structured documentation system that maintains project context across AI sessions:

- **MEMORY.md** — Essential working context, environment details, anti-shortcut protocols
- **STANDARDS.md** — 50+ mandatory development patterns with section references
- **DEPENDENCIES.md** — Change impact checklists (e.g., "adding a new metric requires 7 wiring steps")
- **SESSION_LOG.md** — Detailed session history for context continuity
- **LESSONS.md** — Learned patterns to prevent repeated mistakes

See the **[agentic-dev-kit](https://github.com/josh-lans/agentic-dev-kit)** repo for the full methodology and the drop-in templates.

### Multi-AI Verification

Critical features underwent independent review across multiple AI platforms:
- **Security audits**: Codex P0 audit covering OWASP, SOC2, GDPR, ISO27001 compliance
- **Architecture review**: Cross-model validation of alarm engine, surveillance rules, and collection scheduling
- **Scalability analysis**: Load testing patterns and connection pool tuning verified across Claude, Gemini, and Copilot

### Key Insight

The value wasn't "AI wrote my code." The value was:
1. **Domain expertise** (12 years of SAP Basis) provided the *what* and *why*
2. **AI accelerated** the *how* — translating operational knowledge into production code
3. **Structured methodology** maintained quality at speed
4. **Cross-model verification** caught blind spots no single AI would find

---

## Design Philosophy

- **Progressive disclosure** — Simple by default, power features revealed on expand
- **Domain terminology preserved** — SAP Basis admins see SM50/SM37/ST22 concepts, DBAs see familiar metrics
- **Glassmorphism UI** — Premium feel with semi-transparent cards, backdrop blur, starfield animation, dark/light modes
- **Air-gapped first** — Luna AI works fully offline; cloud is optional enhancement
- **Scale-aware** — Every query is bounded: time-windowed, row-limited, row-count capped
- **Collector-side efficiency** — Keep RFC calls short, release work processes quickly, cache negative results

---

## Additional Documentation

### Architecture
- [`architecture/PLATFORM_ARCHITECTURE.md`](architecture/PLATFORM_ARCHITECTURE.md) — Full platform architecture: backend, frontend, collectors, alarm engine, scalability patterns
- [`architecture/SAP_MONITORING_OVERVIEW.md`](architecture/SAP_MONITORING_OVERVIEW.md) — SAP connector design, surveillance rules engine, auto-detection patterns
- [`architecture/DATABASE_MONITORING.md`](architecture/DATABASE_MONITORING.md) — 7-engine database monitoring: driver interface, HANA NAT cascade, MaxDB JVM bridge
- [`architecture/SECURITY_AND_COMPLIANCE.md`](architecture/SECURITY_AND_COMPLIANCE.md) — 8-phase security hardening, multi-AI audit, RBAC, SSO, encryption, compliance readiness

### Methodology & Templates
- **[agentic-dev-kit](https://github.com/josh-lans/agentic-dev-kit)** — the full multi-agent development methodology (the "Conductor" pattern) plus drop-in `.claude-docs` templates (MEMORY, STANDARDS, DEPENDENCIES, SESSION_PROTOCOL, MULTI_AGENT_PROTOCOL, and more).

---

## Contact

**Joshua Lans** — joshlans@me.com — [LinkedIn](https://linkedin.com/in/joshlans)
