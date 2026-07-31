# Lab 1: Cloud Account Security, Identity and Access Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 1  
**Topic:** Identity governance, least privilege and LocalStack IAM  
**Environment:** LocalStack on `localhost:4566`  
**Name:** Wan

## Lab Summary // Objective

This Session A lab demonstrated cloud account security using LocalStack IAM. It covered the creation of an administrator group and personal administrator account, the assignment of a scoped read-only policy to an analyst account, and access-key hygiene through key creation, listing, and deactivation.

## Evidence Folder

All screenshots used for this report are stored alongside this Markdown file.

| Evidence File | Purpose |
|---|---|
| `2.1createGroup.png` | Creation of the `Admins` IAM group |
| `2.2createUserAdmin.png` | Creation of the `CloudAdmin_Wan` administrator user |
| `2.3.verifyMembership.png` | Verification that `CloudAdmin_Wan` belongs to `Admins` |
| `3.1createUserRead.png` | Creation of the `Analyst_Syed` analyst user |
| `3.2verifyPolicyUserRead.png` | Verification of `AmazonS3ReadOnlyAccess` for `Analyst_Syed` |
| `4.1createAccessKey-redacted.png` | Redacted access-key creation evidence for `Analyst_Syed` |
| `4.2listAccessKey.png` | Access-key metadata listing for `Analyst_Syed` |
| `4.3rotateDeactivateOld Key.png` | Access-key deactivation command |

## Task 1: Map the Cloud Identity Landscape

| Concept | AWS Term | Purpose |
|---|---|---|
| All-powerful owner | Root user | The original account owner with full control over resources and billing. It should be protected and not used for daily administration. |
| Human/app identity | IAM User | A named identity for a person, application, or service that needs credentials to access cloud resources. |
| Permission bundle | IAM Policy | A JSON permission document that defines which actions are allowed or denied on specified resources. |
| Collection of users | IAM Group | A mechanism for managing permissions for multiple users by attaching policies to the group. |
| Temporary identity | IAM Role | An identity that can be assumed temporarily to provide short-lived permissions without long-term user credentials. |

## Session A: LocalStack IAM

### Environment Setup

The AWS CLI was pointed to LocalStack using:

```bash
EP='--endpoint-url=http://localhost:4566'
```

This directs AWS CLI commands to the local LocalStack endpoint rather than a real AWS account.

Verification command:

```bash
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

Expected LocalStack identity:

```json
{
  "UserId": "000000000000",
  "Account": "000000000000",
  "Arn": "arn:aws:iam::000000000000:root"
}
```

The account ID `000000000000` identifies the LocalStack environment used for the exercises.

## Task 2: Create a Least-Privilege Admin

### Step 2.1: Create Admins Group

Command:

```bash
aws $EP iam create-group --group-name Admins
```

Result:

The `Admins` group was created successfully. The returned ARN was `arn:aws:iam::000000000000:group/Admins`.

Evidence:

![Admins group creation](lab1/Evidence/2.1createGroup.png)

### Step 2.2: Attach Administrator Policy to Group

Command:

```bash
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Verification command:

```bash
aws $EP iam list-attached-group-policies --group-name Admins
```

Expected verification output:

```json
{
  "AttachedPolicies": [
    {
      "PolicyName": "AdministratorAccess",
      "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
    }
  ]
}
```

This assigns administrator permissions to the group, allowing permissions to be managed centrally rather than attached directly to individual users.

### Step 2.3: Create Personal Admin User

Command:

```bash
aws $EP iam create-user --user-name CloudAdmin_Wan
```

Result:

The personal administrator user `CloudAdmin_Wan` was created successfully.

Evidence:

![CloudAdmin_Wan user creation](lab1/Evidence/2.2createUserAdmin.png)

### Step 2.4: Add User to Admins Group and Verify Membership

Command:

```bash
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_Wan
```

Verification command:

```bash
aws $EP iam get-group --group-name Admins
```

Result:

The group output lists `CloudAdmin_Wan` with ARN `arn:aws:iam::000000000000:user/CloudAdmin_Wan`. This confirms that the user is a member of `Admins` and inherits its administrator permissions through group membership.

Evidence:

![Verify Admins membership](lab1/Evidence/2.3.verifyMembership.png)

## Task 3: Enforce Least Privilege with a Scoped Policy

