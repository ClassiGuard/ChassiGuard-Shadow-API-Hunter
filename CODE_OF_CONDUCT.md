# Code of Conduct, Acceptable Use Policy & Legal Disclaimer
## ClassiGuard Shadow API Hunter

---

## 1. Important Notice: 6-Month Proof of Concept (POC) & Evaluation Window

> [!IMPORTANT]
> **Proof of Concept (POC) Status & Commercial Notice**:
> This repository and software are published strictly as an open **Proof of Concept (POC) for a temporary evaluation period of six (6) months** (from October 2026 through April 2027). The sole purpose of this public POC release is to demonstrate architecture, solicit technical feedback, gather real-world community performance metrics, and validate enterprise requirements.
> 
> **This software is NOT free forever.**
> Following the conclusion of the 6-month evaluation and feedback period, ClassiGuard reserves all rights to transition future versions, features, updates, and maintenance into commercial enterprise licensing, proprietary distribution tiers, and paid subscription models. Continuous usage beyond evaluation mandates compliance with subsequent commercial licensing terms.

---

## 2. Absolute Operator Liability & Disclaimer of Warranty

1. **Sole Operator Responsibility**:
   Any person, organization, entity, or administrator who downloads, clones, compiles, deploys, configures, or executes the ClassiGuard Shadow API Hunter software assumes **100% sole and exclusive legal, technical, and operational responsibility** for all actions, outcomes, network traffic, system impacts, and data collected resulting from its execution.

2. **Total Disclaimer of Liability (ClassiGuard)**:
   - The software and accompanying specifications are provided **"AS IS" AND "AS AVAILABLE"**, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, TITLE, OR NON-INFRINGEMENT.
   - In no event shall **ClassiGuard**, its project creators, developers, contributors, or maintainers be held liable for any claim, damages, downtime, service interruptions, data corruption, regulatory fines, legal proceedings, civil liabilities, or criminal penalties arising from the installation, execution, configuration, or misuse of this software by any third party or operator.

3. **Indemnification**:
   By using this software, you agree to indemnify, defend, and hold harmless ClassiGuard and its contributors against any and all claims, damages, liabilities, costs, and expenses (including legal fees) arising out of or related to your use, deployment, or unauthorized operation of the tool.

---

## 3. Law Enforcement Cooperation & Zero Tolerance Policy

ClassiGuard is dedicated to defensive cybersecurity and the protection of digital infrastructure. We maintain a **zero-tolerance policy** against malicious, unauthorized, offensive, or destructive activities.

> [!CAUTION]
> **ClassiGuard Stands with Law Enforcement**:
> In the event that this software is misused to conduct unauthorized scanning, intrusive probing, denial of service, data exfiltration, or any other unauthorized or cyber-criminal activity, **ClassiGuard will proactively and fully assist and cooperate with Indian and International Law Enforcement Agencies**.
> 
> This includes immediate cooperation with:
> - **CERT-In (Indian Computer Emergency Response Team)** under Section 70B of the Information Technology Act, 2000.
> - **Indian Cyber Crime Coordination Centre (I4C)** and National Cyber Crime Reporting Portal.
> - **State and Central Cyber Crime Police Wings / Special Cyber Investigation Cells**.
> - **Judicial and Regulatory Authorities**.

ClassiGuard maintainers will provide technical analysis, architectural forensics, and testimony required by statutory authorities to hold malicious actors accountable under the law.

---

## 4. Statutory Legal Framework (Republic of India)

All users and contributors must strictly comply with the laws of the Republic of India and their local territorial jurisdictions:

1. **The Information Technology Act, 2000 (as amended)**:
   - **Section 43 & 43A**: Prohibits unauthorized access, data download/extraction, introduction of contaminants, damage to computer systems, or denial of access. Any scan conducted without written authorization constitutes a statutory violation subject to civil compensation.
   - **Section 66 (Computer Related Offences)**: Prohibits hacking, unauthorized modification, or dishonestly accessing systems.
   - **Section 66C & 66D**: Prohibits identity theft and cheating by personation.
   - **Section 66E**: Prohibits violation of bodily privacy and capturing unauthorized private information.
   - **Section 70 & 70B**: Strict protection of Protected Systems and mandatory cybersecurity incident reporting to CERT-In.
2. **Information Technology (Intermediary Guidelines and Digital Media Ethics Code) Rules, 2021**:
   - Mandatory prohibition against sharing, uploading, or distributing unlawful, harassing, defamatory, or harmful materials or exploit payloads.
3. **Bharatiya Nyaya Sanhita (BNS) / Indian Penal Code (IPC)**:
   - Criminal provisions concerning criminal trespass, mischief, electronic tampering, and unlawful surveillance.

---

## 5. Acceptable Use Policy: Mandatory DO's and DON'Ts

To ensure safe, lawful, and defensive operation, every user and operator must adhere to the following mandatory rules of engagement:

