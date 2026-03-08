# Initial Analysis Questions — AD to midPoint Password Sync

Questions identified after reviewing `official-input/specification.md`. These need answers before a reliable estimate can be produced.

---

## 1. Scope & Architecture Gaps

**Q1. How many Domain Controllers are in scope?**
The spec says the agent must be deployed on "all Domain Controllers" but gives no count. This affects testing effort, deployment complexity, and support scope significantly.

Answer: For integration purposes, we test with only two controllers in one installation.

**Q2. Single domain/forest or multi-domain/multi-forest?**
The solution's complexity changes substantially if multiple AD domains or forests are involved. Trusts, separate DC pools, and separate midPoint endpoints all need to be considered.

Answer: Prepare estimate alternatives for both scenarios

**Q3. What is the IPC mechanism between the DLL and the sending service?**
The spec describes a "local queue" as the hand-off point but does not define the technology: named pipes, memory-mapped files, a file-based queue, or a Windows message queue. This is a fundamental architectural decision with significant implementation impact and must be agreed upon early.

Answer: To be defined in design phase, there is no explicit requirement from customer.

**Q4. What happens when the local queue reaches the configurable maximum depth?**
Does the service silently drop new events, block (which would contradict the LSA non-blocking requirement), or raise an alert? The failure mode must be defined.

Answer: Drop new events but log the events so that admin can republish them.

**Q5. What happens to queued password events after the configurable retry timeout expires?**
Are they discarded silently, moved to a dead-letter location, or is an alert raised? This has security and compliance implications.

Answer: Same as above - drop and log.

**Q6. Which midPoint version is in production?**
The spec references the notifyChange REST API and acknowledges the data structure "may slightly change." Knowing the exact midPoint version (and planned upgrade path) is needed to target the correct API contract.

Asnwer: Assumption is that the API is stable.

**Q7. When will the midPoint resource XML and schema configurations be provided?**
Section 7 states the customer will provide these to replicate target API behavior. This is a hard dependency for integration development and testing. A delay here will block the project.

Answer: Assumption is that reference midPoint configuration is provided in the begining of the projects (first two weeks)

---

## 2. Authentication & Security

**Q8. Will authentication to midPoint use HTTP Basic Auth or OIDC?**
These are fundamentally different to implement. OIDC requires token lifecycle management (token refresh, expiry handling, potentially a client credentials flow), which is considerably more complex than Basic Auth. Which is preferred, and if OIDC, what is the identity provider?

Answer: For now we just assume Basic Auth.

**Q9. Is TLS certificate validation against an internal/private CA required?**
If midPoint uses a certificate issued by an internal enterprise CA, the sending service must trust that CA. Is there a custom CA certificate to bundle or configure? Is certificate pinning a requirement?

Answer: We assume that given OS host will trust midPoint CA. For development purposes we will have escape hatch (ignoring SSL) controllable by configuration file.

**Q10. Is FIPS-compliant cryptography required?**
Many regulated enterprise environments mandate FIPS 140-2/3 validated cryptographic modules. The choice of encryption algorithm for data-at-rest and the TLS stack must align with this if applicable.

Answer: We assume that there is no such requirement.

**Q11. Is code signing of the DLL and MSI required?**
LSA filter DLLs loaded into lsass.exe are closely scrutinized by EDR/AV solutions and Windows itself. A signed DLL and installer significantly reduce deployment friction and are often required by enterprise security policy. If yes — who provides the code-signing certificate?

Answer: We assume that there is no code signing involved.

**Q12. Is a formal security review, penetration test, or code audit required as a deliverable?**
This is not mentioned in the spec but is common for software handling credentials on Domain Controllers. If expected, it needs to be scoped and priced separately.

Answer: We assume that this is out of scope.

---

## 3. Compatibility & Environment

**Q13. Are there existing LSA notification packages already registered on the DCs?**
The installer must coexist with any pre-existing LSA password filter DLLs (e.g., from other IAM or PAM solutions). Knowing this upfront prevents integration conflicts during testing.

Answer: Let's assume that multiple filters can be registered.

**Q14. Is Windows Defender or any specific EDR product in use that must be validated against?**
DLLs injected into lsass.exe are a common detection target. Validating that the solution does not trigger security alerts in the customer's specific AV/EDR environment may require coordinated testing.

