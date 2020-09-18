# Infrastructure

This is a management repo for the osquery project's infrastructure.

## Philosophy and Goals

1. IaaS

- Reduce human errors
- Code Review
- PR History

2. Secure by default & Least Access

## Amazon AWS

These are on seph's personal credit card. But we expect to be inside the free limit.

### Credential management AWS Vault

Don't store them in `.aws/credentials` instead, use https://github.com/99designs/aws-vault with 2fa enabled, please see their documentation on how to setup 2fa using the `aws_profile`.

### AWS Accounts

https://console.aws.amazon.com/organizations/home

| Name             | Account ID   | Email                         | Purpose                |
| ---------------- | ------------ | ----------------------------- | ---------------------- |
| osquery-org      | 032511868142 | infra+aws@osquery.io          | Top Level & Billing    |
| osquery-identity | 834249036484 | infra+aws-identity@osquery.io | IAM: humans and groups |
| osquery-logs     | 072219116274 | infra+aws-logs@osquery.io     | Cloudwatch Logs        |
| osquery-infra    | 107349553668 | infra+aws-infra@osquery.io    | Semi-static infra      |
| osquery-storage  | 680817131363 | infra+aws-storage@osquery.io  | Packages, artifacts    |

There is a default role for cross sharing: `OrganizationAccountAccessRole` but this does not apply to our set up.
This default assumes identity accounts are created in the `osquery-org`, this trust is setup between the child accounts
and this parent. In our setup trust must be created between `osquery-identity` and the other child accounts.

For each child account we should create a `IdentityAccountAccessRole` role that mimics the "Organization" role.

### AWS Account Setup Process

Useful URLs:

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html

The initial boostrapping is manual. It was configured via:

1. Create the AWS org level account.

- Normal AWS signup flow
- Convert it to an org level

2. Create new child accounts
3. **IMPORTANT**: For all child accounts, you must set a proper root password and MFA

- This requires you go through a forgot password flow.

4. Initial user

- An initial `seph-initial` user was created
- And granted administrator access

### User Account Setup Process

If you are a TSC member you will have access to the `osquery-identity` root account.
You can log in to the web console and use IAM to create a `$USERNAME-identity` account (or call it whatever).

Then to manage resources on other accounts you can assume an Administrator role.

Use the following link but set the appropriate Account ID: https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=107349553668.
