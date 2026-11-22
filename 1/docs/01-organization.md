# Phase 1 — Organization: Draw the Map

> **Mission:** turn one standalone account into an organization with a structure that guardrails can attach to.

[⬅ Phase 0](00-prep.md) · [Lab overview](../README.md) · [Phase 2 ➡](02-identity-center.md)

---

## Why this matters

An **account** is AWS's strongest isolation boundary. Resources, IAM and quotas are all separate per account. An **organizational unit (OU)** is a folder of accounts, and policies attached to an OU apply to every account inside it. Design the OUs around **how you want to govern**, not around your org chart.

| OU | Holds | Governed by |
|---|---|---|
| **Security** | `security` account | Strictest access; holds the evidence |
| **Workloads** | Project accounts (empty for now) | Workload guardrails: no IAM users, required tags, encryption |
| **Suspended** | Accounts on their way out | `deny-all`, so nothing can run |

The management account sits at the root, outside every OU, on purpose.

## Concepts in 30 seconds

- **All features** mode turns on SCPs, tag policies and the other policy types. The alternative, consolidated billing only, is just a shared bill.
- **`OrganizationAccountAccessRole`** is created automatically in every account you *create* (not in accounts you *invite*). It trusts the management account and has admin rights, which makes it the break-glass path into member accounts.
- **Account email:** every account needs a unique email address. Plus-addressing (`you+acme-security@example.com`) delivers to your normal inbox but counts as a new address. Gmail and Outlook both support it.

---

## Steps

Sign in as your admin user (`bootstrap-admin` from Phase 0). Check that the account ID in the top-right corner is your management account.

1. **Create the organization:** search **AWS Organizations** → click **Create an organization**. All features are on by default, and you want them on. Nothing exists until you click this. AWS sends a verification email, but you don't need it to create accounts.
2. **Create 3 OUs:** *AWS accounts* → tick **Root** → *Actions* → *Organizational unit* → **Create new**. Repeat for:
   - `Security`
   - `Workloads`
   - `Suspended`
3. **Create the security account:** *Add an AWS account* → **Create an AWS account**:
   - Account name: `security`
   - Email: `you+acme-security@example.com` (use your own address with a `+alias`)
   - IAM role name: leave the default `OrganizationAccountAccessRole`
   
   It shows as *Active* after a few minutes.
4. **Move it:** tick `security` → *Actions* → **Move** → choose the `Security` OU.

The management account stays at Root, outside every OU. That's on purpose.

**Write down the security account ID.** Almost every later phase uses it as `<SECURITY_ID>`.

## Verify

```bash
aws organizations describe-organization --query 'Organization.{Id:Id,FeatureSet:FeatureSet}'
# want FeatureSet = ALL; the Id (o-…) is your <ORG_ID>

aws organizations list-accounts --query 'Accounts[].[Name,Id,State]' --output table
# want 2 ACTIVE accounts: management + security

ROOT=$(aws organizations list-roots --query 'Roots[0].Id' --output text)
aws organizations list-organizational-units-for-parent --parent-id $ROOT --query 'OrganizationalUnits[].[Name,Id]' --output table
# want Security, Workloads, Suspended

aws organizations list-parents --child-id <SECURITY_ID> --query 'Parents[0].Id' --output text
# want the Security OU's id (ou-…), not the root (r-…)
```

---

## 🔥 Field notes

> [!WARNING]
> **The account quota is 2, and your management account counts.** New organizations often start with an applied quota of **2**, so you get one member account. This lab first planned 6 accounts. The quota-increase case stayed *open* for days, so the design was cut to 2 accounts. If you need more, request it early: *Service Quotas → AWS Organizations → "Maximum number of accounts"* (`L-E619E033`, region `us-east-1`).

> [!NOTE]
> **New accounts are verified for up to 2 days.** Until then, some services refuse to start. CloudShell said *"Unable to create the environment. Your account verification is in progress."* Use local CLI SSO profiles in the meantime.

> [!TIP]
> **Renaming an account.** The console wouldn't rename the member account (it was created as `log-archive`). The CLI did, run as an admin *inside* that account:
> ```bash
> aws account put-account-name --account-name security --profile acme-security
> ```

---

## 🎤 Interview corner

<details><summary>Why use multiple accounts instead of one account with IAM boundaries?</summary>

Accounts are a hard boundary. A mistake or a compromise in one can't reach another without an explicit cross-account trust. Each account also has its own quotas and its own bill, and you can apply a different guardrail set per OU. IAM boundaries inside one account are soft by comparison: one overly broad policy and the walls are gone.
</details>

<details><summary>What's the difference between an OU and an account?</summary>

Accounts hold resources. OUs hold accounts and exist to attach policies (SCPs, tag policies and others) that every account inside inherits.
</details>

<details><summary>What is OrganizationAccountAccessRole and why is it sensitive?</summary>

It's an admin role created in every account you create through Organizations, and it trusts the management account. Anyone with admin in management can become admin in every member account. Treat management-account admin as the most privileged access in the org.
</details>

---

**Terraform:** [`terraform/organization/`](../terraform/organization) creates the org, the OUs (`ous.tf`) and the account (`accounts.tf`, with `close_on_deletion = false` and `prevent_destroy`).

[⬅ Phase 0](00-prep.md) · [Lab overview](../README.md) · [Phase 2 ➡](02-identity-center.md)
