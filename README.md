# Azure Function App Deployment Review (Friday Deploy)

## Overview

This lab involved reviewing a newly deployed Azure Function App before it could be promoted to production. The deployment contained several security and operational weaknesses that needed to be identified before release.

I investigated the application's configuration in the Azure portal, compared it against an existing production deployment, and verified whether a storage blob could be accessed without authentication. I then prioritized the findings based on their impact and urgency.

## Environment

- **Cloud platform:** Microsoft Azure
- **Compute service:** Azure Functions
- **Storage service:** Azure Storage
- **Identity platform:** Microsoft Entra ID
- **Tools:** Azure portal, Function App configuration, Azure Storage browser, private browsing window
- **Access level:** Reader access in a shared Azure training environment
- **Resource group:** `rg-madhat-lab-chap4`
- **Application under review:** `func-intern-notify`
- **Production comparison:** `func-madhat-shared`

## Investigation

### Checkpoint 1: Regional Deployment Review

I opened the Function App's Overview blade and inspected its deployment location. The application was deployed in a different region from the production environment it was intended to work alongside.

I compared the configuration against the production baseline to identify the regional mismatch.

**Finding:** The Function App was deployed in an inconsistent region.

**Potential impact:**
- Increased latency when communicating with resources in another region.
- Potential cross-region data transfer charges.
- Unnecessary deviation from the established deployment architecture.

**Recommendation:** Align the Function App's region with the approved architecture after confirming service availability, data residency requirements, and resource dependencies.

**Evidence: Function App regional configuration**

<img width="852" height="357" alt="image" src="https://github.com/user-attachments/assets/ff24b206-c5f9-4177-9c6d-8f036ee10bfe" />


### Checkpoint 2: Managed Identity Review

I inspected the intern's Function App under **Identity** and found that neither a system-assigned nor a user-assigned managed identity was configured.

I then examined the production Function App's identity configuration to understand how the established deployment handled workload authentication.

**Finding:** The intern's Function App had no managed identity configured.

**Potential impact:**
- The application may rely on stored credentials to access dependent services.
- Stored credentials can be exposed through application configuration, source code, or deployment artifacts.
- Credential rotation and access management can become more difficult.

**Recommendation:** Determine the application's authentication requirements and implement an appropriate managed identity. Assign only the required permissions at the narrowest practical scope, avoiding stored credentials wherever managed identity authentication is supported.

**Evidence: Identity and storage investigation**

<img width="940" height="692" alt="image" src="https://github.com/user-attachments/assets/f1d79517-604b-4e08-a180-1b866d58ef55" />

<img width="960" height="553" alt="image" src="https://github.com/user-attachments/assets/aaed097f-6398-4f56-9a64-ce0663eeb25a" />


### Checkpoint 3: Public Storage Exposure

I inspected the storage account associated with the deployment and reviewed the containers' anonymous access settings.

One container permitted public access. To verify the exposure, I copied the URL for `deploy-notes.txt` and opened it in a private browser window without signing in.

The file loaded without requiring authentication.

**Finding:** A storage blob containing deployment notes was accessible anonymously over the internet.

**Potential impact:**
- Unauthorized disclosure of internal deployment information.
- Exposure of operational details that could assist further investigation by an attacker.
- Additional risk if credentials or other sensitive information are present in publicly accessible files.

**Recommendation:** Immediately restrict unnecessary anonymous access, review the exposed content for credentials or other sensitive information, and investigate available access logs. Rotate any credentials found to have been exposed and assess whether further incident response is necessary.

The exposure should be treated as a security issue requiring investigation, not simply a configuration cleanup.

**Evidence: Anonymous access verified in a private browser window**

<img width="835" height="347" alt="image" src="https://github.com/user-attachments/assets/7ef7529a-fb23-478f-97d9-e94427c7cde3" />

<img width="952" height="346" alt="image" src="https://github.com/user-attachments/assets/df9aef19-1374-495f-b6db-c3ceccbd75be" />

