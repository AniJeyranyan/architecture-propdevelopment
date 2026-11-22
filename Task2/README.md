# Task 2. Security checklist for the business systems

## Deliverable

- **[security-checklist.md](security-checklist.md)** is the completed checklist: 73 questions in 8 sections, each with a status, a priority, the business problem it relates to, and a comment based on the case materials.

## 1. Systems and how they handle data

The systems are identified from the case description and the as-is C4 container diagram.

| Domain | Systems | Security-relevant behaviour |
| ------ | ------- | --------------------------- |
| Sales (clients) | Sales storefront (monolith), `client-tour-app`, `client-mart-app` (online deal), `client-crm-app`, `client-mart-estate-app`, each with its own DB except the storefront | Collect full name, phone, email and extended forms for the deal. A client can be **registered in several places** (Keycloak, `client-crm-app`). The online deal connects to government systems through **SMEV** (SOAP). The property catalogue is published through Kafka. |
| Housing and utilities (owners) | Owner storefront and mobile app, `tenant-core-app` with its DB, the owner CRM with its DB | Utility payments (payment system), **smart home and the door lock**, and integration with **management company APIs** and four utility providers. |
| Finance | `accountant-service-1`, its DB, its own AD | Financial accounting. Employees are split across a separate AD domain. |
| Data | Greenplum DWH, S3, BI and reporting | **Raw CDC** from five production DBs, with no filtering or access restrictions. The BI analyst works with it directly. |
| Shared | `auth-service-1` (Keycloak) and `auth-db`, AD, the firewall | Keycloak authenticates clients and owners, and AD authenticates employees. The firewall covers incoming traffic from external users only. |
| Infrastructure | Own data centre, private cloud, rented cloud capacity | Teams can use rented cloud resources for development and testing. All DBs are backed up. |

The main security traits of the landscape:

1. Personal data is spread across 8+ stores and copied in raw form into the DWH and S3.
2. Identity is fragmented: there are several places where a client gets registered, plus two AD domains.
3. Partner integrations have no common security policy and no tenant isolation.
4. There is one security specialist, and changes reach production without their review.

## 2. Why these sections were chosen

The audit has four business goals, referred to as **P1–P4** in the checklist:

- **P1**: clients see another client's name in their personal account.
- **P2**: BI/ML/AI is blocked by data management problems.
- **P3**: preventing a leak like the competitor's.
- **P4**: compliance with Russian law (152-FZ).

Each section was kept or added only if it helps answer at least one of these goals. The course template (sections I–V) is used as the base because it gives a full baseline, and every external auditor will ask those questions. Three sections are added because the template doesn't cover the company's most specific problems.

### Template sections

| Section | Decision | Why |
| ------- | -------- | --- |
| **I. Access management** | Kept, top priority | **P1** is mainly an identity problem: there are several uncoordinated registration points and no single account registry (I.11, I.12), so clients get mixed up. **P3**: partner over-access, no least privilege on the DWH and the door lock in the owner app all make access the main leak path. |
| **II. Data security** | Kept, top priority | **P2** is a data governance problem: no classification, no inventory, no anonymization, raw CDC into Greenplum (II.1, II.7–II.9, II.13). **P4**: 152-FZ requires knowing where personal data is, protecting it and destroying it on time (II.7, II.10). **P1**: wrong names are an integrity failure (II.11). |
| **III. Infrastructure protection** | Kept, medium priority | **P3**: the landscape spans three hosting environments, faces the internet and has a firewall for incoming traffic only. These questions also cover the perimeter around partner traffic (III.12). Most answers are *Unknown*, which is a finding in itself: nobody has described the network to the security specialist. |
| **IV. Incident management** | Kept, high priority | **P3**: management's concern is a competitor's leak, so the audit must check whether a leak would be **detected** (IV.1, IV.2) and **handled** (IV.3). **P4**: 152-FZ sets 24h / 72h notification deadlines. The company learned about P1 from complaints, not from monitoring. |
| **V.1. Specific aspects: general** | Kept, low priority, except V.1.4 | These questions have no direct link to the case's problems and no supporting data. They are kept so the baseline is complete and comparable with later audits. V.1.4 (cloud backups) is medium because S3 holds a full copy of the DWH. |
| **V.2. Equipment and physical security** | Kept, low priority | The company has its own data centre, so physical access is in scope, but the case gives no data. These questions are only on-site checks for the auditors. |

None of the template sections were removed. Removing them would hide questions an auditor has to ask anyway. Instead, the **Priority** column shows which questions matter for PropDevelopment's goals. Low-priority items (11 of 73) are general hygiene to confirm, not things to fix first.

### Added sections