### DO's (Authorized & Defensive Use Only)
- **DO obtain prior, written, explicit authorization**: Only deploy sensors or run discovery scans on networks, hosts, APIs, cloud subscriptions, and applications that you legally own or for which you have signed, documented authorization from the asset owner.
- **DO prioritize passive observation**: Rely primarily on passive telemetry ingestion (network taps, mirrored traffic, eBPF filters, reverse proxy access logs, and gateway trace streams) to discover APIs without sending packets to targets.
- **DO verify authorization for active verification**: If using approved active verification modules (`shadowapi-verifier`), restrict tests strictly to pre-approved non-production environments or authorized change windows, enforce strict rate limiting, and maintain continuous operator supervision.
- **DO enforce emergency kill switches**: Maintain immediate operational capability to trigger the emergency verification kill switch (`shadowapi-cli verifier abort`) to halt all outbound verification requests instantaneously.
- **DO enforce payload redaction & privacy**: Ensure all sensitive customer data, Payment Card data (PANs), authentication credentials, bearer tokens, and personally identifiable information (PII) are scrubbed before storage.
- **DO provide constructive feedback**: Submit technical feedback, architectural critiques, bug reports, and telemetry benchmarks via GitHub Issues during this 6-month evaluation POC.

### DON'Ts (Strictly Prohibited & Unlawful Activities)
- **DON'T scan third-party or unauthorized targets**: NEVER point discovery tools, active verifiers, or network probes against third-party endpoints, external vendors, SaaS platforms, or internet-facing domains without explicit legal authority. Unauthorized scanning is an offense under Section 43/66 of the IT Act, 2000.
- **DON'T probe government or critical information infrastructure**: Probing, scanning, or fuzzing state or national government portals, banking infrastructure, defense systems, or critical infrastructure is strictly illegal and will trigger immediate escalation to national security agencies.
- **DON'T conduct Denial-of-Service (DoS) or destructive testing**: Do not use this tool to flood endpoints, exhaust server resources, perform brute-force credential stuffing, or inject disruptive payloads.
- **DON'T tap networks without administrative authority**: Do not install capture agents or eBPF hooks on infrastructure where you do not possess administrative ownership or where tapping violates telecommunications privacy regulations.
- **DON'T exfiltrate or store unredacted secrets**: Never extract, store, or transmit unredacted database credentials, API private keys, or customer cardholder data.
- **DON'T misrepresent the POC license**: Do not redistribute, resell, or white-label this 6-month evaluation POC as a perpetual commercial license, and do not remove copyright notices or statutory disclaimers.

---

## 6. Community Standards & Code of Conduct

Maintainers and contributors pledge to maintain a professional, respectful, and harassment-free community for everyone regardless of age, disability, ethnicity, gender identity, experience level, race, caste, or religion.

### Expected Conduct
- Communicating with courtesy, professionalism, and empathy.
- Providing constructive, respectful technical feedback.
- Focusing on platform stability, enterprise security, and defensive integrity.

### Unacceptable Conduct
- Trolling, insulting, derogatory, or personal attacks.
- Harassment, intimidation, stalking, or discriminatory language.
- Doxxing (publishing private information without consent).
- Submitting malicious pull requests, trojanized components, or exploit payloads.

---

## 7. Entity Status & Contact Information Policy

Please review the following administrative clarification:
- **Project Legal Status**: This project is an **independent, open technical initiative** currently in its 6-month public Proof of Concept (POC) evaluation phase and is **not an incorporated corporate entity**.
- **Privacy & No Contact Information Sharing**:
  - In order to safeguard individual maintainer privacy, prevent unauthorized solicitation, and maintain security, **no personal telephone numbers, personal physical/residential addresses, or personal email addresses are published or shared anywhere in this repository or associated documentation**.
  - All legitimate technical communications, POC feedback, and statutory grievances must be routed exclusively through repository channels as detailed below.

---

## 8. Grievance Redressal & Enforcement Mechanism

In compliance with the grievance redressal principles under the Information Technology (Intermediary Guidelines and Digital Media Ethics Code) Rules, 2021:

1. **Filing a Grievance or Reporting Misuse**:
   - If you encounter a violation of this Code of Conduct, suspect unauthorized usage, or wish to report a legal or security concern, open a **Private Security Advisory** or submit an issue labeled `[Grievance / Code of Conduct]` directly within this GitHub repository.
2. **Statutory Response Timelines**:
   - Maintainers will acknowledge the receipt of any formal grievance within **forty-eight (48) hours**.
   - Maintainers will investigate and take appropriate remediation or response measures within **fifteen (15) calendar days** from the date of acknowledgment.
3. **Enforcement Powers**:
   - Maintainers reserve the right to remove, edit, or reject any comment, issue, pull request, or contribution that violates this policy, and to permanently ban repeat offenders.

---

*Document Version: 2.0 (POC Evaluation Release)*  
*Effective Date: October 2026*  
*Jurisdiction: Republic of India*
