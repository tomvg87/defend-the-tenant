# Defend The Tenant – Phase 1 · Tenny McTenantFace

## Introduction

The work is part of the Microsoft Defend The Tenant challenge. Phase 1 secures a preconfigured Microsoft 365 tenant (Contoso); Phase 2 turns these controls into a customer-focused hardening plan. These 10 security controls were configured:

| # | Control | Location | Result |
| --- | --- | --- | --- |
| 01 | Require MFA for admins | Entra ID → Conditional Access | `CA001` On |
| 02 | Enforce MFA for everyone | Entra ID → Conditional Access | `CA002` On |
| 03 | Block legacy authentication | Entra ID → Conditional Access | `CA003` On |
| 04 | Challenge risky sign-ins | Entra ID → Conditional Access | `CA004` On |
| 05 | Remediate high-risk users | Entra ID → Conditional Access | `CA005` On |
| 06 | Restrict app consent | Entra ID → Enterprise apps | User consent off, admin consent workflow on |
| 07 | Control Teams lobby access | Teams admin center | Bypass lobby: people who were invited |
| 08 | Limit presenter privileges | Teams admin center | Only organizers and co-organizers |
| 09 | Block anonymous meeting access | Teams admin center | Anonymous join off |
| 10 | Self-service password reset | Entra ID → Password reset | SSPR enabled for All |

**Redaction:** screenshots are blurred where they show the tenant UPN or the break-glass account names. Tenant ID and object IDs are blurred or omitted.

## Preparation

Before any policy was enforced, the tenant was prepared so that Conditional Access could work and no administrator could be locked out.

1. **Security defaults disabled.** Conditional Access policies cannot be used while security defaults are on.
2. **Two cloud-only break-glass accounts** were created with a permanent Global Administrator assignment and no licence. Both are members of the security group `SG-BreakGlass-Excluded`, the only exclusion in every Conditional Access policy.
3. **Licences verified.** Entra ID P1 covers Conditional Access and SSPR; Entra ID P2 (ID Protection) is required for the risk-based policies CA004 and CA005.
4. **Authentication methods** enabled for all users: Passkey (FIDO2), Microsoft Authenticator, Temporary Access Pass, software OATH tokens and email OTP. SMS and voice call stay disabled.

![Security defaults are disabled](images/fig01-security-defaults.png)
*Figure 1 – Security defaults disabled.*

![SG-BreakGlass-Excluded group with two members (names blurred)](images/fig02-breakglass-group.png)
*Figure 2 – Break-glass exclusion group with two members (account names blurred).*

![License usage showing Entra ID P1 and P2 entitlements](images/fig03-licenses.png)
*Figure 3 – Entra ID P1 and P2 licences available.*

![Authentication methods policies](images/fig04-auth-methods.png)
*Figure 4 – Authentication methods enabled for all users.*

**Working method for every Conditional Access policy:** create in **Report-only**, validate with the **What If** tool (admin and standard user), then switch to **On**. Policies were tested and then enabled one at a time.

## Conditional Access policies

All Conditional Access policies target **All resources** and exclude only `SG-BreakGlass-Excluded`.

### 01 – Require MFA for administrators (CA001)

CA001 requires multifactor authentication for 14 privileged directory roles, so a stolen admin password alone no longer grants access to the entire tenant.

| Setting | Value |
| --- | --- |
| Name | `CA001-Require-MFA-Admins` |
| Include | 14 directory roles (Global, Application, Authentication, Billing, Cloud Application, Conditional Access, Exchange, Helpdesk, Password, Privileged Authentication, Privileged Role, Security, SharePoint and User Administrator) |
| Exclude | `SG-BreakGlass-Excluded` |
| Grant | Require multifactor authentication |

![CA001 include: 14 directory roles selected](images/fig05-ca001-include-roles.png)
*Figure 5 – CA001 targets 14 administrative roles.*

![CA001 exclude: SG-BreakGlass-Excluded group](images/fig06-ca001-exclude.png)
*Figure 6 – Only the break-glass group is excluded.*

