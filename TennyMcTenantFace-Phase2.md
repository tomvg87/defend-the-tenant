# Defend The Tenant – Phase 2 \
Tenny McTenantFace

## The customer

- About 100 users who work from anywhere and meet with clients and partners in Teams.
- Two admins who also run day-to-day IT. No Security Operations Center.
- Microsoft 365 E7. Entra ID P2 (Conditional Access, ID Protection, PIM), Intune, Defender, Purview, the Entra Suite, Copilot and Agent 365 are already licensed.
- The current gap is configuration (only two admins) and not budget (E7 licenses)

## How I ranked the security controls

I ranked by **how likely the attack is and/or how much damage it does**, and then by **effort for two admins**.
Every Cpnditional Access policy follows the same pattern:

1. Create two cloud-only **break-glass accounts**. They are the only exclusion in every policy, so a mistake can never lock them out.
2. Turn the policy on in **report-only** mode and check it with **What If**, both with a regular user and a break-glass account
3. Pilot with the two admins and a few volunteers, then enable the policy for everyone, one policy at a time.

| Rank | Control | Main risk it mitigates | Admin effort |
|---|---|---|---|
| 1 | MFA for everyone, phishing-resistant for admins | Phishing, password spray, account takeover | Medium (mostly user registration) |
| 2 | Least privilege access with PIM | One compromised admin = full tenant takeover | Low |
| 3 | Block legacy authentication | MFA bypass through old protocols | Low–medium |
| 4 | Risk-based Conditional Access | Attacks nobody is watching for | Low |
| 5 | App consent governance | Consent phishing, over-permissioned apps and AI agents | Low |

---

## 1. Enforce MFA for all users, with phishing-resistant MFA for admins

**Why:**
- "Hackers don't break in; they log in": almost every breach starts with a stolen or guessed password. MFA on all users and all resources stops the vast majority of these attacks.
- Admins get a stronger requirement: **phishing-resistant MFA** (passkeys/FIDO2 or Windows Hello for Business). Admin accounts are the most valuable target so an extra layer of security is added for them.

**Rollout:**
- **Week 1:**
  - Enable Microsoft Authenticator and passkeys in the Authentication methods policy.
  - Turn on registration at next sign-in.
  - Send one email that explains why and what to expect.
- **Week 1–2:**
  - Track progress in the *Authentication methods → User registration details* report.
  - Give the few people who get stuck a **Temporary Access Pass** so they can start registering their MFA authentication methods.
- **Week 2–3:**
  - Create **CA – MFA for admin roles**: target directory roles, not a group, so new admins are covered automatically. Use *Phishing-resistant MFA* authentication strength once both admins have a passkey.
  - Create **CA – MFA for all users** once all users have their authentication methods set.
  - Run both in report-only for a week and check sign-in logs daily, then switch them On.
- **Effort**: about 1–2 admin days, spread over three weeks. Most of the work is done by the users themselves.

**Drawbacks & tradeoffs:**
- The first week(s) will have more questions popping up from users who aren't familiar with MFA methods, which is why it's important to send a company-wide email with necessary info first.
- Users without a company smartphone will need to install an authenticator app on their personal device, or use alternatives such as a FIDO2 key or TAP.
- **MFA fatigue.** Users just want to log in, having to go to their phone to find/type another number is annoying for them but necessary for security.
- **Lockout risk** if a policy is wrong. Break-glass accounts and report-only mode mitigate it.
- **Cost and logistics of FIDO2 keys** for admins and break-glass accounts. Phishing-resistant MFA for all users is the long-term goal, but it is too much change at once for two admins.

## 2. Apply least privilege access with Privileged Identity Management (PIM)

**Why:**
- Small tenants often have far more Global Administrators than admins. My challenge tenant had **six** permanent Global Administrator user accounts plus three service principals with the same role, for a company with two admins.
- Every standing Global Administrator is a full-takeover target 24/7. Least privilege plus just-in-time access means admin power exists only while it is used, and every activation is logged with a reason. It is included in E7 (Entra ID P2).

**Rollout:**
- Export the current role assignments as a .CSV.
- Keep two permanent Global Administrators: the break-glass accounts.
- Give the two named admins eligible roles in PIM:
  - Day-to-day roles for normal work: User, Helpdesk, Exchange, SharePoint, Teams Administrator etc.
  - Global Administrator only for rare tasks: maximum 4 hours, MFA and a justification on activation, and a notification to the other admin.
- Remove Global Administrator from service principals. Give each service principal only the permission its task needs.
- Remove a broad role only after its replacement role has been tested.
- Schedule a quarterly access review of privileged roles through Entra ID Governance (it takes 15 minutes)
- **Effort**: about half a day to set up, plus 15 minutes per quarter for the access review.

**Drawbacks and tradeoffs:**
- **Activation adds some delay**, which is annoying in an urgent situation. The break-glass accounts can be used as a fallback.
- **Approval-based activation does not work well with only two admins.** If one is on holiday, the other can't get approval. So I recommend **self-activation with MFA, a justification and a notification** instead of approval.
- **Finding the smallest role that works (within the dozens of different Entra roles) takes some trial and error** in the first weeks.

