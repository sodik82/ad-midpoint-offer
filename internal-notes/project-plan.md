# Project Plan — AD to midPoint Password Sync

**Version:** 0.2
**Date:** 2026-03-08
**Status:** Internal draft

---

## Scope Summary

Deliver a production-ready, open-source (EUPL 1.2) solution that captures AD password changes on all Domain Controllers and propagates them to midPoint via REST. The solution consists of a C/C++ LSA Password Filter DLL, a C# Windows Service, a WiX MSI installer, a CI/CD pipeline, and full documentation. Validated on Windows Server 2016, 2019, 2022, and 2025.

**Out of scope for v1:** OIDC/OAuth authentication, code signing of binaries, FIPS-140 compliance, centralised proxy/aggregator for multi-forest.

---

## Epic 1 — Project Setup & Repository

Foundation: repository structure, licensing, and toolchain configuration before any implementation begins.

### E1-S1: Initialize Repository Structure
Create the full folder hierarchy as defined in the architecture design (`src/filter-dll/`, `src/sync-service/`, `src/installer/`, `tests/`, `tools/mock-midpoint/`, `docker/`, `docs/`, `.github/workflows/`). Add `.gitignore` covering MSVC build outputs, .NET `bin/obj`, WiX intermediates, and docker volume data. Add `.editorconfig` enforcing consistent indentation across C++, C#, WiX XML, and YAML files.

### E1-S2: EUPL 1.2 Licensing Setup
Add the full EUPL 1.2 text as `LICENSE` in the repository root. Create `NOTICE` listing third-party dependencies and their licences (Serilog: Apache 2.0; Google Test: BSD-3; WiX Toolset: MS-RL — verify compatibility). Create a reusable EUPL 1.2 file-header template (comment block) for use in all `.cpp`, `.h`, and `.cs` files. Document the header requirement in the contributing section of the developer guide.

### E1-S3: README and Badges
Create `README.md` with: project overview, architecture summary diagram link, quick-start installation command, EUPL 1.2 licence badge, supported OS versions badge, and link to the admin guide. Keep it concise — detailed documentation goes in `docs/`.

### E1-S4: Dev Environment Documentation (Prerequisite Collection)
Document the exact prerequisites for the local development environment: Visual Studio 2022 (or Build Tools) with Desktop C++ workload, .NET SDK 8.0+, WiX Toolset v4 CLI, CMake 3.21+, Git, Docker Desktop (for the real midPoint instance). Obtain from the customer the midPoint resource definition XML files and schema configurations needed to configure the local real midPoint instance (per specification section 7).

### E1-S5: docker-compose Setup for Real midPoint
Create `docker/docker-compose.yml` that spins up a real midPoint instance for local development and E2E testing. Import the customer-provided midPoint resource definition XML files and schema configurations into the container on startup (via mounted init directory or a startup script). Document the `docker compose up` workflow in the build guide, including: how to reset midPoint state, how to verify the `notifyChange` endpoint is reachable, and how to inspect midPoint audit logs to confirm events were received.

---

## Epic 2 — Password Filter DLL (C/C++)

The `lsass.exe`-hosted LSA password filter. Minimal, deterministic, no network I/O.

### E2-S1: CMake Project Setup
Create `src/filter-dll/CMakeLists.txt`. Configure for 64-bit MSVC build. Set required compiler flags: `/GS` (buffer security check), `/DYNAMICBASE` (ASLR), `/NXCOMPAT` (DEP), minimal CRT dependency (`/MT` or `/MD` as appropriate). Export the three required LSA symbols (`InitializeChangeNotify`, `PasswordFilter`, `PasswordChangeNotify`). Produce `AdMidpointFilter.dll`.

### E2-S2: LSA Interface Implementation
Implement `dllmain.cpp`: `DllMain` with no-op `DLL_PROCESS_ATTACH`/`DETACH`. Implement `InitializeChangeNotify()`: initialise internal state (queue directory path from registry or compile-time constant), return `TRUE`. Implement `PasswordFilter()`: always return `TRUE` — password complexity filtering is not in scope. Add EUPL 1.2 headers.