![CA001 grant: Require multifactor authentication](images/fig07-ca001-grant.png)
*Figure 7 – Grant control: require MFA. Require authentication strength cannot be combined with it. Once the company baseline is set, an upgrade to more phishing-resistant MFA should be implemented with "require authentication strength".*

![What If for an administrator: CA001 applies](images/fig08-ca001-whatif-admin.png)
*Figure 8 – What If for an administrator: CA001 applies.*

![What If before CA002: no policy applies to a standard user](images/fig09-whatif-before-ca002.png)
*Figure 9 – What If for a standard user before CA002: 0 policies apply, which shows the gap control 02 closes.*

### 02 – Enforce MFA for all users (CA002)

CA002 extends MFA to every user, which blocks password spray and credential-stuffing attacks against regular accounts.

| Setting | Value |
| --- | --- |
| Name | `CA002-Require-MFA-All-Users` |
| Include | All users |
| Exclude | `SG-BreakGlass-Excluded` |
| Grant | Require multifactor authentication |

![CA002 assignments](images/fig10-ca002-assignments.png)
*Figure 10 – CA002 applies to all users.*

![What If for a standard user: CA002 applies](images/fig11-ca002-whatif.png)
*Figure 11 – What If for a standard user: CA002 now applies.*

### 03 – Block legacy authentication (CA003)

CA003 blocks Exchange ActiveSync and other legacy clients. These protocols cannot perform MFA, so leaving them open would bypass CA001 and CA002.

| Setting | Value |
| --- | --- |
| Name | `CA003-Block-Legacy-Authentication` |
| Include | All users |
| Exclude | `SG-BreakGlass-Excluded` |
| Client apps | Exchange ActiveSync clients, Other clients (Browser and Mobile/desktop left unchecked) |
| Grant | Block access |

![CA003 assignments with break-glass exclusion](images/fig12-ca003-assignments.png)
*Figure 12 – CA003 assignments.*

![CA003 client apps (admin UPN blurred)](images/fig13-ca003-client-apps.png)
*Figure 13 – Only legacy client apps are targeted (the admin UPN is blurred).*

![What If input: Exchange ActiveSync client](images/fig14-ca003-whatif-input.png)
*Figure 14 – What If input: Exchange ActiveSync client.*

![What If result: CA003 blocks](images/fig15-ca003-whatif-blocked.png)
*Figure 15 – Legacy sign-in is blocked by CA003.*

![What If with browser: CA003 does not apply](images/fig16-ca003-whatif-browser.png)
*Figure 16 – A modern browser sign-in is not affected by CA003.*

### 04 – Challenge risky sign-ins (CA004)

CA004 forces an MFA prompt when ID Protection rates a sign-in as medium or high risk, such as an unfamiliar location or a known malicious IP address.

| Setting | Value |
| --- | --- |
| Name | `CA004-Require-MFA-Medium-High-SignIn-Risk` |
| Include / Exclude | All users / `SG-BreakGlass-Excluded` |
| Condition | Sign-in risk: High, Medium |
| Grant | Require multifactor authentication |
| Session | Sign-in frequency: Every time |

![CA004 sign-in risk condition: High and Medium](images/fig17-ca004-signin-risk.png)
*Figure 17 – CA004 sign-in risk levels.*

![CA004 session: sign-in frequency every time](images/fig18-ca004-session.png)
*Figure 18 – Session control: reauthenticate every time.*

![What If with medium sign-in risk](images/fig19-ca004-whatif-medium.png)
*Figure 19 – What If with medium sign-in risk: CA004 applies.*

![What If with low sign-in risk](images/fig20-ca004-whatif-low.png)
*Figure 20 – What If with low sign-in risk: CA004 does not apply.*

### 05 – Remediate high-risk users (CA005)

CA005 makes a user whose account is likely compromised prove their identity with MFA and set a new password before they can continue, which removes the attacker's known credential.

| Setting | Value |
| --- | --- |
| Name | `CA005-Remediate-High-User-Risk` |
| Include / Exclude | All users / `SG-BreakGlass-Excluded` |
| Condition | User risk: High |
| Grant | Require MFA **and** require password change (require all selected controls) |
| Session | Sign-in frequency: Every time |