## 3. Block legacy authentication

**Why:**
- Old protocols (POP, IMAP, SMTP AUTH etc.) cannot do MFA. Attackers use them deliberately because a password alone still works there.
- Without this control, control 1 can be bypassed.

**Rollout:**
- First, filter the Entra sign-in logs on legacy client apps to see who still uses them. For instance, an old mail client or a script.
- Create **CA – Block legacy authentication** for the client app types *Exchange ActiveSync* and *Other clients*. Run it in report-only for 1–2 weeks, fix what shows up, then switch it On.
- Add defense in depth: disable SMTP AUTH in Exchange Online tenant-wide. This can be done in the Exchange Admin Center or Exchange Online PowerShell.
- **Effort**: a few hours, plus whatever the old devices need.

**Drawbacks and tradeoffs:**
- **Some legacy printers, scripts or mail clients will break.** They need to be reconfigured to use OAuth, Microsoft Graph or an SMTP relay connector, or be replaced.

## 4. Turn on risk-based Conditional Access (sign-in risk and user risk)

**Why:**
- With only two admins and no SOC, nobody notices an impossible-travel sign-in or leaked credentials in time. Entra ID Protection detects them, and Conditional Access responds automatically:
  - **Sign-in risk medium/high:** require fresh MFA *every time*. An attacker with a valid password is challenged at the moment of risk.
  - **User risk high:** require MFA and a secure password change. The leaked password stops working on the user's next sign-in, without either admin having to reset anything.

**Rollout:**
- Create two policies, one for sign-in risk and one for user risk, so each has one purpose and is easy to troubleshoot.
- Run them in report-only for a week and check the Risky sign-ins report for false positives, for example VPN users and travellers.
- Users must be MFA-registered (control 1), or a flagged user gets blocked instead of remediating themselves.
- A 15-minute weekly look at risky users plus an email alert for high risk.
- **Effort**: about 2 hours to set up.

**Drawbacks and tradeoffs:**
- **False positives.** A user on a hotel network or a VPN may get an extra prompt or have to change their password. It's annoying, but they can solve it themselves.
- It relies on Microsoft's detection signals, which can't be fine-tuned by the admins.

## 5. Govern application consent (no user consent + admin consent workflow)

**Why:**
- If a user clicks "Accept" on a malicious app, that app then gets long-lived access to mail and files through tokens. **MFA does not stop this since the user already passed the MFA check.**
- Copilot extensions and AI agents increasingly ask for broad Microsoft Graph permissions. Governing consent means only reviewed apps and agents get access to company data.

**Rollout:**
- Set user consent to **Do not allow**.
- Enable the **admin consent workflow**, with both admins as reviewers, email notifications, and a 30-day expiry on requests.
- Do a **one-time review of existing grants**: Enterprise applications → Permissions, or app governance in Defender for Cloud Apps. Give each app an owner, then keep or revoke it.
- Use Agent 365 to inventory AI agents and give each one an owner.
- **Effort**: 1 hour for the settings, plus half a day for the one-time review. After that, a few minutes per request.

**Drawbacks and tradeoffs:**
- **Users have to wait** for approval, which can feel slow. If requests take too long, people turn to shadow IT, so a response time of 1–2 working days is needed.
- **Reviewers need to understand permissions.** A simple rule helps, for example "read-only and verified publisher = OK, but mail/files write = ask questions".
- The setting only stops **new** grants. Existing ones need the one-time review.

---

## Just outside the top five

These are worth doing once the top five are live, and they are cheap with E7:
- **Self-service password reset for all users.** It means fewer tickets for the help desk as users are able to reset their own passwords.
- **Teams meeting settings:** only invited users bypass the lobby, only organizers present, no anonymous join. This one's priority depends on the company's Teams usage rate.
- **Break-glass sign-in alerts.** Make sure their sign-ins are legitimate since they hold a standing Global Admin role
- **Defender for Office 365 preset policies.** Implement useful features such as Safe Attachments and Safe Links
- **Intune device compliance** in Conditional Access. Make sure devices are up to date in terms of OS, BitLocker, antivirus software, etc.
- **Purview sensitivity labels and oversharing review** Make sure data within the company is only visible and shareable by those with proper access.

## Expected result

After about **6–8 weeks** in total, these improvements should be visible:
- from "a password opens everything" to verified identity on every sign-in, with phishing-resistant admins
- from nine accounts that can take over the tenant to two monitored break-glass accounts
- no MFA bypass routes, automatic response to risky sign-ins and leaked credentials, and reviewed access for every app and AI agent.

If the admins are ever unsure about which control to implement next, the Microsoft Learn website has a lot of great documentation on security baselines:
https://learn.microsoft.com/en-us/microsoft-365/baseline-security-mode/baseline-security-mode-settings?view=o365-worldwide
