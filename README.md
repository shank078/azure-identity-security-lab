# 🔐 Azure Identity Security & Incident Response Lab
### *A full Red Team compromise and Blue Team recovery — built, broken, and fixed by one person.*

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Entra ID](https://img.shields.io/badge/Entra_ID-00BCF2?style=for-the-badge&logo=microsoft&logoColor=white)
![Security](https://img.shields.io/badge/Blue_Team-1B3A5C?style=for-the-badge&logo=shield&logoColor=white)

---

## What This Project Is (And Why I Built It)

Most cloud security content teaches you how to *configure* things. This project is about what happens when configuration alone isn't enough.

I designed this lab to simulate a real corporate Microsoft Entra ID environment, then deliberately attack it using a technique that bypasses a control most organisations think is bulletproof: **MFA**. The goal was to understand — at a technical and operational level — exactly how an attacker exploits the gap between *enabling* a security control and *enforcing* it.

This isn't a CTF walkthrough. Every decision mirrors a real-world scenario: budget constraints, identity architecture tradeoffs, the urgency of incident response, and the importance of an audit trail that holds up under scrutiny.

---

## Technical Stack

| Layer | Technology |
|---|---|
| Cloud Provider | Microsoft Azure |
| Identity Platform | Microsoft Entra ID (formerly Azure AD) |
| Security Controls | Multi-Factor Authentication (MFA), Security Groups, RBAC |
| Monitoring | Entra ID Sign-in Logs, Audit Logs, Identity Protection |
| Attack Tooling | VPN (Geographic Spoofing), Private Session Hijacking |

---

## The Full Story: A 15-Day Security Sprint

This lab runs in five phases. Each phase builds directly on the last.

---

### 🏗️ Phase 1 — Environment Setup

Every secure environment starts with a solid identity foundation. I provisioned a dedicated lab directory and established two distinct user personas from day one:

- **Admin Account** — elevated privileges, management tasks only
- **Standard User** — least-privilege employee account used as the attack target

I also configured a dedicated **Security Group** for policy targeting, keeping the blast radius of any changes controlled and auditable from the start.

**The real-world constraint I hit immediately:** Azure's free tier doesn't give you Conditional Access. That's a Premium P2 feature. Rather than parking the project, I documented the roadblock and pivoted — which is exactly what engineers do in production when the budget doesn't match the security roadmap.

| Screenshot | What It Shows |
|---|---|
| `01_entra_user_provisioning.png` | Creating the Admin and Standard user accounts |
| `02_security_group_configuration.png` | Setting up the test group for policy targeting |
| `03_licensing_constraint_documentation.png` | The tenant creation roadblock — documented, not hidden |

---

### 🔒 Phase 2 — Identity Hardening

With the environment provisioned, I moved to hardening. This is where the free-tier constraint forced a real engineering decision.

**The "Free-Tier" Pivot:** Conditional Access — the gold standard for enforcing MFA policies — sat behind a P2 licence wall. So I implemented **Per-User MFA enforcement** instead. Not the ideal enterprise solution, but a legitimate, effective control using only the tools available. In a constrained environment, a good engineer works with what they have.

**Verification:** Before moving to the attack phase, I confirmed the hardening was effective. Attempting a login without completing MFA registration returned a mandatory "More Information Required" block. The perimeter held — for now.

| Screenshot | What It Shows |
|---|---|
| `04_premium_feature_restriction.png` | Conditional Access locked behind P2 — the licensing wall |
| `05_manual_mfa_enforcement.png` | Activating Per-User MFA as a free-tier bypass |
| `06_mfa_login_verification.png` | Verifying the "More Information Required" block is active |

---

### 💀 Phase 3 — The Compromise (Red Team)

Here's what most organisations miss: **MFA *Enabled* is not the same as MFA *Enforced*.**

When a user has MFA enabled but hasn't completed registration yet, there is a window. A small one — but an attacker who moves first wins.

I acted as an insider threat with access to stolen credentials and exploited that exact gap:

**Step 1 — MFA Hijack**
Logged into the target account *before* the legitimate user registered their own device. Registered my authenticator as the primary MFA method. The account now trusted my device, not theirs.

**Step 2 — Establishing Persistence**
Performed an administrative password reset from inside the account. The legitimate user is now locked out completely — they can't log in, and they can't recover via MFA because the attacker controls that too.

**Step 3 — Full Account Takeover (ATO)**
Complete. The victim has no path back in without admin intervention. This is a documented real-world technique used in Business Email Compromise (BEC) campaigns targeting Microsoft 365 environments.

| Screenshot | What It Shows |
|---|---|
| `07_attacker_mfa_registration.png` | Initiating the hijack — attacker's view of the MFA setup screen |
| `08_unauthorized_mfa_method_added.png` | Attacker's device successfully linked as the trusted MFA method |
| `09_persistence_via_password_reset.png` | Attacker resetting the password to lock the legitimate user out |

---

### 🔍 Phase 4 — Detection & Analysis (Blue Team)

Switching hats. I'm now the SOC Analyst who just got the ticket: a user is locked out of their account and doesn't know why.

**Making it realistic:** The attack was conducted over a VPN exiting in **Seattle, WA**. My legitimate admin session was running from **Australia**. That geographic gap was intentional — it's precisely the kind of signal that Entra ID Sign-in Logs capture and that a trained analyst knows to chase.

**The Investigation:**

1. Pulled the **Sign-in Logs** and filtered for the compromised account
2. Identified an **Impossible Travel** scenario — the same account authenticated from Australia and Seattle within minutes of each other. Physically impossible without a VPN or a very fast plane.
3. Pinned the exact **attacker IP address** and the precise timestamp of the rogue MFA registration
4. Cross-referenced the **Audit Logs** — found the MFA device addition and the subsequent password reset

The Sign-in log told me *who* and *where*. The Audit log told me *what*. Together, they rebuilt the entire attack timeline.

| Screenshot | What It Shows |
|---|---|
| `10_impossible_travel_vpn_log.png` | The smoking gun — Sign-in log showing Seattle, WA login from an AU account |
| `11_authentication_metadata_analysis.png` | Confirming the rogue MFA registration in the authentication metadata |

---

### 🛡️ Phase 5 — Incident Response & Recovery

Detection means nothing without a documented, repeatable response. I followed a standard IR workflow: **Contain → Eradicate → Recover → Document.**

**Contain — Global Session Revocation**
Immediately terminated every active session on the compromised account. The attacker was kicked out of all active portal sessions simultaneously — mid-session, no warning, no graceful exit.

**Eradicate — Rogue MFA Method Deletion**
Removed the attacker's device from the account's authentication methods. The backdoor is gone.

**Recover — Identity Restoration**
Performed a final administrative password reset to hand control back to the legitimate user with clean credentials.

**Document — Audit Trail Verification**
Confirmed every remediation action appeared correctly in the Audit Logs with accurate timestamps. In a real incident, this is what you hand to your CISO, legal counsel, or a regulator. If it isn't in the logs, it didn't happen.

| Screenshot | What It Shows |
|---|---|
| `12_global_session_revocation.png` | Admin revoking all active sessions — attacker evicted |
| `13_rogue_mfa_method_deletion.png` | Removing the attacker's phone from the trusted authentication methods |
| `14_administrative_identity_recovery.png` | Performing the final admin password reset |
| `15_remediation_success_confirmation.png` | Password Reset Successful — account restored |
| `16_final_audit_trail_verification.png` | Audit log showing every admin action taken during the incident |

---

## Key Findings & Lessons Learned

### 1. The MFA Gap Is Real — And Widely Underestimated

Organisations that enable MFA and consider the job done are leaving a window open. The attack in Phase 3 required zero malware, zero exploits, and zero technical sophistication. Just timing and stolen credentials.

**The fix:** Enforce **Registration Campaigns** immediately upon account creation so users are pushed to register before an attacker can. Or use **Conditional Access with Trusted Locations** (P2) to block authentication from unrecognised regions entirely.

### 2. Two Log Sources. One Complete Picture.

Most analysts pull one log and stop there.

- **Sign-in Logs** → Behavioural anomalies. Impossible travel, unfamiliar locations, repeated failures, new device fingerprints.
- **Audit Logs** → Administrative actions. MFA method changes, password resets, role assignments, group membership changes.

You need both to reconstruct a full attack timeline. Either one alone leaves gaps a good attacker can hide in.

### 3. A Constrained Environment Is Not a Defenceless One

Premium licences unlock powerful automation. But Per-User MFA, Sign-in Logs, and Audit Logs exist at every tier. The tools here cost nothing. What they require is an analyst who knows how to use them.

---

## What's Next

This lab deliberately ends with manual hunting and manual response. The natural next iteration is automation:

- **Integrate Microsoft Sentinel** — replace manual log review with automated analytic rules that fire on impossible travel and MFA registration anomalies
- **Build KQL Watchlists** — flag high-risk IP ranges and known VPN exit nodes automatically
- **Automate Containment via Playbooks** — trigger Global Session Revocation the moment an anomaly is detected, cutting attacker dwell time from hours to seconds

*(The foundation for this is already live — see my [Azure Sentinel Honeypot & SIEM project](https://github.com/shank078/azure-sentinel-honeypot-siem))*

---

## Repository Structure