### E2-S3: PasswordChangeNotify Core Logic
Implement `password_notify.cpp`: build the JSON event payload in memory (`eventId` as UUID via `UuidCreate`, `timestamp` as ISO-8601 via `GetSystemTimeAsFileTime`, `domain` via `GetComputerNameEx(ComputerNameNetBIOS)`, `username`, `password`). Use RAII wrappers for all heap allocations to prevent password leakage on early exit. Call `SecureZeroMemory` on all in-memory password buffers immediately after the encrypted blob is written. Return `STATUS_SUCCESS` unconditionally — never block the LSA.

### E2-S4: DPAPI Encryption (DLL Side)
Implement `queue_writer.cpp` DPAPI section: call `CryptProtectData` with `CRYPTPROTECT_LOCAL_MACHINE` flag so the machine-scoped key is used (enabling the separately-running Windows Service to decrypt). Handle `CryptProtectData` failure gracefully — log to Event Log and return without writing a file (password is not persisted in plaintext under any condition).

### E2-S5: Atomic Queue File Write
Implement the atomic write in `queue_writer.cpp`: write the encrypted blob to `queue\<timestamp>-<uuid>.tmp`, then rename to `queue\<timestamp>-<uuid>.evt` via `MoveFileEx`. Only `.evt` files are consumed by the service, making partial writes invisible. Handle all failure cases (disk full, permissions error) by logging to Windows Application Event Log and returning without crashing.

### E2-S6: Windows Event Log Error Reporting
Implement `event_log.cpp`: thin wrapper around `ReportEvent` for writing structured error messages to the Windows Application Event Log. Used only for DLL-side failures (queue write error, DPAPI failure). No password values are ever included in Event Log messages.

---

## Epic 3 — Password Sync Service (C#)

The C# Windows Service that owns all smart logic: queue consumption, DPAPI decryption, retry, TLS, REST delivery, and logging.

### E3-S1: Service Project Scaffolding
Create `src/sync-service/AdMidpointSync.csproj` targeting `net8.0-windows`, published as a self-contained single-file executable for `win-x64`. Implement `Program.cs` as a Windows Service host using `IHostedService` / `BackgroundService`. Implement `Worker.cs` (`OnStart`, `OnStop`, `OnShutdown`): load config, initialise logging, start `FileSystemWatcher`, start polling timer, signal cancellation on stop, drain in-flight events with a 10-second grace period. Add EUPL 1.2 headers to all `.cs` files.

### E3-S2: Configuration Model and Loader
Implement `Configuration.cs`: strongly-typed model for all `config.json` parameters (see design section 3.5: `midPointUrl`, `midPointUsername`, `midPointPassword`, `queueDirectory`, `maxQueueDepth`, `retryTimeoutSeconds`, `retryIntervalSeconds`, `threadPoolSize`, `tlsSkipVerification`, `logDirectory`, `logLevel`, `logRetentionDays`). Implement loader: read and deserialise `config.json` on startup. Implement validator: refuse to start with a structured error log if any required field is missing or invalid. Implement `FileSystemWatcher`-based live reload for non-sensitive fields (`logLevel`, `threadPoolSize`) — credential changes require a service restart.

### E3-S3: DPAPI Helper
Implement `DpapiHelper.cs`: wrapper around `System.Security.Cryptography.ProtectedData` with `DataProtectionScope.LocalMachine`. Expose `Encrypt(byte[])` → `string` (Base64) and `Decrypt(string)` → `byte[]`. Used for: decrypting event file blobs written by the DLL, and decrypting the `midPointUsername`/`midPointPassword` from `config.json`.

