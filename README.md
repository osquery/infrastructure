# Infrastructure

This is a management repo for the osquery project's infrastructure.

## Philosophy and Goals

1. IaaS
   a. Reduce human errors
   b. Code Review
   c. PR History
2. Secure by default & Least Access

## Amazon AWS

These are on seph's personal credit card. But we expect to be inside the free limit

### Credential management AWS Vault

Don't store them in `.aws/credentials` instead, use https://github.com/99designs/aws-vault with 2fa enabled, please see their documentation on how to setup 2fa using the `aws_profile`

### AWS Accounts

https://console.aws.amazon.com/organizations/home

| Name             | Account ID   | Email                         | Purpose                |
| ---------------- | ------------ | ----------------------------- | ---------------------- |
| osquery-org      | 032511868142 | infra+aws@osquery.io          | Top Level & Billing    |
| osquery-identity | 834249036484 | infra+aws-identity@osquery.io | IAM: humans and groups |
| osquery-logs     | 072219116274 | infra+aws-logs@osquery.io     | Cloudwatch Logs        |

Roll for cross sharing: `OrganizationAccountAccessRole` (the default)

### AWS Account Setup Process

Useful URLs:

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html

The initial boostrapping is manual. It was configured via:

1. Create the AWS org level account.
   a. Normal AWS signup flow
   b. Convert it to an org level
2. Create new child accounts
3. **IMPORTANT**: For all child accounts, you must set a proper root password and MFA
   a. This requires you go through a forgot password flow.
4. Initial user
   a. An initial `seph-initial` user was created
   b. And granted administrator access
