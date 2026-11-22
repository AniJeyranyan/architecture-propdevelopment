# PropDevelopment — Information Security Audit Checklist

Filled in from the case description and the as-is C4 container diagram. Sections I–V follow the course template. Sections VI–VIII are added for PropDevelopment's specific problems; [README.md](README.md) explains why.

**Status values**

- **Yes**: the control is in place, and the case materials confirm it.
- **Partial**: some of it exists, but it isn't complete, systematic or documented.
- **No**: the case materials show the control is missing.
- **Unknown**: the case materials say nothing about it, so there isn't enough data to answer. Each of these is a question to confirm in the audit interviews. The comment says what to check and why it matters here.

**Priority** (order of verification and remediation)

- **High**: directly tied to one of the business problems below.
- **Medium**: a baseline control that supports a business problem.
- **Low**: general hygiene with no direct link to the case's problems.

**Business problems** (the `Problem` column)

- **P1**: clients see another client's data (full name) in their personal account.
- **P2**: data management problems are blocking the new BI/ML/AI products.
- **P3**: preventing a data leak like the competitor's.
- **P4**: compliance with Russian law (152-FZ on personal data).

| №   | Question | Status | Priority | Problem | Comment |
| --- | -------- | ------ | -------- | ------- | ------- |
| **I. Access management** | | | | | |
| 1   | Is multi-factor authentication (MFA) used to access critical systems and services? | Unknown | High | P3 | Clients and owners sign in through `auth-service-1` (Keycloak), and employees use Active Directory. Nothing in the case says MFA is enforced. Check the owner app first: it controls smart-home devices and the **apartment door lock**, so a stolen password could open a real door. Then check employee access to the CRMs, `accountant-service-1` and Greenplum. |
| 2   | Are password complexity and regular rotation requirements configured? | Unknown | Medium | P3 | Keycloak and AD can enforce password policies, but the case doesn't describe any company password policy. Ask for the Keycloak realm settings and the AD Group Policy. |
| 3   | Are privileged accounts audited and reviewed regularly? | No | High | P3 | There is one security specialist for the whole company, and since the IT leadership changes they no longer hear about every decision and change. Nobody can review privileged accounts they don't know about. Admin access to the DBs, Kafka, Greenplum and the cloud should be reviewed first. |
| 4   | Is the principle of least privilege applied when roles and access rights are assigned? | No | High | P2, P3 | The case says the data processing systems have no restrictions on data access. The BI analyst works directly in Greenplum, which receives **raw CDC** copies of five production databases, including personal data. In the partner APIs, one management company can see and change another company's clients and residential complexes. |
| 5   | Are accounts disabled or removed automatically when an employee leaves or changes role? | Unknown | Medium | P3 | Employees are split across two AD domains (sales and finance), and no HR-system integration is described. The recent turnover among IT team leads makes stale accounts likely, so check this early in the interviews. |
| 6   | Are all operations that access systems and data logged? Who has access to the logs? | Unknown | High | P1, P3 | The case describes no access logging for the apps, DBs, Kafka, the DWH or the partner APIs. Without access logs, nobody can investigate the "another client's name in my account" complaints or a possible leak, so this is the first thing to confirm. |
| 7   | Is attribute-based access control (ABAC) used to make access management more flexible? | No | High | P3 | The partner API contracts don't check tenant attributes, which is why one management company can reach another company's data. ABAC on `management_company_id`, `complex_id` and owner is exactly the missing control. |
| 8   | Are user authentication systems used to access the systems? | Yes | Medium | P1 | `auth-service-1` (Keycloak) handles client and owner registration and sign-in. AD authenticates the managers in `client-crm-app`, finance users in `accountant-service-1` and the BI analyst. Remaining gaps: the two AD domains are separate, and partner systems don't use any common identity provider. |
| 9   | Are isolated environments set up for subsystems that handle sensitive data? | No | High | P2, P3 | CDC copies five DBs (`estate`, `crm`, `mart`, `tenant-core`, `crm-tenant`) unfiltered into one Greenplum instance and on to S3. Personal data, deal documents and finance-related data end up in one shared store with no isolation. |
| 10  | Is access to the systems from external sources restricted? | Partial | High | P3 | External users (clients and owners) come in through a firewall. However, partner APIs are integrated without common security policies, and partners can do things they shouldn't be able to. See section VI. |
| 11  | Is there a single registry of accounts? | No | High | P1 | A new client can be registered at several points with no coordination between them: Keycloak, `client-crm-app` ("client registration, card, profile") and the owner CRM. Employees are split across two ADs. Duplicate or mismatched client records are the most likely cause of clients seeing **another client's full name** in their personal account. |
| 12  | Are accounts managed with tools built on that single registry? | No | High | P1 | There's no single registry (I.11), so each system manages its own accounts and profiles, and they drift apart. |
| 13  | Are there tools for revoking access rights? | Unknown | Medium | P3 | Keycloak and AD have the technical means to disable users, but the case doesn't describe any process for it. Check how partner API credentials and direct DB/DWH accounts are revoked. |
| 14  | Are centralized access-control tools used? | No | High | P3 | The case says the limits on what data is passed are set mostly by the business processes between the products. The C4 diagram shows no API gateway or central policy point for the internal REST and Kafka flows or for the partner APIs. |
| **II. Data security** | | | | | |
| 1   | Is there a data classification policy, and is it kept up to date with current legislation? | No | High | P2, P4 | There isn't one. Task 1 of this project proposes the first version (public / internal / confidential / secret). The company itself isn't sure it complies with Russian law, in particular **152-FZ** on personal data. |
| 2   | Is encryption used at database and file-system level and for data in transit? | Unknown | High | P3 | The case doesn't describe encryption at rest for MSSQL, PostgreSQL, Greenplum or S3, or TLS for the internal REST and Kafka traffic and the CDC replication. Check the CDC flow and S3 first, because they hold full copies of the personal data. |
| 3   | Are encryption keys managed? | Unknown | Medium | P3 | The case mentions no KMS, HSM or key rotation. Also check how integration secrets (partner API keys, payment system credentials) are stored. |
| 4   | Is data backed up regularly? Are backups kept in protected storage? | Partial | Medium | P3 | All databases are backed up "to the depth required for recovery". Where the backups are stored and how that storage is protected isn't described. |
| 5   | Is restoring data from backups tested regularly? | Unknown | Medium | P3 | The case doesn't describe restore testing, RPO/RTO targets or recovery drills. |
| 6   | Is access to backups restricted, and are the backups encrypted? | Unknown | Medium | P3, P4 | Backups of `crm-db`, `crm-tenant-db` and the deal DBs hold full personal data and document details, so an unprotected backup is a leak channel. The case says nothing about encrypting them or restricting access. |
| 7   | Is personal data inventoried, and are specific handling and storage procedures applied to different data types? | No | High | P2, P4 | Personal data is spread across Keycloak, `client-crm-db`, `client-mart-db`, `client-tour-db`, `tenant-core-db`, `crm-tenant-db`, Greenplum, S3 and the partner APIs, with no inventory. The partner API contracts openly expose personal data. |
| 8   | Is data access monitored according to its classification? Are security labels used to mark data? | No | High | P2 | There's no classification (II.1), so nothing is labelled. Greenplum receives raw data with no labels or column-level policies. |
| 9   | Are Data Governance processes and systems in place? | No | High | P2 | There is a Data domain (DWH, BI, reporting), but the case says the problems with managing and storing data flows are blocking the new data products. No data owners, catalogue, lineage or quality rules are described. |
| 10  | Is there a system or process for secure data destruction? | Unknown | Medium | P4 | The case describes no retention periods or deletion process. 152-FZ requires personal data to be destroyed once the purpose of processing is met (for example, a client who never bought a flat). Copies in the DWH, S3 and backups make this harder. |
| 11  | Are data-integrity controls used? | No | High | P1 | The wrong full name in personal accounts is itself an integrity failure. There's no master client ID and no reconciliation between the CRM, Keycloak and the sales apps, and the raw CDC copies carry those errors downstream. |
| 12  | Are there systems or processes to make sure data is preserved (not lost)? | Partial | Medium | P3 | Backups exist (II.4). The case doesn't say whether anyone checks that backups complete or tests restores (II.5). |
| 13  | Are anonymization tools or processes used? | No | High | P2, P4 | The C4 diagram labels the CDC flows "raw data transfer". Full personal data goes straight to Greenplum and S3, and the BI analyst works on it directly. Pseudonymization or anonymization is needed both for ML/AI and for 152-FZ. |
| 14  | Is there a written procedure for restoring data? | Unknown | Medium | P3 | The case doesn't describe a restore runbook. |
| **III. Infrastructure protection** | | | | | |
| 1   | Is a firewall used to restrict incoming and outgoing traffic? | Partial | Medium | P3 | A firewall controls **incoming** traffic from external users. The case doesn't describe any filtering of **outgoing** traffic to the payment system, utility providers, SMEV or the rented cloud. |
| 2   | Is the network segmented into logical zones (DMZ, corporate network, guest network) to limit access to critical resources? | Unknown | Medium | P3 | The landscape spans the company's own data centre, a private cloud and rented cloud capacity, and the case describes no DMZ or zones. Check whether the public storefront sits in the same network as the CRMs, finance and the DWH. |
| 3   | Are IDS/IPS systems used to detect and prevent network attacks? | Unknown | Low | P3 | Not described. |
| 4   | Is network traffic monitored for anomalies and possible attacks such as DDoS? | Unknown | Medium | P3 | The public storefront and the owner app face the internet, and no DDoS protection is described. |
| 5   | Are operating systems, applications and network devices scanned for vulnerabilities regularly? | Unknown | Medium | P3 | Not described. With one security specialist who isn't told about every change, regular scanning is unlikely. See VII.4 for scanning in CI/CD. |
| 6   | Are antivirus and EDR installed on servers and workstations? | Unknown | Low | P3 | Not described. |
| 7   | Are unused services and ports disabled on servers, workstations and network equipment? | Unknown | Low | P3 | Not described. Temporary cloud environments for development and testing are the likeliest place to find leftover open services. |
| 8   | Are system-file integrity controls used? Is access to system resources logged? | Unknown | Low | P3 | Not described. |
| 9   | Are cloud environments scanned automatically against security baselines? | Unknown | Medium | P3 | Teams can use rented cloud resources and managed services for development and testing. The case doesn't describe any baseline scanning for them. |
| 10  | Are anti-malware signatures updated automatically? | Unknown | Low | P3 | Not described. |
| 11  | Is traffic between network segments filtered? | Unknown | Medium | P3 | Depends on III.2. Check especially the traffic between the data centre, the private cloud and the rented cloud. |
| 12  | Are tools used to control the access perimeter? | Partial | High | P3 | The external firewall is the only perimeter control the case describes. Partner API traffic doesn't go through an API gateway or WAF, according to the C4 diagram. See VI.5. |
| **IV. Incident management** | | | | | |
| 1   | Is a SIEM used to correlate events and analyse anomalies in real time? | Unknown | High | P3 | Not described. Depends on access logging being in place (I.6). |
| 2   | Are automatic alerts set up for incidents and suspicious activity (for example, unauthorized access attempts)? | No | High | P1, P3 | The company found out about the wrong-name problem from **client complaints**, not from monitoring. |
| 3   | Is there an incident response plan covering analysis, escalation and system recovery? Are employees trained on it? | Unknown | High | P3, P4 | Not described. 152-FZ requires telling Roskomnadzor about a personal-data leak **within 24 hours** and sending the investigation results **within 72 hours**. Without a plan the company can't meet those deadlines in a scenario like the competitor's. |
| 4   | Are incident drills run to check that staff are ready to respond to cyberattacks? | Unknown | Medium | P3 | Not described. |
| 5   | Are statistics kept for every registered incident, and are the findings used to adjust security policy? | No | Medium | P1, P3 | The development teams noticed the cross-company partner access "more than once" during integrations, and clients keep complaining about wrong names. Neither problem has been fixed or traced to its source, which means they aren't being tracked as incidents. |
| 6   | How often are internal and external security audits held? Are policies and procedures updated based on their results? | No | Medium | P4 | The company is only now preparing its first comprehensive security audit, so there's no regular audit cycle. Recommendation: an annual external audit and quarterly internal reviews. |
| **V. Specific aspects** | | | | | |
| **V.1. General** | | | | | |
| 1   | Is a mobile device management (MDM) system used to control access to corporate data and applications? | Unknown | Low | P3 | Not described. This applies to employee devices, not the owner app. |
| 2   | Are VPN and encryption set up for secure remote access to corporate resources? | Unknown | Low | P3 | Not described. Engineers work across the company's own data centre and two clouds, so ask how they connect. |
| 3   | Are unregistered or unauthorized devices on the network detected and restricted automatically? | Unknown | Low | P3 | Not described. |
| 4   | Are automatic backup policies set up in the cloud environment? Is access to those backups controlled? | Unknown | Medium | P3 | The case describes backups only for databases. S3 (the DWH export) and cloud workloads aren't mentioned. |
| **V.2. Equipment management and physical security** | | | | | |
| 1   | Are access control systems (biometrics, badge-based physical access control) used in server rooms? | Unknown | Low | P3 | The company runs its own data centre, but physical access control isn't described. Check on site. |
| 2   | Is all equipment inventoried? Is its condition tracked with asset management systems? | Unknown | Low | P3 | Not described. The hardware spans the company's own data centre, a private cloud and rented cloud. |
| 3   | Is data removed securely before equipment is disposed of or handed to other departments? | Unknown | Low | P3, P4 | Not described. |
| **VI. API and integration security** *(added for this case)* | | | | | |
| 1   | Is there a single security policy for partner API integrations? | No | High | P3 | The case says outright that PropDevelopment uses the management companies' APIs "without unified security policies". The company plans to grow through partner services, so this risk will increase. |
| 2   | Are API contracts reviewed so that only the data needed for the purpose is passed? | No | High | P3, P4 | The case says the API contracts contain data categories that openly expose personal data, and there is no systematic control over the data passed. |
| 3   | Do partner APIs enforce tenant isolation, so a management company sees only its own owners and residential complexes? | No | High | P3 | A partner from one management company can view or change another company's client and residential-complex data. This is a broken access control at object level (OWASP API1: BOLA). |
| 4   | Does each partner system use its own credentials (for example, OAuth2 client credentials or mTLS) with permissions limited to what it needs? | Unknown | High | P3 | The case doesn't say how partners authenticate. Since one partner can reach another's data, the credentials are at least not scoped by tenant. |
| 5   | Does all partner traffic go through a central point (an API gateway) that validates requests, limits request rates and logs calls? | No | High | P3 | The C4 diagram shows no API gateway, and the firewall is shown only for clients and owners. |
| 6   | Are the internal data flows between products (REST, Kafka, CDC) documented, with an owner and an approved data set for each? | No | High | P1, P2 | The case says internal data flows between the products are controlled unsystematically. Five CDC flows copy raw data into the DWH. |
| 7   | Is every new integration, with partners or government systems, reviewed for security before it goes live? | No | High | P3, P4 | Integrations with service providers and new resident services have been added without a security review, and the security specialist doesn't hear about every change. The company isn't sure these processes are legal. |
| 8   | Do the outgoing integrations (payment system, utility providers, SMEV) use encrypted channels, with their secrets kept in a secrets store? | Unknown | Medium | P3, P4 | Not described. SMEV has its own message-signing requirements, so check that the other integrations are protected too. |
| **VII. Change management and secure development** *(added for this case)* | | | | | |
| 1   | Does every architecture change, new system or new integration go through a mandatory security review? | No | High | P3 | The case says the security specialist hasn't been told about every decision and change. This is the root cause behind most of the gaps in VI. |
| 2   | Is there a documented process for passing security-relevant information between teams and their leads? | No | High | P3 | The case says information flow between teams broke down after the IT team leads changed. |
| 3   | Does the security team have enough people for the size of the landscape, for example with security champions in the product teams? | No | High | P3 | There is one security specialist for four domains, more than ten applications, three hosting environments and several external integrations. |
| 4   | Do the DevOps CI/CD pipelines include security checks (SAST, DAST, dependency and container scanning)? | Unknown | Medium | P3 | Each product team has a DevOps engineer, but the case doesn't mention security checks in the pipelines. |
| 5   | Are temporary cloud environments for development and testing created through an approved process, without real production personal data? | Unknown | High | P3, P4 | Teams can use rented cloud resources and managed services for development and testing. Check whether those environments get copies of production personal data. |
| 6   | Are security requirements part of the product backlog and the definition of done? | Unknown | Medium | P3 | Not described. |
| **VIII. Personal data compliance (152-FZ)** *(added for this case)* | | | | | |
| 1   | Has the company notified Roskomnadzor that it processes personal data, and is that notification up to date for all current purposes? | Unknown | High | P4 | Not described. The IT landscape has grown with new integrations and services, so the registered purposes may be out of date. |
| 2   | Is there a legal basis (consent, contract or law) for each processing purpose: booking, deal, utility services, smart home, analytics? | Unknown | High | P2, P4 | Not described. Using client data for the planned BI/ML/AI services is a new purpose that the existing consents probably don't cover. |
| 3   | Do contracts cover every transfer of personal data to third parties: management companies, utility providers, the payment system and the cloud provider? | Unknown | High | P4 | Personal data goes to partners through the APIs (VI.2) and is processed in rented cloud capacity. Check the data-processing agreements (Article 6(3) of 152-FZ). |
| 4   | Is the personal data of Russian citizens first recorded and stored in databases located in Russia, including in the rented cloud? | Unknown | High | P4 | Data localization is required by Article 18(5) of 152-FZ. The cloud provider and its data centre locations aren't named. |
| 5   | Have protection levels been set for the personal data systems, and are the matching protection measures in place (Government Decree No. 1119, FSTEC Order No. 21)? | Unknown | High | P4 | Not described. The CRMs, the deal systems, Greenplum and S3 all process personal data of clients and owners. |
| 6   | Is there a procedure for data subject requests (access, correction, deletion, withdrawal of consent) that meets the 10-working-day deadline? | Unknown | Medium | P1, P4 | Not described. With no master client record (I.11), the company can't reliably find or correct all of a person's data. |

## Result

| Section | Questions | Yes | Partial | No | Unknown |
| ------- | --------- | --- | ------- | -- | ------- |
| I. Access management | 14 | 1 | 1 | 7 | 5 |
| II. Data security | 14 | 0 | 2 | 6 | 6 |
| III. Infrastructure protection | 12 | 0 | 2 | 0 | 10 |
| IV. Incident management | 6 | 0 | 0 | 3 | 3 |
| V.1. General | 4 | 0 | 0 | 0 | 4 |
| V.2. Equipment and physical security | 3 | 0 | 0 | 0 | 3 |
| VI. API and integration security | 8 | 0 | 0 | 6 | 2 |
| VII. Change management and secure development | 6 | 0 | 0 | 3 | 3 |
| VIII. Personal data compliance (152-FZ) | 6 | 0 | 0 | 0 | 6 |
| **Total** | **73** | **1** | **5** | **25** | **42** |

By priority: **High**: 37, **Medium**: 25, **Low**: 11.
