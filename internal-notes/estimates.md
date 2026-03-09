# Estimates — AD to midPoint Password Sync

**Version:** 1.0
**Date:** 2026-03-09
**Assumptions:** Mix of mid-to-senior developers. Fibonacci scale: 0, 1, 2, 3, 5, 8, 13, 21 man-days. Epics exceeding 13 days are split into smaller parts for planning.

---

## Epic Summary

Epics with a raw estimate above 13 days are split into logical parts. Each part's estimate is a Fibonacci number.

| Epic | Part | Stories Included | Estimate (man-days) | Notes |
|---|---|---|---|---|
| **E1 — Project Setup** | — | E1-S1 … E1-S5 | **8** | Includes customer coordination for midPoint XML config (E1-S4) and docker-compose midPoint setup (E1-S5) |
| **E2 — Filter DLL** | A — Infrastructure | E2-S1, E2-S2, E2-S6 | **5** | CMake setup, LSA exports skeleton, Event Log wrapper |
| | B — Core Implementation | E2-S3, E2-S4, E2-S5 | **8** | Security-critical: JSON payload + RAII, DPAPI from lsass context, atomic queue write |
| **E3 — Sync Service** | A — Foundation | E3-S1, E3-S2, E3-S3, E3-S8 | **8** | Service scaffold, config model + live reload, DPAPI helper, Serilog setup |
| | B — Core Logic | E3-S4, E3-S5, E3-S6, E3-S7 | **13** | Queue reader, retry state machine (complex), thread pool gate, REST client with dual TLS policy |
| **E4 — Mock midPoint** | — | E4-S1 | **2** | ASP.NET Core stub with fault injection for CI only |
| **E5 — MSI Installer** | A — Structure & Installation | E5-S1, E5-S2, E5-S3, E5-S4 | **8** | WiX project, NTFS ACLs, append-safe registry write, service lifecycle |
| | B — Config & Extras | E5-S5, E5-S6, E5-S7, E5-S8 | **8** | Config upgrade preservation, managed DPAPI custom action (complex), silent install, OS version conditions |
| **E6 — CI/CD Pipeline** | — | E6-S1, E6-S2, E6-S3 | **8** | Build + unit test + MSI pipeline, integration test pipeline with mock, release pipeline |
| **E7 — Automated Testing** | A — Unit Tests | E7-S1, E7-S2 | **8** | Google Test suite (C++ DLL incl. DPAPI + disk-full simulation), xUnit suite (C# — retry, REST client, config, queue) |
| | B — Integration & E2E | E7-S3, E7-S4 | **5** | Integration test scenarios (happy path, fault injection, exhaustion), E2E PowerShell against real midPoint |
| **E8 — Manual Testing** | — | E8-S1 … E8-S5 | **8** | OS compatibility on 4 VM images, Defender test, burst load, upgrade/uninstall, security checklist |
| **E9 — Documentation** | — | E9-S1 … E9-S4 | **8** | Admin guide, build/dev guide, support proposal, EUPL compliance pass |
| **E10 — Release** | — | E10-S1, E10-S2 | **2** | Changelog, v1.0.0 GitHub Release, repository handover |
| | | **Total** | **99** | |

---

## Detailed Story Breakdown

### E1 — Project Setup & Repository

| Story | Description | Estimate (man-days) |
|---|---|---|
| E1-S1 | Initialize repository structure, `.gitignore`, `.editorconfig` | 1 |
| E1-S2 | EUPL 1.2 LICENSE, NOTICE, file-header template | 1 |
| E1-S3 | README.md with overview, badges, quick start | 1 |
| E1-S4 | Dev environment documentation + obtain midPoint XML configs from customer | 2 |
| E1-S5 | docker-compose setup for real midPoint (local dev and E2E) | 3 |
| **E1 Total** | | **8** |

### E2 — Password Filter DLL (C/C++)

| Story | Description | Estimate (man-days) |
|---|---|---|
| E2-S1 | CMake project setup, MSVC hardening flags, DLL exports | 1 |
| E2-S2 | DllMain, three LSA exports (`InitializeChangeNotify`, `PasswordFilter`, `PasswordChangeNotify` stub) | 2 |
| E2-S3 | `PasswordChangeNotify` core: JSON payload, RAII, `SecureZeroMemory` | 3 |
| E2-S4 | DPAPI machine-scope encryption (`CryptProtectData`, lsass context) | 2 |
| E2-S5 | Atomic queue file write (`.tmp` → `.evt` rename, error handling) | 2 |
| E2-S6 | Windows Application Event Log wrapper (`ReportEvent`) | 1 |
| **E2 Total** | | **11** → split as **5 + 8** |

### E3 — Password Sync Service (C#)

| Story | Description | Estimate (man-days) |
|---|---|---|
| E3-S1 | Windows Service scaffold: `IHostedService`, `Worker`, graceful shutdown (10 s drain) | 2 |
| E3-S2 | Configuration model, loader, validator, `FileSystemWatcher` live reload | 2 |
| E3-S3 | DPAPI helper (`ProtectedData`, `LocalMachine` scope, encrypt/decrypt) | 1 |
| E3-S4 | Queue reader: `FileSystemWatcher` + 30 s polling fallback + `maxQueueDepth` guard | 3 |
| E3-S5 | Retry engine: exponential backoff state machine, drop-on-timeout, permanent 4xx failure | 5 |
| E3-S6 | Thread pool gate: `SemaphoreSlim`-gated `Task` dispatch, live `threadPoolSize` reload | 2 |
| E3-S7 | midPoint REST client: `HttpClient`, Basic Auth, TLS 1.3 / TLS 1.2 policy per OS version | 3 |
| E3-S8 | Serilog structured logging: rolling file, JSON format, configurable level and retention | 2 |
| **E3 Total** | | **20** → split as **8 + 13** |

### E4 — Mock midPoint Tool

| Story | Description | Estimate (man-days) |
|---|---|---|
| E4-S1 | ASP.NET Core stub / WireMock.NET runner with configurable fault injection (200 / 500 / 401) | 2 |
| **E4 Total** | | **2** |

### E5 — MSI Installer (WiX v4)

| Story | Description | Estimate (man-days) |
|---|---|---|
| E5-S1 | WiX v4 project setup, `MajorUpgrade`, version from Git tag | 1 |
| E5-S2 | `ProgramData` directory tree creation with correct NTFS ACLs | 2 |
| E5-S3 | DLL install to `System32`; append-safe `Notification Packages` registry write and uninstall | 2 |
| E5-S4 | Windows Service registration, start/stop lifecycle, service account | 2 |
| E5-S5 | Default `config.json` on fresh install; preserve config / queue / logs on upgrade and uninstall | 2 |
| E5-S6 | Managed WiX custom action: DPAPI-encrypt credentials from MSI properties into `config.json` | 3 |
| E5-S7 | Silent install MSI public properties (`MIDPOINT_URL`, `USER`, `PASS`, etc.) | 1 |
| E5-S8 | WiX `VersionNT` conditions for WS 2016 / 2019 / 2022 / 2025, block unsupported OS | 1 |
| **E5 Total** | | **14** → split as **8 + 8** |

### E6 — CI/CD Pipeline

| Story | Description | Estimate (man-days) |
|---|---|---|
| E6-S1 | `build.yml`: DLL build (CMake/MSVC), service publish, unit tests, MSI build, artifact archive | 3 |
| E6-S2 | `integration-test.yml`: start mock, deploy service, run integration test scenarios, collect logs | 3 |
| E6-S3 | `release.yml`: GitHub Release on `v*` tag with MSI asset and changelog body | 2 |
| **E6 Total** | | **8** |

### E7 — Automated Testing

| Story | Description | Estimate (man-days) |
|---|---|---|
| E7-S1 | C++ Google Test suite: queue writer, atomic rename, DPAPI round-trip, disk-full simulation | 3 |
| E7-S2 | C# xUnit suite: retry engine, REST client (mock handler), config, DPAPI, queue reader, thread pool gate | 5 |
| E7-S3 | Integration tests: happy path, transient / permanent failure, retry exhaustion, recovery | 3 |
| E7-S4 | E2E PowerShell scripts against real midPoint docker-compose: happy path, burst test (1000 accounts) | 3 |
| **E7 Total** | | **14** → split as **8 + 5** |

### E8 — Manual & Acceptance Testing

| Story | Description | Estimate (man-days) |
|---|---|---|
| E8-S1 | OS compatibility test on WS 2016, 2019, 2022, 2025 VM images (incl. TLS 1.2 fallback on 2016/2019) | 3 |
| E8-S2 | Windows Defender real-time protection compatibility test; document result and any required exclusion | 1 |
| E8-S3 | Burst load acceptance test (1000 password changes); document results for customer | 1 |
| E8-S4 | Upgrade (v1→v2) and uninstall scenario test; verify queue survival and data directory preservation | 1 |
| E8-S5 | Security review checklist: no passwords in logs, DPAPI context, TLS enforcement, ACLs, service account | 2 |
| **E8 Total** | | **8** |

### E9 — Documentation

| Story | Description | Estimate (man-days) |
|---|---|---|
| E9-S1 | Administrator guide: install, config reference, credential rotation, log interpretation, troubleshooting | 3 |
| E9-S2 | Build and developer guide: prerequisites, build commands, docker-compose midPoint, contributing | 2 |
| E9-S3 | Support proposal: 12-month post-deployment maintenance quotation (separate commercial document) | 2 |
| E9-S4 | EUPL 1.2 header compliance pass across all source files; finalise `NOTICE` | 1 |
| **E9 Total** | | **8** |

### E10 — Release

| Story | Description | Estimate (man-days) |
|---|---|---|
| E10-S1 | `CHANGELOG.md` finalisation, version management (Git tag → MSI/EXE version) | 1 |
| E10-S2 | Tag `v1.0.0`, trigger release pipeline, verify GitHub Release, hand over to customer | 1 |
| **E10 Total** | | **2** |

---

## Estimation Notes

- **E2-S4 / E2-S5** (DPAPI from lsass + atomic write): security-critical path with no margin for bugs; estimate includes time for explicit testing of the `CRYPTPROTECT_LOCAL_MACHINE` constraint from the `lsass.exe` process context.
- **E3-S5** (Retry engine): the most complex single story — a per-event state machine with exponential backoff, timeout tracking, and two distinct failure paths. Estimate reflects the need for thorough unit testing.
- **E5-S6** (DPAPI custom action): managed WiX custom actions require careful DLL isolation and sequencing; DPAPI must run after the service account exists to use the correct machine key context.
- **E8-S1** (OS compatibility): assumes pre-built VM snapshots with AD DS already promoted; add 3–5 extra days if VM images must be built from scratch.
- **E9-S3** (Support proposal): commercial deliverable outside the open-source repo; 2 days covers drafting but does not include commercial negotiation or approval cycles.