### Step 3.1: Create Read-Only Analyst User

Command:

```bash
aws $EP iam create-user --user-name Analyst_Syed
```

Result:

The user `Analyst_Syed` was created successfully.

Evidence:

![Analyst_Syed user creation](lab1/Evidence/3.1createUserRead.png)

### Step 3.2: Attach S3 Read-Only Policy

Command:

```bash
aws $EP iam attach-user-policy --user-name Analyst_Syed \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3.3: Verify Analyst Permissions

Verification command:

```bash
aws $EP iam list-attached-user-policies --user-name Analyst_Syed
```

Result:

The output lists only `AmazonS3ReadOnlyAccess` with policy ARN `arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess`. This confirms that the analyst account has a read-only S3 policy attached.

Evidence:

![Analyst read-only policy verification](lab1/Evidence/3.2verifyPolicyUserRead.png)

### Least Privilege Explanation

- `Analyst_Syed` is restricted to read-only Amazon S3 access rather than being granted administrator permissions.
- If this account were compromised, an attacker would be limited to the permissions in that policy and could not use this identity to administer IAM or modify cloud resources.
- Limiting permissions in this way reduces the blast radius of a compromised identity.

## Task 4: Credential Hygiene and Access Keys

### Step 4.1: Create Access Key

Command:

```bash
aws $EP iam create-access-key --user-name Analyst_Syed
```

Result:

An access key was created for `Analyst_Syed` with status `Active`.

Evidence:

![Redacted access key creation](lab1/Evidence/4.1createAccessKey-redacted.png)

Security note: The generated screenshot contains a secret access key. Secret access keys must be treated as credentials: they must not be committed to repositories, shared publicly, or stored in plaintext. The value is intentionally not reproduced in this report text.

### Step 4.2: List Access Keys

Command:

```bash
aws $EP iam list-access-keys --user-name Analyst_Syed
```

Result:

The access-key metadata shows one key for `Analyst_Syed` and confirms that its status was initially `Active`. Listing key metadata supports credential inventory and rotation activities without exposing the secret access-key value.

Evidence:

![Access key listing](lab1/Evidence/4.2listAccessKey.png)

### Step 4.3: Rotate and Deactivate Old Key

Command:

```bash
aws $EP iam update-access-key --user-name Analyst_Syed \
  --access-key-id <old-access-key-id> --status Inactive
```

Result:

The old access key was deactivated by setting its status to `Inactive`. This demonstrates the deactivation stage of access-key rotation and prevents the old credential from continuing to authenticate requests.

Evidence:

![Access key deactivation command](<lab1/Evidence/4.3rotateDeactivateOld Key.png>)

## Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching policies to groups centralises permission management. A policy can be attached or updated once for the group, and all members inherit the change. This makes access reviews easier and reduces errors that occur when individual user permissions are managed separately.

### Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a long-term identity for a person or application and can have long-lived credentials, such as access keys. An IAM Role is an assumable identity that provides temporary credentials. Roles are generally preferred for workloads because they reduce reliance on permanent credentials.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

`Analyst_Syed` demonstrates least privilege because it was assigned `AmazonS3ReadOnlyAccess` instead of administrative access. If compromised, the attacker would be constrained to the limited read-only permissions granted by that policy. They would not receive administrator capabilities through this account, which limits the potential impact or blast radius.

## Security Best-Practices Checklist

- [x] A dedicated administrator identity, `CloudAdmin_Wan`, was created instead of using the root identity for routine administration.
- [x] Administrator permissions are managed through the `Admins` group rather than being attached directly to the administrator user.
- [x] A scoped analyst identity, `Analyst_Syed`, was created and assigned `AmazonS3ReadOnlyAccess`.
- [x] Access keys were created, listed, and deactivated to demonstrate credential hygiene and rotation.
- [x] Secret access-key material is not repeated in the report text.

## Conclusion

Session A successfully demonstrated core IAM account-security practices in LocalStack. Administrative access was assigned through the `Admins` group and inherited by `CloudAdmin_Wan`, while `Analyst_Syed` was limited to Amazon S3 read-only access to demonstrate least privilege. The lab also demonstrated the access-key lifecycle by creating, listing, and deactivating an analyst access key. Together, these controls reduce unnecessary permissions and limit the impact of credential compromise.