Answer: We will just test it against Windows Defender. Please estimate little buffer for testing but if code signing is needed, it will be extra effort out of the scope of initial estimate.

**Q15. Is "validated on Windows Server 2016, 2019, 2022, 2025" a customer-verified requirement, or vendor self-certified?**
Setting up and maintaining test environments for all four OS versions adds significant effort. Clarifying whether the customer expects evidence of testing on all versions (e.g., test reports) or only self-attestation affects the cost.

Answer: We will have to test it on those 4 environments - in design phase we will try to automize the installation but worst case we will propare 4 VM images with starting configuration.

**Q16. Is the TLS 1.2 fallback on Windows Server 2016/2019 acceptable to the customer's security/compliance team?**
The spec acknowledges this but does not confirm it has been cleared by the customer's information security policy. If TLS 1.3 is mandated by policy, Windows Server 2016/2019 support may need to be dropped or handled differently.

Answer: Yes, this is explicitly mentioned below - however it is little bit contracting initial requirement.

---

## 4. Deliverables Clarification

**Q17. Does the MSI need to support upgrade scenarios (version-to-version)?**
The spec covers initial installation but is silent on upgrading. A proper MSI upgrade (including migration of existing queued data and configuration) is non-trivial, especially when the LSA DLL is loaded into a running lsass.exe process.

Answer: We assume only simple upgrade scenarios - update of code (DLL or sender) with very simple versioned data (we always assume that sender is up to date)

**Q18. Does the MSI need to support clean uninstallation?**
Specifically: what should happen to queued, unsent password events and the encryption keys on uninstall? Should they be purged or preserved?

Answer: Yes, allow uninstaller to keep data on filesystem.

**Q19. Where should the Git repository be hosted?**
Should the vendor create and own the repository, or should it be transferred to the customer's own GitHub/GitLab organization? Who administers the repository post-delivery?

Answer: We will use vendor's git repository.

**Q20. Is the support proposal (12-month maintenance) in scope for this offer, or is it a separate document?**
Section 8 requests "a separate quotation." Should this be included as an addendum to the current offer or submitted independently?

Answer: All support will be as separate time&material after user acceptance. We still need to estimate user acceptance activities like fixing...

---

## 5. Post-Deployment Support

**Q21. What SLA terms are expected for the 12-month support period?**
Response time and resolution time targets (e.g., P1 critical issue: response within 4 hours, resolution within 24 hours) are needed to price the support engagement correctly.

Answer: Explicitly assuming that we just do next business day support.

**Q22. What is the scope of "Windows Server update compatibility"?**
Does this mean the vendor must proactively test and certify compatibility with every Patch Tuesday, every cumulative update, or only major feature/version releases? The level of proactive testing expected has a large cost impact.

Answer: Assuming we won't proactively test new Windows Server updates.

**Q23. What is the expected timeframe for delivering security patches?**
For example: critical CVEs patched within 30 days, other security issues within 90 days. Without defined SLOs the support pricing cannot be accurately scoped.

Answer: After acceptance, this will be part of support period. We don't want any special SLOs as it will be time&material (and best effort).

---

## 6. Testing & Acceptance

**Q24. What are the formal acceptance criteria?**
How does the customer define "done"? Is there a formal UAT phase where the customer validates the solution in their environment, and if so, what resources will they provide for that phase?

Answer: Assume yes, after production ready package is available to customer, we assume 2 week for testing. However we don't expect that we will know acceptance criteria before hand.

**Q25. Are there specific test coverage or quality requirements?**
Unit test coverage thresholds, integration test suites, or specific test scenarios (e.g., mass password expiration burst, DC failover during sync) should be agreed upon to avoid scope creep during delivery.

Answer: There are no explicit criteria for test coverage, however basic happy scenario end2end tests are expected to be checked in automated way. We will implement also reasonable unit test. Manual testing will be also needed. We shall test manually test scenario where 1000 passwords expires at the same time.

**Q26. Is observability integration (SIEM, Windows Event Log) in scope beyond structured file-based logs?**
The spec requires structured logs but does not mention Windows Event Log entries or SIEM forwarding. Many enterprise DC environments rely on Event Viewer and SIEM ingestion for security monitoring. If expected, this is additional scope.

Answer: File based logs are explicitly required but in implementation we shall use some C## alternative to SLF4j library that allow to configure different loggign targets.

-----

Another assumptions:
* No i18n needed - interface will be in English