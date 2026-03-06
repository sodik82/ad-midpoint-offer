**Active Directory to midPoint Password Synchronization** 

**1\. Executive Summary** 

We are seeking the development of a fault-tolerant mechanism to intercept native password changes on Active Directory (AD) Domain Controllers and propagate them to our Identity Governance system, midPoint. The goal is to ensure that when a user changes their password in Windows, it is 

immediately and securely synchronized to midPoint to maintain consistent authentication across the 

enterprise. 

**2\. High-Level Architecture** 

The solution will be deployed as an agent on all AD Domain Controllers. To ensure stability and performance of the critical Windows Local Security Authority (LSA) subsystem, the architecture must consist of two decoupled components: 

1\. **The Listener (DLL):** A lightweight PasswordChangeNotify filter loaded by the LSA. Its only 

responsibility is to capture the password change and hand it off immediately to a local queue. It must not perform network I/O or heavy processing. 

2\. **The Sending Service:** A background Windows service that picks up events from the queue and 

handles the transmission to the midPoint REST API. This service handles the logic for retries, encryption, and connectivity management. 

As for the programming languages, the listener must be developed in C/C++. For the sending service, C\# is preferred. 

**3\. Key Functional Requirements** 

• **Interception:** The solution must intercept successful password changes on every Domain 

Controller. 

• **Near-Synchronous Propagation**: While the user experience should feel synchronous, the technical implementation must be asynchronous (near-synchronous) to prevent blocking the LSA subsystem. 

• **Target API:** Interaction will be done via midPoint's REST API (specifically notifyChange) using 

TLS 1.3. Link to details is below. 

•   
**Offline Capability:** If midpoint is unreachable, password changes must be encrypted and queued locally on the DC. The service must attempt to replay these changes until successful or 

until a configurable timeout. 

**4\. Security & Compliance** 

• **Transport Security:** All traffic to midPoint must be encrypted via HTTPS/TLS 1.3. 

**Data at Rest:** Queued passwords waiting to be sent must be encrypted on the local disk. The 

encryption keys should be managed securely (e.g., via Windows Credential Manager or DPAPI) 

• 

•   
rather than stored in plain text configuration files. 

**Authentication:** The service will authenticate to midPoint using HTTP Basic Authentication or 

OIDC. 

**Least Privilege:** The service should run with the minimum necessary permissions on the DC. 

**5\. Non-Functional Requirements** 

• 

•   
**Reliability:** No material delay in AD password setting operations. The failure of the 

synchronization service must not prevent the user from changing their password in AD. 

**Performance:** The system must handle burst loads (e.g., mass password expirations) via threading and queuing without dropping events, up to a configurable queue depth. 

• **Logging:** Structured logs must be generated for operations (intercept, queue, send, success/fail) to support observability, while strictly excluding actual password values from logs. 

**6\. Implementation Guidelines** 

•   
**Rate Limiting:** Instead of a fixed request-per-second limit, the sending service should manage load via a configurable thread pool size. 

**Policy Alignment**: Password complexity policies will be managed by administrators in AD and 

midpoint separately; the software does not need to enforce policy logic, only report delivery success/failure. 

**7\. Development & Testing Environment** 

• **Infrastructure Responsibility:** The vendor is expected to provision and maintain their own 

local development and testing environment (Active Directory Domain Controller \+ midPoint instance). Access to the client's internal environment will not be provided during the development phase. 

•   
**Configuration Support:** To ensure the local environment matches our production requirements, we will provide the specific midpoint resource definitions (XML) and schema configurations necessary to replicate the target API behavior. 

**8\. Required Deliverables** 

•   
**Automated Installer (MSI):** The solution must be delivered as a professional Windows Installer (MSI) package. 

• It must handle the registration of the LSA Password Filter DLL in the Windows Registry. 

It must install and register the Background Service. 

• It must support silent installation arguments for automated deployment via Group Policy or SCCM. 

•   
**Source Code & Licensing:** 

•   
• The full source code must be delivered via a Git repository. 

• **Licensing:** The software will be published as Open Source under the **EUPL 1.2** 

**(European Union Public License)**. The code headers and repository structure must 

reflect this license. 

**Documentation:** 

• **Administrator Guide:** Instructions for installation, configuration (e.g., changing timeout values, rotating encryption keys), and troubleshooting logs. 

**Build Documentation:** Instructions on how to build the solution from source, including all dependency requirements. 

• **Support Proposal:** A separate quotation for 12 months of post-deployment maintenance, covering security patches, Windows Server update compatibility, and bug fixes. 

**9\. Supported Platforms & Compatibility** 

The solution must be validated and fully functional on all Windows Server versions currently under Microsoft Mainstream or Extended Support. 

• 

· 

•   
**Target OS Versions:** The LSA Filter and Synchronization Service must operate seamlessly on **Windows Server 2016, 2019**, **2022, and 2025.** 

**Backward Compatibility:** The installer (MSI) must correctly detect the underlying OS version and apply the appropriate registration logic for the LSA notification package without disrupting existing legacy configurations. 

*Note:* We *are aware that* Windows *Server* 2016 *does not natively support* TLS 1.3. *For Server* 2016/2019*, the application* should *default* to *the* highest *supported protocol* (TLS 1.2*)* or strictly *enforce* TLS *1.3 only where the* OS allows *it (Server* 2022+). 

**10\. Additional Resources & References** 

• **Inalogy AD Password Agent (GitHub)**: 

•   
https://github.com/inalogy/ad-password-agent 

*Note:* This implementation is very complex and does not fulfill all requirements. 

**REST API \- notifyChange Endpoint**: 

https://docs.evolveum.com/midpoint/reference/support-4.10/interfaces/rest/ 

operations/notify\-op\-rest/ 

*Note:* the data structure details may slightly change. 