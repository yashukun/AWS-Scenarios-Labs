# Phase 0 — Prep: Lock the Front Door

> **Mission:** make the account that will become your management account safe *before* it controls anything else.

[⬅ Lab overview](../README.md) · [Phase 1 ➡](01-organization.md)

---

## Why this matters

The management account will be able to create accounts, attach guardrails, and sign people into every account. If someone takes over its **root user**, they control the whole organization, and **SCPs never apply to the management account**. So we secure it first.

## What you'll set up

| Item | Why |
|---|---|
| Root MFA | The root user can do anything, including closing the account |
| No root access keys | Keys can leak; a root key leaking is game over |
| Budget alerts (C14) | A forgotten resource shouldn't surprise you at month end |
| Alternate contacts | AWS can reach security and billing contacts during an incident |
| AWS CLI | Every phase ends with CLI checks |
| Paid plan check | The newer AWS **Free plan** may block Organizations |

---

## Steps

> **Who does what:** root signs in **once**, for the few root-only tasks in Part A. Everything else, CLI included, runs as a normal IAM admin user. That user is temporary: you delete it at the end of Phase 2, once SSO works.

### Part A: As root (then sign out)
1. **Root MFA:** IAM → *Security credentials* → **Assign MFA device** (a passkey or an authenticator app).
2. **No root access keys:** on the same page, delete any access keys you see.
3. **Let IAM users see billing:** Account → *IAM user and role access to Billing information* → **Activate**. Without this, your admin user gets "access denied" on Budgets.
4. **Create a temporary admin user:** IAM → Users → *Create user*:
   - Name: `bootstrap-admin`
   - ☑ *Provide user access to the Management Console*, and set a password
   - Attach the **`AdministratorAccess`** policy
   - Then open the user → *Security credentials* → **Assign MFA device**
5. Sign out of root. From here on, sign in as `bootstrap-admin` (the sign-in URL is on the IAM dashboard).

### Part B: As `bootstrap-admin`
1. **Check the account plan:** Billing → *Plan*. It must be **Paid**. Upgrading keeps your free credits.
2. **Budget:** Billing → Budgets → *Create budget* → *Customize* → *Cost budget*:
   - Monthly, **$10**
   - Alerts at 50%, 80% and 100% of actual cost, plus 100% of **forecasted** cost
   
   A budget in the management account covers **every** account in the org, through consolidated billing.
3. **Alternate contacts:** Account → *Alternate contacts* → **Security** and **Billing**. Your own email is fine for both.
4. **CLI login:** on **your own computer's terminal**, not CloudShell:
   ```bash
   aws login
   ```
   A browser opens; sign in as `bootstrap-admin`. The CLI gets short-lived credentials, and no access key is stored on disk. (Older CLIs: create an access key for `bootstrap-admin` and run `aws configure`.)

## Verify

Run these as `bootstrap-admin`. You don't need root to read root's status.

```bash
aws sts get-caller-identity
# Arn should end in :user/bootstrap-admin, NOT :root

aws iam get-account-summary --query 'SummaryMap.{RootMFA:AccountMFAEnabled,RootKeys:AccountAccessKeysPresent}'
# want RootMFA = 1, RootKeys = 0

aws freetier get-account-plan-state --query accountPlanType
# want "PAID"

aws organizations describe-organization
# want AWSOrganizationsNotInUseException: a clean account, ready for Phase 1

aws budgets describe-budgets --account-id <MGMT_ID> --query 'Budgets[].[BudgetName,BudgetLimit.Amount]'
```

---

## 🔥 Field notes

> [!NOTE]
> **Free plan vs paid plan.** AWS accounts created after mid-2025 can start on a *Free plan* with credits. Some services, including Organizations, may be unavailable until you upgrade. `aws freetier get-account-plan-state` shows the plan and the remaining credits.

> [!WARNING]
> **`InvalidClientTokenId` on the very first command.** An old access key was still in `~/.aws/credentials`, and AWS had already deleted or deactivated it. Fixing the config doesn't help, because the key itself is dead. Move the file out of the way (`mv ~/.aws/credentials ~/.aws/credentials.old`) and run `aws login`.

> [!TIP]
> **Copying commands from this guide.** Type only the command itself. Don't copy a leading `$` (that stands for the prompt, and gives `$: command not found`) or a `...` (that stands for "the rest of the command").

> [!TIP]
> **One free tier for the whole org.** With consolidated billing, the free-tier allowance is shared across all accounts. Creating more accounts does **not** give you more free usage.

---

## 🎤 Interview corner

<details><summary>Why shouldn't you run workloads in the management account?</summary>

SCPs never apply to it, so no guardrail can protect it. It already holds the most powerful permissions in the org (creating accounts, attaching policies, SSO for everyone). Keep it small: Organizations, Identity Center, billing, and the org trail.
</details>

<details><summary>What's the first thing you do with a brand-new AWS account?</summary>

Turn on MFA for root, make sure root has no access keys, set alternate contacts, create a budget alarm, then stop using root. All daily work goes through federated, short-lived roles.
</details>

---

**Terraform:** the budget is in [`terraform/organization/budget.tf`](../terraform/organization/budget.tf). Root MFA can't be automated; that's by design.

[⬅ Lab overview](../README.md) · [Phase 1 ➡](01-organization.md)