Sign-in risk and user risk are kept in separate policies.

![CA005 user risk condition: High](images/fig21-ca005-user-risk.png)
*Figure 21 – CA005 user risk level.*

![CA005 grant: require MFA and password change](images/fig22-ca005-grant.png)
*Figure 22 – Grant controls for CA005.*

![What If with high user risk](images/fig23-ca005-whatif-high.png)
*Figure 23 – What If with high user risk: CA005 applies.*

![What If with no user risk](images/fig24-ca005-whatif-none.png)
*Figure 24 – What If with no user risk: CA005 does not apply.*

## Application, Teams & password controls

### 06 – Restrict application consent

Users can no longer grant third-party apps access to company data, which stops consent-phishing attacks. They can still request an app through an admin consent workflow.

| Setting | Value |
| --- | --- |
| User consent for applications | Do not allow user consent |
| Users can request admin consent | Yes |
| Reviewers | 1 user (MOD Administrator) |
| Email notifications / reminders | Yes / Yes |
| Request expires after | 30 days |

![User consent settings: Do not allow user consent](images/fig25-user-consent.png)
*Figure 25 – User consent disabled.*

![Admin consent settings](images/fig26-admin-consent.png)
*Figure 26 – Admin consent workflow enabled.*

![Selected reviewer](images/fig27-consent-reviewer.png)
*Figure 27 – Selected reviewer.*

### 07–09 – Teams meeting controls

The three Teams controls were set in the **Global (Org-wide default)** meeting policy in the Teams admin center. Together they stop uninvited or anonymous people from entering meetings and from taking over the screen.

| # | Control | Setting | Value |
| --- | --- | --- | --- |
| 07 | Lobby access | Who can bypass the lobby | People who were invited |
| 08 | Presenter privileges | Who can present | Only organizers and co-organizers |
| 09 | Anonymous access | Anonymous users can join a meeting unverified | Off |

![Who can bypass the lobby: people who were invited](images/fig28-teams-lobby.png)
*Figure 28 – Control 07: only invited people bypass the lobby.*

![Anonymous users can join a meeting unverified: Off](images/fig29-teams-anonymous-off.png)
*Figure 29 – Control 09: anonymous join switched Off.*

![Who can present: only organizers and co-organizers](images/fig30-teams-presenters.png)
*Figure 30 – Control 08: only organizers and co-organizers can present.*

### 10 – Self-service password reset

SSPR is enabled for all users, so a locked-out user can recover securely with two verified methods instead of calling a helpdesk. More authentication methods are enabled (see figure 4) than the two required to reset, so every user can meet the requirement.

| Setting | Value |
| --- | --- |
| Self service password reset enabled | All |
| Number of methods required to reset | 2 |
| Require users to register when signing in | Yes |
| Days before re-confirming authentication info | 180 |
| Notify users on password resets | Yes |
| Notify all admins when other admins reset | Yes |

![SSPR enabled for All](images/fig31-sspr-all.png)
*Figure 31 – SSPR enabled for all users.*

![Number of methods required to reset: 2](images/fig32-sspr-methods.png)
*Figure 32 – Two methods required.*

![Registration required, re-confirm every 180 days](images/fig33-sspr-registration.png)
*Figure 33 – Registration enforced.*

![Notifications for users and admins enabled](images/fig34-sspr-notifications.png)
*Figure 34 – Reset notifications enabled.*

## Verification and results

The assessment script confirmed all 10 recommendations as **IMPLEMENTED** on 20 October 2026 at 12:16 UTC, with 0 not configured and 0 unknown.

![All five Conditional Access policies in state On](images/fig35-ca-final-state.png)
*Figure 35 – Final state: CA001–CA005 all On.*

![Assessment script output (object ID and tenant ID blurred)](images/fig36-script-output.png)
*Figure 36 – Assessment script summary: 10 implemented, 0 not configured, 0 unknown. The group object ID and the tenant ID in the snapshot path are blurred.*
