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
| [osquery-org](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=032511868142)      | 032511868142 | infra+aws@osquery.io          | Top Level & Billing    |
| [osquery-identity](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=834249036484) | 834249036484 | infra+aws-identity@osquery.io | IAM: humans and groups |
| [osquery-logs](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=072219116274)     | 072219116274 | infra+aws-logs@osquery.io     | Cloudwatch Logs        |
| [osquery-infra](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=107349553668)    | 107349553668 | infra+aws-infra@osquery.io    | Semi-static infra      |
| [osquery-storage](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=680817131363)  | 680817131363 | infra+aws-storage@osquery.io  | Packages, artifacts    |
| [osquery-dev](https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=204725418487)      | 204725418487 | infra+aws-dev@osquery.io      | Dev and test hosts. Initial CI work |

There is a default role for cross sharing: `OrganizationAccountAccessRole` but this does not apply to our set up.
This default assumes identity accounts are created in the `osquery-org`, this trust is setup between the child accounts
and this parent. In our setup trust must be created between `osquery-identity` and the other child accounts.

For each child account we should create a `IdentityAccountAccessRole` role that mimics the "Organization" role.

### AWS Account Setup Process

AWS account setup is a somewhat cumbersome manual process. Notes about it.

Useful URLs:

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html

#### Initial AWS account

The first thing we did was create the `osquery-org` account. This is
the toplevel account. It was created using the normal AWS signup flow,
then converted to being an org account.

#### Subsequent Child Accounts

Sometimes we need to create additional AWS child accounts. There are a
couple of steps to that.

1. Login to AWS
2. Find your way to the organization screen
   * https://console.aws.amazon.com/organizations/home?region=us-east-2#/accounts
3. Click "Add account"
   * Name and email should conform to the convention in the table above
   * You can leave IAM role name to the default `OrganizationAccountAccessRole`
4. **IMPORTANT**: Set a root password and MFA (see below)

**IMPORTANT**: When an AWS account is created this way, it does _not_
have a root password of MFA set. This means the account is vulnerable
to a class of takeover attacks. The recommend approach is to use the
"forgot password" flow to set a root password and MFA device. We use a
virtual MFA device in the same 1password entry.

### User Account Setup Process

If you are a TSC member you will have access to the `osquery-identity` root account.
You can log in to the web console and use IAM to create a `$USERNAME-identity` account (or call it whatever).

Then to manage resources on other accounts you can assume an Administrator role.

To login, you need to use one of the magic switchrole links. For
example:
https://signin.aws.amazon.com/switchrole?roleName=IdentityAccountAccessRole&account=107349553668
(See the account table for others)