### E3-S4: Queue Reader
Implement `QueueReader.cs`: watch `queueDirectory` with `FileSystemWatcher` for new `*.evt` files. Start a polling timer (default 30-second interval) to scan the directory and catch files missed under high load or after `FileSystemWatcher` overflow events. For each detected file, check current queue depth against `maxQueueDepth` — if exceeded, log a Warning and skip the file (it remains in the queue for the next poll). Otherwise, hand the file path to the thread-pool gate. Ensure exclusive file open (`FileShare.None`) to prevent concurrent processing of the same file.

### E3-S5: Retry Engine
Implement `RetryEngine.cs`: per-event state machine. On HTTP 2xx: delete event file, log Information. On transient failure (network error, HTTP 5xx, timeout): compute next retry time as `retryIntervalSeconds * 2^attempt` (capped at a maximum interval), re-queue with delay. If `now - event.timestamp > retryTimeoutSeconds`: move file to `failed\`, log Error with `eventId`, `username`, `domain`, and event age — never log the password. On HTTP 4xx (permanent failure, e.g., 401 Unauthorized): move file to `failed\`, log Error with status code details.

### E3-S6: Thread Pool Gate
Implement `SemaphoreSlim`-gated `Task` dispatch in `Worker.cs` or a dedicated `TaskDispatcher.cs`. Maximum concurrency controlled by `threadPoolSize` config value. Support live reload of `threadPoolSize` without service restart.

### E3-S7: midPoint REST Client
Implement `MidPointClient.cs`: `HttpClient` with `SocketsHttpHandler` for connection pooling. TLS policy: detect OS version at startup via `Environment.OSVersion`; on WS 2022+ enforce TLS 1.3; on WS 2016/2019 allow TLS 1.2 as the highest supported protocol. If `tlsSkipVerification=true`: set `ServerCertificateCustomValidationCallback` to always return `true` and log a startup Warning. Authentication: `Authorization: Basic <base64(user:pass)>` header with credentials decrypted from config at startup. Implement `PostNotifyChangeAsync(EventPayload)` that serialises the payload to the midPoint `notifyChange` REST request format. After each call (success or failure), overwrite the in-memory password byte array.

### E3-S8: Structured Logging (Serilog)
Implement `Logging/LoggingConfiguration.cs`: configure Serilog in code (no `appsettings.json`, no ASP.NET dependency). Default sink: rolling file under `logDirectory`, one file per day, retention `logRetentionDays`. Output format: structured JSON by default; switchable to human-readable template via config. Log all event types defined in the design (section 3.6) at their specified levels. Enforce that no logging call site ever receives a password value — enforced by code review checklist item.

---

## Epic 4 — Mock midPoint Tool

Lightweight mock REST endpoint used exclusively in CI integration tests where spinning up a real midPoint container is not practical. Local development and E2E testing use the real midPoint instance from E1-S5 instead.

### E4-S1: Mock midPoint REST Endpoint (CI Integration Tests Only)
Implement `tools/mock-midpoint/MockMidPoint.csproj`: minimal ASP.NET Core stub (or WireMock.NET runner) that exposes the `notifyChange` endpoint. Configurable via command-line args to return HTTP 200, 500, or 401, enabling scenario-driven fault injection in the CI integration test pipeline. Log all received requests (excluding password values) to stdout. Document usage in the build guide, clearly noting its CI-only scope.

---

## Epic 5 — MSI Installer (WiX v4)

Single `.msi` package for all supported OS versions, supporting silent deployment via GPO/SCCM.

### E5-S1: WiX v4 Project Setup
Create `src/installer/AdMidpointSync.wixproj`. Configure WiX v4 build (CLI `wix build`). Define product version tied to a build-time property so the MSI version matches the Git tag. Add `MajorUpgrade` element for in-place upgrades.

### E5-S2: Directory Tree and NTFS ACL Setup
WiX component to create `C:\ProgramData\AdMidpointSync\` with subdirectories `queue\`, `failed\`, `logs\`. Set NTFS ACLs as defined in the design: `SYSTEM` — Full Control; service account — Modify on `queue\`, `failed\`, `logs\`, Read on `config.json`; `Administrators` — Full Control; `Everyone` — No Access.

### E5-S3: DLL Installation and Registry Registration
Install `AdMidpointFilter.dll` to `C:\Windows\System32\`. Register the DLL by **appending** its name to `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Notification Packages` (not overwriting — multi-filter coexistence must be preserved). On uninstall, remove only the DLL's own entry from the `Notification Packages` value.

### E5-S4: Windows Service Installation
Install `AdMidpointSync.exe` to `C:\Program Files\AdMidpointSync\`. Register and configure the Windows Service (`ServiceInstall` WiX element): `start=auto`, `type=own`, running as the dedicated low-privilege service account. Start the service at end of installation. Stop the service at uninstall/upgrade start; restart at upgrade completion.

### E5-S5: Default Config File and Upgrade Preservation
Write a default `config.json` template to `C:\ProgramData\AdMidpointSync\config.json` on fresh installation only (use WiX component condition `NOT Installed`). On upgrade, preserve the existing `config.json`, `queue\`, `failed\`, and `logs\` directories — explicitly exclude them from the component removal. On uninstall, do not remove `C:\ProgramData\AdMidpointSync\` — document the manual purge procedure in the admin guide.

### E5-S6: DPAPI Credential Encryption Custom Action
Implement `CustomActions/ConfigAction.cs`: managed WiX custom action that reads `MIDPOINT_URL`, `MIDPOINT_USER`, and `MIDPOINT_PASS` MSI properties, DPAPI-encrypts the username and password values (machine scope), and writes them into `config.json`. This action runs after the service account is created so DPAPI encryption uses the correct machine context. If MSI properties are not provided, skip the action and document the post-install manual configuration step.

### E5-S7: Silent Install Support
Ensure all relevant configuration is settable via MSI public properties on the `msiexec` command line: `MIDPOINT_URL`, `MIDPOINT_USER`, `MIDPOINT_PASS`, `QUEUE_DIR`, `LOG_LEVEL`. Document the full silent install command in the admin guide with GPO/SCCM deployment examples.

### E5-S8: OS Version Detection
Add WiX conditions on `VersionNT` for Windows Server 2016 (1607), 2019 (1809), 2022 (2009), 2025 (2025). Apply version-specific registry or behaviour differences. Block installation on unsupported OS versions with a clear user-facing error message.

---

## Epic 6 — CI/CD Pipeline

Automated build, test, and release pipeline on GitHub Actions.

### E6-S1: Build Pipeline (`build.yml`)
Trigger: push to `main`, pull requests to `main`. Runner: `windows-2022`. Steps: checkout with full history; CMake configure + MSVC build of the DLL (Release and Debug); `dotnet publish` of the service as self-contained `win-x64` single-file EXE; run C++ unit tests via `ctest --output-on-failure`; run C# unit tests via `dotnet test` with TRX output for the GitHub Actions test reporter; WiX v4 CLI build of the MSI; archive DLL, EXE, and MSI as pipeline artifacts.

### E6-S2: Integration Test Pipeline (`integration-test.yml`)
Trigger: push to `main`, pull requests (optionally scheduled nightly). Runner: `windows-2022`. Steps: start the mock midPoint tool; install the service against the mock endpoint; run the integration test suite (`dotnet test` — integration category): happy path, transient failure (mock returns 500), permanent failure (mock returns 401), retry exhaustion (short `retryTimeoutSeconds`), midPoint unavailable then recovers; collect logs as artifacts on failure.

### E6-S3: Release Pipeline (`release.yml`)
Trigger: Git tag matching `v*` (e.g., `v1.0.0`). Steps: run (or reuse) the full build pipeline; extract the changelog section for the tagged version; create a GitHub Release with the MSI as a binary asset and the changelog as the release body; publish versioned artifacts.

---

## Epic 7 — Automated Testing

Unit and integration test suites covering all components.

### E7-S1: C++ Unit Tests (Google Test)
Create `tests/filter-dll-tests/` with Google Test suite. Test cases: queue file created with correct `<timestamp>-<uuid>.evt` name format; atomic write via `.tmp` → `.evt` rename (verify partial `.tmp` files are not picked up by the service); DPAPI machine-scope encrypt/decrypt round-trip (verify the decrypted payload matches the original JSON); error path — disk-full simulation: function returns success, no crash, Event Log entry written; error path — permissions error: same behaviour.

### E7-S2: C# Unit Tests (xUnit)
Create `tests/sync-service-tests/`. Test cases: **RetryEngine** — verify exponential backoff intervals, verify drop-on-timeout moves file to `failed\`, verify 4xx triggers permanent failure move; **MidPointClient** — mock `HttpMessageHandler`, verify correct `Authorization: Basic` header, correct TLS policy per OS version, `tlsSkipVerification` behaviour; **Configuration** — valid config loads correctly, missing required field causes startup refusal with structured error, DPAPI credential round-trip; **DpapiHelper** — encrypt then decrypt returns original bytes; **QueueReader** — `FileSystemWatcher` event triggers processing, polling fallback scans directory and picks up files; **Thread Pool Gate** — `maxDegreeOfParallelism` is respected under concurrent load.

### E7-S3: Integration Tests
Create integration test category within `tests/sync-service-tests/`. Scenarios: happy path — inject `.evt` file → verify mock midPoint received the expected `notifyChange` POST with correct username and domain; transient failure — mock returns 500 → verify retry with exponential backoff; permanent failure — mock returns 401 → verify event moved to `failed\` without retry; retry exhaustion — configure short `retryTimeoutSeconds` → verify event moved to `failed\` after timeout; midPoint unavailable then recovers — stop mock, inject event, restart mock → verify eventual delivery.

### E7-S4: E2E PowerShell Scripts (Against Real midPoint)
Create `tests/e2e/`: `Invoke-HappyPath.ps1` — force a password change for a test AD account, wait, verify the real midPoint (docker-compose instance) received `notifyChange` with the correct username by querying midPoint audit logs or a dedicated verification endpoint; `Invoke-BurstTest.ps1` — bulk-set `pwdLastSet=0` on 1000 test accounts, trigger password changes, monitor queue depth and delivery count against real midPoint; `helpers/StartMidPoint.ps1` — helper to start/stop/reset the docker-compose midPoint stack from PowerShell before and after E2E runs.

---

## Epic 8 — Manual and Acceptance Testing

Non-automated test scenarios that require real Windows Server VM environments and manual execution.

### E8-S1: OS Compatibility Test Execution
Set up 4 pre-configured VM images with AD DS role promoted (or use Vagrant/Packer snapshots): Windows Server 2016 (TLS 1.2 fallback path), 2019, 2022, 2025. On each: install the MSI, change a password, verify delivery, verify logs. Specifically test TLS 1.2 fallback on WS 2016/2019. Record results per OS version. This story is a gate before release.

### E8-S2: Windows Defender Compatibility Test
With Windows Defender real-time protection enabled, install the MSI on a WS 2022 VM. Verify no quarantine or security alert is raised for the DLL loading into `lsass.exe`. Document the results. If an exclusion is required, add it to the admin guide and note that registered LSA filters are expected by Defender.

### E8-S3: Burst Load Test (1000 Passwords)
Run `Invoke-BurstTest.ps1` against a real or simulated DC environment: bulk-trigger 1000 password changes, monitor queue depth does not exceed `maxQueueDepth`, verify all events are eventually delivered or logged, verify no events are silently dropped. Document results for customer acceptance.

### E8-S4: Upgrade and Uninstall Scenarios
Install v1 MSI, change a password (event enters queue), install v2 MSI (upgrade): verify queue events survive the upgrade and are delivered after the service restarts. Uninstall: verify `C:\ProgramData\AdMidpointSync\` directory and its contents are preserved. Document results.

### E8-S5: Security Review Checklist
Execute a structured checklist before release: confirm no password value appears in any log file or Windows Event Log entry; confirm DPAPI encryption succeeds from the `lsass.exe` process context and is decryptable from the service context; confirm TLS 1.3 is enforced on WS 2022+ and TLS 1.2 on WS 2016/2019; confirm `tlsSkipVerification=false` in the default `config.json`; confirm service runs as a least-privilege account; confirm `config.json` ACL restricts read access.

---

## Epic 9 — Documentation

All written deliverables required by the specification.

### E9-S1: Administrator Guide (`docs/admin-guide.md`)
Cover: installation (MSI parameters, silent install examples for GPO and SCCM); configuration (every `config.json` parameter with type, default, valid range, and example); credential rotation (how to update the midPoint password after initial setup — re-run installer or manual DPAPI re-encrypt procedure); log interpretation (what each log event type means, how to correlate `eventId` across DLL Event Log entries and service log files); troubleshooting (DLL not loaded, service not starting, midPoint unreachable, queue growing, `failed\` accumulating); checking status (`reg query` for Notification Packages, `sc query` for service status, Process Explorer); purge procedure for complete post-uninstall cleanup.

### E9-S2: Build and Developer Guide (`docs/build-guide.md`)
Cover: prerequisites (exact versions of VS 2022, .NET SDK, WiX v4, CMake, Git, Docker Desktop); local build commands (step-by-step `cmake`, `dotnet build`, `wix build`); local dev environment options (Option A: real AD DC VM + real midPoint via docker-compose — full stack; Option B: real midPoint via docker-compose only, no DC — service-level development without DLL); running the real midPoint stack (`docker compose up`, reset procedure, verifying the `notifyChange` endpoint); running tests locally (`ctest`, `dotnet test`); running E2E scripts against the docker-compose midPoint instance; running CI integration tests against the mock (fault-injection scenarios); contributing (branch naming, PR process, code style, mandatory EUPL 1.2 header for new files).

### E9-S3: Support Proposal (Separate Document)
Prepare a separate quotation document for 12 months of post-deployment maintenance as required by specification section 8. Cover: security patches for discovered vulnerabilities; Windows Server update compatibility testing (new Windows Update releases, future WS versions); bug fixes with defined SLA response times; scope boundaries (what is and is not covered). Deliver as a separate commercial document, not part of the open-source repository.

### E9-S4: EUPL 1.2 Header Compliance Pass
Before release, verify that every `.cpp`, `.h`, and `.cs` source file in the repository contains the correct EUPL 1.2 header comment. Add any missing headers. Verify the `NOTICE` file accurately reflects all third-party dependencies and their licences after the full dependency list is finalised.

---

## Epic 10 — Release

Final packaging and handover.

### E10-S1: Changelog and Version Management
Maintain `CHANGELOG.md` throughout development following Keep a Changelog conventions. Before release, complete the `[1.0.0]` section with all significant changes, security notes, and known limitations. Version the MSI and service EXE from the Git tag (`v1.0.0`).

### E10-S2: v1.0.0 Release
Tag `v1.0.0` in Git, triggering the release pipeline. Verify the GitHub Release is created with: the signed (or unsigned, per v1 scope) MSI binary; the full changelog section for 1.0.0; links to the admin guide and build guide. Confirm the repository is public and the EUPL 1.2 licence is correctly displayed on GitHub. Hand over the repository URL and the MSI download link to the customer.

---

## Summary Table

| Epic | Story | Brief Description |
|---|---|---|
| **E1 — Project Setup** | E1-S1 | Initialize repository folder structure, `.gitignore`, `.editorconfig` |
| | E1-S2 | Add EUPL 1.2 LICENSE, NOTICE, and file-header template |
| | E1-S3 | Create README.md with overview, badges, quick start |
| | E1-S4 | Document dev prerequisites; obtain midPoint XML configs from customer |
| | E1-S5 | docker-compose setup for real midPoint (local dev and E2E) |
| **E2 — Filter DLL** | E2-S1 | CMake project setup with MSVC hardening flags |
| | E2-S2 | Implement DllMain and three LSA exports |
| | E2-S3 | PasswordChangeNotify: JSON payload build, SecureZeroMemory, RAII |
| | E2-S4 | DPAPI machine-scope encryption of event payload |
| | E2-S5 | Atomic queue file write (`.tmp` → `.evt` rename) |
| | E2-S6 | Windows Application Event Log error reporting wrapper |
| **E3 — Sync Service** | E3-S1 | C# Windows Service scaffolding: host, Worker, graceful shutdown |
| | E3-S2 | Configuration model, loader, validator, live reload |
| | E3-S3 | DPAPI helper (ProtectedData encrypt/decrypt, machine scope) |
| | E3-S4 | Queue reader: FileSystemWatcher + polling fallback + maxQueueDepth guard |
| | E3-S5 | Retry engine: exponential backoff, drop-on-timeout, permanent failure |
| | E3-S6 | Thread pool gate: SemaphoreSlim, configurable parallelism |
| | E3-S7 | MidPoint REST client: HttpClient, Basic Auth, TLS 1.3/1.2 policy |
| | E3-S8 | Serilog structured logging: rolling file, JSON format, configurable level |
| **E4 — Mock midPoint** | E4-S1 | Mock REST endpoint for CI integration tests only (fault injection) |
| **E5 — MSI Installer** | E5-S1 | WiX v4 project setup, MajorUpgrade configuration |
| | E5-S2 | ProgramData directory tree with correct NTFS ACLs |
| | E5-S3 | DLL install to System32; append-safe Notification Packages registry write |
| | E5-S4 | Service install, registration, start/stop lifecycle in MSI |
| | E5-S5 | Default config.json on fresh install; preserve on upgrade/uninstall |
| | E5-S6 | Custom action: DPAPI-encrypt credentials from MSI properties into config |
| | E5-S7 | Silent install MSI properties (MIDPOINT_URL, USER, PASS, etc.) |
| | E5-S8 | OS version detection (WiX VersionNT conditions, WS 2016–2025) |
| **E6 — CI/CD** | E6-S1 | build.yml: DLL + service build, unit tests, MSI, artifact archive |
| | E6-S2 | integration-test.yml: mock midPoint, service deployment, test scenarios |
| | E6-S3 | release.yml: GitHub Release on v* tag with MSI and changelog |
| **E7 — Automated Tests** | E7-S1 | C++ Google Test suite: queue writer, DPAPI, atomic rename, error paths |
| | E7-S2 | C# xUnit suite: retry engine, REST client, config, DPAPI, queue reader |
| | E7-S3 | Integration tests: happy path, transient/permanent failure, retry exhaustion |
| | E7-S4 | E2E PowerShell scripts against real midPoint (docker-compose): happy path, burst test |
| **E8 — Manual Testing** | E8-S1 | OS compatibility test on WS 2016, 2019, 2022, 2025 VMs |
| | E8-S2 | Windows Defender compatibility test |
| | E8-S3 | Burst load acceptance test (1000 simultaneous password changes) |
| | E8-S4 | Upgrade and uninstall scenario test |
| | E8-S5 | Security review checklist (no passwords in logs, DPAPI context, TLS) |
| **E9 — Documentation** | E9-S1 | Administrator Guide: install, config, credential rotation, troubleshooting |
| | E9-S2 | Build and Developer Guide: prerequisites, build steps, docker-compose midPoint, contributing |
| | E9-S3 | Support Proposal: 12-month post-deployment maintenance quotation |
| | E9-S4 | EUPL 1.2 header compliance pass across all source files |
| **E10 — Release** | E10-S1 | Changelog finalisation and version management |
| | E10-S2 | v1.0.0 GitHub Release: MSI, changelog, repository handover to customer |