### Checkpoint 4: Inbound Access Restrictions

I inspected the Function App's **Networking → Access restrictions** configuration and compared it with the production application's settings.

The intern's application allowed traffic from any source and had no meaningful restrictive access rules configured. The production baseline used deny rules to restrict inbound access.

**Finding:** The application had an unnecessarily permissive inbound access configuration.

**Potential impact:**
- Untrusted sources could reach the application endpoint.
- An unnecessarily exposed endpoint increases the application's attack surface.
- Public reachability could enable probing and abuse of the application.

**Recommendation:** Confirm whether inbound access is required. If it is not, disable public network access where appropriate. Otherwise, configure restrictive access rules and validate the unmatched-rule behavior against legitimate application traffic requirements.

**Evidence: Intern deployment access restrictions**

<img width="956" height="817" alt="image" src="https://github.com/user-attachments/assets/45a5d8df-e0d6-4b81-82e8-bd03e9a455f1" />


**Production baseline comparison**

<img width="951" height="807" alt="image" src="https://github.com/user-attachments/assets/20a972c4-3033-46d4-bf2d-ad364aadc8a7" />


### Checkpoint 5: Risk Prioritization and Remediation

After investigating the deployment, I evaluated the four findings according to their urgency and potential impact.

| Priority | Finding | Recommended action |
|---|---|---|
| 1 — Immediate | Publicly accessible storage blob | Restrict anonymous access immediately, assess the exposed content, and investigate potential access |
| 2 — High | Missing managed identity | Implement managed identity and least-privilege authorization based on the application's dependencies |
| 3 — High | Unrestricted inbound access | Restrict or disable unnecessary inbound access and validate required connectivity |
| 4 — Planned correction | Regional mismatch | Align the deployment with the approved regional architecture |

The public storage exposure takes immediate priority because I verified that the file could be retrieved without authentication. This is a confirmed exposure rather than a hypothetical weakness.

The remaining findings also require remediation. Their implementation order should account for the application's dependencies and exposure, while the regional mismatch should be corrected through the deployment process after confirming the intended architecture.

**Key decision:** Address confirmed public data exposure first, then remediate the identity and inbound access weaknesses, and correct the regional deployment deviation.

## Key Findings Summary

| Finding | Category | Main concern |
|---|---|---|
| Regional mismatch | Architecture and cost | Latency and potential cross-region transfer charges |
| Missing managed identity | Identity and access management | Potential reliance on stored credentials |
| Public storage blob | Data security | Confirmed unauthenticated access to deployment notes |
| Permissive inbound access | Network security | Unnecessary public attack surface |

## What I Learned

- **Baseline comparisons:** Comparing a deployment against a known-good production configuration helps identify deviations that may otherwise be overlooked.
- **Managed identities:** Azure workloads should use managed identity authentication where supported, with authorization limited to the permissions they actually need.
- **Exposure validation:** Testing the blob URL in a private browser provided direct evidence that the file was accessible without authentication.
- **Access restrictions:** Applications should expose only the endpoints required for their intended operation.
- **Risk prioritization:** An actively accessible data exposure requires a different response priority from a regional inefficiency or a configuration weakness that has not resulted in confirmed exposure.
- **Deployment reviews:** A deployment can function correctly while still introducing security and operational risks.

## Conclusion

This investigation demonstrated how a cloud deployment can work as intended while still introducing security and operational weaknesses.

By inspecting the Function App, comparing it against the production baseline, and validating anonymous storage access, I identified configuration deviations and established a remediation order based on risk.

The central lesson was to investigate actual configurations, verify exposure with evidence, and prioritize remediation according to impact and urgency rather than treating every finding as equally important.

## Security Note

This project was completed in a shared Azure training environment. Before publishing screenshots or evidence to a public repository, redact lab flags, credentials, tokens, tenant identifiers, account information, and other sensitive values. Do not publish exposed secrets or the contents of sensitive deployment notes.
