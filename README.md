# Azure Identity Security & Incident Response Lab

### A red-team account takeover in Microsoft Entra ID, then the blue-team detection and recovery — built, broken, and fixed in one lab.

<p align="left">
  <img src="https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoftazure&logoColor=white"/>
  <img src="https://img.shields.io/badge/Entra_ID-00BCF2?style=for-the-badge&logo=microsoft&logoColor=white"/>
  <img src="https://img.shields.io/badge/Red_Team-MFA_Hijack-E63946?style=for-the-badge&logo=shield&logoColor=white"/>
  <img src="https://img.shields.io/badge/Blue_Team-Incident_Response-1B3A5C?style=for-the-badge&logo=shield&logoColor=white"/>
  <img src="https://img.shields.io/badge/MITRE_ATT%26CK-T1098.005-E63946?style=for-the-badge&logo=shield&logoColor=white"/>
</p>

---

## TL;DR — What Happened

| Phase | Action | Result |
|-------|--------|--------|
| **Setup** | Provisioned an Entra ID lab with Admin + Standard user personas | Clean identity baseline |
| **Hardening** | Deployed Per-User MFA as the free-tier Conditional Access alternative | MFA block confirmed active |
| **Attack** | Exploited the MFA *enabled* vs *enforced* gap — hijacked MFA, reset the password | Full account takeover |
| **Detection** | Pulled Sign-in Logs + Audit Logs | Impossible travel found (Australia → Seattle); timeline rebuilt |
| **Response** | Contain → Eradicate → Recover → Document | Sessions revoked, rogue MFA removed, account restored, audit trail verified |

The full timestamped incident write-up is in [`documentation/incident-response-log.md`](documentation/incident-response-log.md).

---

## What This Project Is (And Why I Built It)

Most cloud security content teaches you how to *configure* things. This project is about what happens when configuration alone isn't enough.

I built this lab to simulate a small corporate Microsoft Entra ID environment, then attacked it using a technique that gets past a control most organisations trust by default: MFA. The goal was to understand, at a technical and operational level, how an attacker exploits the gap between *enabling* a control and *enforcing* it — and then to work the incident from the other side.

It's not a CTF walkthrough. The decisions along the way mirror real ones: a budget constraint that ruled out Conditional Access, the pressure of an account-lockout ticket, and an audit trail that has to hold up afterwards.

---

## Technical Stack

| Layer | Technology |
|---|---|
| **Cloud Provider** | Microsoft Azure |
| **Identity Platform** | Microsoft Entra ID (formerly Azure AD) |
| **Security Controls** | Multi-Factor Authentication (MFA), Security Groups, RBAC |
| **Monitoring** | Entra ID Sign-in Logs, Audit Logs, Identity Protection |
| **Attack Technique** | VPN geographic relocation, MFA registration hijack |
| **MITRE ATT&CK** | T1078 (Valid Accounts), T1556.006 (Modify Auth Process: MFA), T1098.005 (Account Manipulation: Device Registration) |

---

## Walkthrough

### Phase 1 — Environment Setup

I provisioned a dedicated lab directory with two user personas:

- **Admin account** — elevated privileges, used for management only
- **Standard user** — least-privilege employee account, used as the attack target

A **Security Group** was set up for policy targeting so any changes stayed scoped and auditable.

> **Constraint hit immediately:** Azure's free tier doesn't include Conditional Access — that's a Premium P2 feature. Rather than park the project, I documented the roadblock and pivoted to Per-User MFA, which is the kind of tradeoff you make when the budget doesn't match the security roadmap.

**Creating the Admin and Standard user accounts**

![01_entra_user_provisioning](images/01_entra_user_provisioning.png)

**Setting up the Security Group for policy targeting**

![02_security_group_configuration](images/02_security_group_configuration.png)

**The licensing constraint — documented, not hidden**

![03_licensing_constraint_documentation](images/03_licensing_constraint_documentation.png)

---

### Phase 2 — Identity Hardening

Conditional Access is the standard way to enforce MFA, but it sat behind the P2 licence wall, so I used **Per-User MFA enforcement** instead — not the ideal enterprise control, but a legitimate one with the tools available on the free tier.

Before moving on, I confirmed the hardening worked: attempting a login without completing MFA registration returned the mandatory "More Information Required" block.

**Conditional Access locked behind P2 — the licensing wall**

![04_premium_feature_restriction](images/04_premium_feature_restriction.png)

**Activating Per-User MFA as the free-tier alternative**

![05_manual_mfa_enforcement](images/05_manual_mfa_enforcement.png)

**Verifying the "More Information Required" block is active**

![06_mfa_login_verification](images/06_mfa_login_verification.png)

---

### Phase 3 — The Compromise (Red Team) &nbsp; `T1078 — Valid Accounts` · `T1098.005 — Device Registration` · `T1556.006 — MFA`

The gap this exploits: **MFA *enabled* is not the same as MFA *enforced*.** When a user has MFA switched on but hasn't completed registration yet, there's a window, and whoever registers a device first controls the second factor.

**Step 1 — MFA hijack.** Logged into the target account before the legitimate user registered their own device, and registered my authenticator as the account's MFA method. The account now trusts my device.

**Step 2 — Persistence.** Did an administrative password reset from inside the account. The real user is now locked out — they can't log in, and they can't self-recover, because the MFA channel is mine.

**Step 3 — Account takeover complete.** No path back in without admin intervention. This is the same pattern used in Business Email Compromise campaigns against Microsoft 365.

**Attacker's view of the MFA setup screen**

![07_attacker_mfa_registration](images/07_attacker_mfa_registration.png)

**The attacker device registered as the account's MFA method**