| Section | Why it was added |
| ------- | ---------------- |
| **VI. API and integration security** | The partner API problem is one of the company's two main problems, and the company plans to grow through partner services. The template only touches it indirectly (I.7, I.10, III.12). This section asks the questions that really matter here: data minimization in the contracts, tenant isolation (the "one management company sees another's data" issue), separate partner credentials, an API gateway, an inventory of internal data flows, and a security review for each new integration. |
| **VII. Change management and secure development** | It covers the **root cause** behind most gaps. After the IT leadership changes, the single security specialist stopped hearing about decisions and changes. Without a mandatory security review in the change process, any fix from this audit will slowly erode. This section also covers dev/test environments in the rented cloud and security checks in the DevOps CI/CD. |
| **VIII. Personal data compliance (152-FZ)** | **P4** is an explicit audit goal, but the template has no legal questions. The company processes personal data of clients and owners, passes it to partners and the cloud, and plans to use it for ML/AI. So the auditors need to check the Roskomnadzor notification, the legal basis for each purpose, data-processing agreements with third parties, data localization, protection levels (Government Decree No. 1119, FSTEC Order No. 21) and the handling of data subject requests. |

## 3. How the checklist was filled in

- **Sources:** only the case description and the C4 container diagram. Nothing was assumed beyond them.
- **Status:** **Yes** / **Partial** / **No** only where the case materials support the answer. When the case says nothing, the status is **Unknown**. Per the assignment, an answer is filled in only where there is enough data. Unknown items still have a comment that says what to check in the audit interviews and why it matters for PropDevelopment.
- **Priority:** **High** means directly tied to P1–P4, **Medium** means a baseline control that supports them, and **Low** means general hygiene.

## 4. Overall picture

| Section | Questions | Yes | Partial | No | Unknown |
| ------- | --------- | --- | ------- | -- | ------- |
| I. Access management | 14 | 1 | 1 | 7 | 5 |
| II. Data security | 14 | 0 | 2 | 6 | 6 |
| III. Infrastructure protection | 12 | 0 | 2 | 0 | 10 |
| IV. Incident management | 6 | 0 | 0 | 3 | 3 |
| V. Specific aspects | 7 | 0 | 0 | 0 | 7 |
| VI. API and integration security | 8 | 0 | 0 | 6 | 2 |
| VII. Change management and secure development | 6 | 0 | 0 | 3 | 3 |
| VIII. Personal data compliance (152-FZ) | 6 | 0 | 0 | 0 | 6 |
| **Total** | **73** | **1** | **5** | **25** | **42** |

There are **25 confirmed gaps**. They cluster in access management, data security and partner integrations, which are exactly the areas behind P1–P3. Only basic controls are confirmed: authentication through Keycloak and AD, a firewall for incoming traffic, and database backups. **42 items are Unknown**, and that is a finding too: the single security specialist doesn't have the information to answer them, which matches the broken communication described in the case.

## 5. Key gaps mapped to the business problems

| Business problem / audit goal | Root-cause gaps from the checklist |
| ----------------------------- | ---------------------------------- |
| **P1. Clients see another client's data (full name) in their personal account** | No single account or client registry, with several uncoordinated registration points (I.11, I.12). No data-integrity control (II.11). Internal data flows are undocumented (VI.6). Access logging to investigate the cases is unconfirmed (I.6). |
| **P2. BI / ML / AI initiatives are blocked** | No Data Governance (II.9) and no classification or labels (II.1, II.8). Raw CDC with personal data goes into Greenplum and S3 without anonymization (II.13), with no isolation (I.9) and unrestricted analyst access (I.4). No legal basis is confirmed for analytics as a processing purpose (VIII.2). |
| **P3. Preventing a leak like the competitor's** | No least privilege (I.4) and no tenant isolation or common policy in the partner APIs (VI.1–VI.3, I.7). No API gateway (VI.5). Alerts come from customer complaints, not monitoring (IV.2). No security review in the change process (VII.1, VII.2), and too few security staff (VII.3). |
| **P4. Compliance with Russian law (152-FZ)** | No personal-data inventory (II.7). Personal data is passed to partners beyond what's needed (VI.2). No audit cycle (IV.6). Retention and destruction, the incident plan for the 24h / 72h deadlines, localization and protection levels are unconfirmed (II.10, IV.3, VIII.1–VIII.6). |

## 6. Priority recommendations

1. **Single client identity (P1).** Make Keycloak the single registration and identity source, give each client a master ID, and have every other system reference that ID instead of registering clients itself.
2. **Partner API security (P3, P4).** Route partner traffic through an API gateway with OAuth2 client credentials and ABAC on the management company and residential complex. Strip personal data that a contract doesn't need, and log every partner call.
3. **Data protection and governance (P2, P4).** Approve the classification from Task 1 and keep a register of personal data. Pseudonymize or anonymize data on the way into the DWH, restrict analyst access by role and data class, and encrypt data at rest and in transit, including backups.
4. **Security in the change process (all goals).** Make a security review mandatory for every architecture change and new integration, appoint security champions in the product teams, add security checks to CI/CD, and plan for growing the security team.
5. **Access management (P3).** MFA for employees and for critical actions in the owner app, such as the door lock. Least privilege, quarterly reviews of privileged accounts, merging the two AD domains, and automatic deprovisioning tied to HR.
6. **Detection, response and compliance (P3, P4).** Centralized logging and a SIEM, alerting, and an incident response plan that includes the 152-FZ notification procedure. Confirm the Roskomnadzor notification, the legal bases, the data-processing agreements and data localization.
