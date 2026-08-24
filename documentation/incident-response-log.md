# Incident Response Log — Entra ID Account Takeover (Lab)

This is my write-up for the account-takeover exercise in this repo. I ran the whole thing in my own Entra ID lab directory and played both sides: first the attacker who takes over an account through the MFA gap, then the analyst who has to figure out what happened and clean it up.

The times below come straight off my Sign-in and Audit log screenshots. One thing that caught me out early: the two log blades were set to different timezones, so I've noted which is which and lined them up by hand.

## Summary

The Standard User account was taken over on 11 May 2026. The attacker signed in from a Seattle VPN exit while the account normally signs in from Australia, registered their own Microsoft Authenticator as the account's MFA method, and reset the password to lock the real user out. I caught it from the impossible-travel sign-in, confirmed the rogue MFA device in the Audit logs, revoked the sessions, deleted the attacker's MFA method, and reset the password back.

## Key details

| Field | Value |
|---|---|
| Date | 11 May 2026 |
| Affected account | `standard.user` (Standard User persona) |
| Account used for response | lab global admin |
| Attacker source IP | 149.40.62.5 — Seattle, Washington (VPN exit) |
| Normal source IP | 163.47.70.26 — Glebe, NSW, Australia |
| Technique | MFA registration hijack, then admin password reset |
| Severity | High — full account takeover |
| MITRE ATT&CK | T1078 (Valid Accounts), T1556.006 (Modify Auth Process: MFA), T1098.005 (Account Manipulation: Device Registration) |

A note on the timestamps, because it matters for the timeline: the **Sign-in logs** screenshot is set to **UTC**, and the **Audit logs** screenshot is set to **Local (AEST, UTC+10)**. So the Seattle sign-in at `08:46:15 UTC` is the same moment as `6:46 PM` local, and the response actions in the Audit log (`6:56`–`6:57 PM` local) are about ten minutes after that. I had both blades open side by side and converted to local to work out the order.

## Timeline

| Time | Source | What happened | Where I saw it |
|---|---|---|---|
| ~08:36 UTC | 163.47.70.26 (AU) | Standard User doing normal "My Profile" activity from Australia | Sign-in logs |
| during the compromise | attacker | Registered the attacker's Microsoft Authenticator as the account's MFA method | Auth methods / Audit logs |
| during the compromise | attacker | Reset the account password, locking the real user out | Attack phase |
| 08:46:15 UTC | 149.40.62.5 (Seattle) | Standard User signs in to the Azure Portal from Seattle — about 10 minutes after being active from Australia. This is the impossible travel that flagged it. | Sign-in logs |
| 6:56:48 PM (AEST) | admin | Deleted the attacker's Microsoft Authenticator method ("Admin deleted Microsoft Authenticator App") | Audit logs |
| 6:57:22 PM (AEST) | admin | Admin password reset — status "Successfully completed reset" | Audit logs |

## How I detected it

I needed two log sources, not one.

**Sign-in logs — the where.** The Standard User account signed in from Seattle (149.40.62.5) at 08:46:15 UTC, ten minutes after the same account was active from Glebe, NSW (163.47.70.26). You can't get from Australia to Seattle in ten minutes, so that's the impossible travel. The Conditional Access column on that row says "Not applied", which is the whole problem — nothing stopped the login.

**Audit logs — the what.** The sign-in log tells you someone logged in from an odd place, but not what they did with the account. The Audit log showed the attacker's Microsoft Authenticator method being added. That's how they got past MFA: the account had MFA, the attacker just owned the second factor.

## What I did to respond

I worked through it in the NIST SP 800-61 order (Contain → Eradicate → Recover → Document):

1. **Contain** — revoked all active sessions on the account, which killed the attacker's portal session straight away.
2. **Eradicate** — deleted the attacker's Microsoft Authenticator method from the account (Audit log, 6:56:48 PM). Until that method is gone, a password reset on its own doesn't fix anything, because the attacker still controls the second factor.
3. **Recover** — did an admin password reset (Audit log, 6:57:22 PM) to hand a clean login back to the real user.
4. **Document** — checked that every action I took shows up in the Audit log with a timestamp. In a real incident that trail is what you hand over afterwards.

The order was the part I actually had to think about. If I'd reset the password first and left the rogue MFA method in place, the attacker could still have gone through the "forgot password" flow using the MFA channel they controlled. Kill the MFA method first, then reset the password.

## Root cause

MFA was *enabled* on the account but not *enforced* early through a registration campaign, and there was no Conditional Access to block the unfamiliar location. (Conditional Access needs a Premium P2 licence, which the free lab tier doesn't include, so I used per-user MFA instead.) That left a gap: the attacker registered their own authenticator before the real user ever set one up. MFA was on the whole time — it just protected the wrong person.

## What I'd change

- Turn on an **authentication method registration campaign** so a new user is prompted to register their own MFA immediately, which closes the window this attack used.
- With a P2 licence, a **Conditional Access policy with trusted/named locations** would have blocked the Seattle sign-in before it succeeded.
- Longer term, an automated **impossible-travel rule in Microsoft Sentinel** (there's a note about wiring this up in the main README) so this gets caught by a rule instead of me reading raw sign-in logs by hand.

## Notes and limitations

This is a lab I built and attacked myself, so the "attacker" and the "analyst" are both me on the same tenant. The point was doing the full loop — build it, break it, detect it, respond, write it up — not the scale of it. The timeline here is reconstructed from the Sign-in and Audit log screenshots in this repo; if a time looks slightly off against another screenshot, it's the UTC-vs-local difference and not a missing step.