![08_unauthorized_mfa_method_added](images/08_unauthorized_mfa_method_added.png)

**Password reset from inside the account — legitimate user locked out**

![09_persistence_via_password_reset](images/09_persistence_via_password_reset.png)

---

### Phase 4 — Detection & Analysis (Blue Team) &nbsp; `T1078.004 — Cloud Accounts`

Switching sides. The ticket: a user is locked out and doesn't know why.

To make it realistic, the attack ran over a VPN exiting in **Seattle, WA**, while the account normally signs in from **Australia**. That geographic gap is exactly the signal a Sign-in log captures and an analyst learns to chase.

Rather than stop at the dashboard graphs, I went into the raw event properties:

- **Cross-log correlation** — pivoted from the anomaly in the Sign-in Logs (identity: where the login came from) to the mechanism in the Audit Logs (what changed on the account).

| Log Source | Field | What it showed |
|------------|-------|----------------|
| **Sign-in Logs** | `Location` / `IP address` | Impossible travel — Australia to Seattle, WA within minutes |
| **Sign-in Logs** | `Authentication Details` | MFA satisfied via a newly added, attacker-controlled method |
| **Audit Logs** | `Activity: Update User` | The rogue MFA device being added to the account |

In this run, the Seattle login came from `149.40.62.5` while the account's normal IP was `163.47.70.26` in Australia. The full timestamped timeline — including the UTC-vs-local detail between the two log blades — is in [`documentation/incident-response-log.md`](documentation/incident-response-log.md).

**Sign-in log: the Seattle login on an account that's normally in Australia**

![10_impossible_travel_vpn_log](images/10_impossible_travel_vpn_log.png)

**Confirming the rogue MFA registration in the authentication metadata**

![11_authentication_metadata_analysis](images/11_authentication_metadata_analysis.png)

---

### Phase 5 — Incident Response & Recovery &nbsp; `NIST SP 800-61 r2`

**Contain — global session revocation.** Terminated every active session on the account, so the attacker's portal session died mid-session.

**Eradicate — rogue MFA method deletion.** Removed the attacker's device from the account's authentication methods. This has to come before the password reset — otherwise the attacker can self-recover through the MFA channel they still control.

**Recover — password reset.** An administrative password reset returned a clean login to the real user.

**Document — audit trail verification.** Confirmed every remediation action shows up in the Audit Logs with a timestamp. In a real incident, that trail is what you hand over afterwards.

**Revoking all active sessions — attacker evicted**

![12_global_session_revocation](images/12_global_session_revocation.png)

**Removing the attacker's device from trusted authentication methods**

![13_rogue_mfa_method_deletion](images/13_rogue_mfa_method_deletion.png)

**Administrative password reset**

![14_administrative_identity_recovery](images/14_administrative_identity_recovery.png)

**Reset successful — account restored**

![15_remediation_success_confirmation](images/15_remediation_success_confirmation.png)

**Audit log showing every admin action taken during the incident**

![16_final_audit_trail_verification](images/16_final_audit_trail_verification.png)

---

## Key Findings & Lessons Learned

### 1. The MFA gap is real and easy to underestimate
The attack needed no malware and no exploit — just timing and a set of valid credentials against a control that was enabled but not enforced. Enabling MFA and calling the job done leaves this window open. The fix is an **authentication method registration campaign** so users register before an attacker can, or **Conditional Access with trusted locations** (P2) to block the unfamiliar region outright.

### 2. Two log sources, one picture
The Sign-in Logs told me *where* — the impossible-travel login. The Audit Logs told me *what* — the rogue MFA device being added. Either one alone leaves a gap: a strange login with no known action, or an account change with no idea who made it. You need both to rebuild the timeline.

### 3. A constrained environment isn't a defenceless one
Per-User MFA, Sign-in Logs, and Audit Logs all exist at every licence tier. Premium unlocks automation, but the detection and response here cost nothing beyond knowing which logs to read and in what order.

---

## What's Next

This lab ends with manual hunting and manual response. The natural next step is automation:

- **Microsoft Sentinel** — an analytic rule that fires on impossible travel and MFA-registration anomalies instead of me reading raw logs
- **KQL watchlists** — flag known VPN exit ranges automatically
- **Logic Apps containment** — trigger session revocation the moment an anomaly is detected, cutting dwell time

*The Sentinel side of this is already built — see the [Dual SIEM Detection Lab](https://github.com/shank078/Dual-SIEM-Detection-Lab) and [Azure Sentinel Honeypot SIEM](https://github.com/shank078/azure-sentinel-honeypot-siem).*

---

## Repository Structure

```
azure-identity-security-lab/
├── README.md
├── documentation/
│   └── incident-response-log.md    # full timestamped IR write-up
└── images/                          # 16 screenshots — setup, attack, detection, response
```

---

## Related Projects

| Project | Description |
|---------|-------------|
| [Dual SIEM Detection Lab](https://github.com/shank078/Dual-SIEM-Detection-Lab) | 5 MITRE-mapped detections across Sentinel + Splunk on live attacker traffic |
| [Azure Sentinel Honeypot SIEM](https://github.com/shank078/azure-sentinel-honeypot-siem) | 1,400+ real brute-force attempts captured and mapped globally |
| [SOAR Pipeline — Sentinel to Jira](https://github.com/shank078/azure-sentinel-jira-soar-pipeline) | Automated incident ticketing from Sentinel |

---

## About

Built and documented by **Shankar Baral** — junior SOC analyst in Canberra, Australia. More about me and my other labs: [github.com/shank078](https://github.com/shank078) · [LinkedIn](https://www.linkedin.com/in/shankarbaral1)
