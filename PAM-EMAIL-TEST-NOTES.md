# PAM lab notes: existing access and approval email test

Date: 28 October 2026

## Current findings

Project: `smanley-test-pam-lab-01`.

- Setup administrator and approver: `smanley@trekai.org` (`pam-admin`).
- Requester: `smanley-adm@trekai.org` (`pam-requester`).
- The requester successfully read `lab-secret` before requesting a PAM secret-reader grant.
- The requester's grant search returned no grants for `lab-secret-reader`.
- Read-only inspection found no IAM bindings directly on the secret. The project gives the
  requester Browser and a temporary PAM Storage Object Viewer binding; neither explains
  secret-payload access.
- The project is directly under organization `470598084059`. Its organization policy grants
  Owner to `owners@trekai.org`. This is a possible inherited access path, but the requester's
  direct or nested membership in that group has **not been confirmed**.
- IAM Troubleshooter worked with numeric project number `354321550587`, but returned
  `UNKNOWN_INFO_DENIED`; it could not fully evaluate group membership and deny policies.
- A separate read-only test confirmed secret access still succeeds as the requester; the
  secret output was discarded. No IAM roles were changed during the investigation.

Existing access means this account cannot currently demonstrate the secret's denied →
allowed → denied behavior reliably. That investigation is paused. We can still test whether
a request enters PAM's approval workflow and triggers an approver email.

## Test approval notification only

Use `lab-secret-reader`, which requires approval from smanley. Do not use
`lab-bucket-reader`, which is self-service.

These are instructions to run manually. Creating a grant request is expected to trigger
an email; no request was submitted when this note was written.

### 1. Load the lab and verify the entitlement

In your Mac terminal:

```bash
source ~/Documents/Google-Cloud/PAM/google-cloud-pam/PAM-LAB/.env
gcloud config configurations activate pam-admin
gcloud pam entitlements describe lab-secret-reader \
  --project="$PROJECT_ID" --location=global \
  --format='yaml(state,eligibleUsers,approvalWorkflow)'
```

Confirm the entitlement is available, the requester is `smanley-adm@trekai.org`, and its manual
approval step lists `smanley@trekai.org` with one approval required.

### 2. Submit one request as smanley-adm

```bash
gcloud config configurations activate pam-requester
gcloud pam grants create \
  --entitlement=lab-secret-reader \
  --requested-duration=1800s \
  --justification="PAM lab: test approval email delivery to smanley" \
  --project="$PROJECT_ID" --location=global \
  --account=smanley-adm@trekai.org
```

Do not add admin service-account impersonation. Save the returned grant name or ID. Submit
only once; the same test can also be performed in the console as smanley-adm using
**My entitlements → lab-secret-reader → Request grant**, but do not use both methods.

### 3. Check the request and email

```bash
gcloud pam grants search \
  --entitlement=lab-secret-reader \
  --caller-relationship=had-created \
  --project="$PROJECT_ID" --location=global \
  --format='table(name.basename():label=GRANT_ID,state,createTime)'
```

Expected state: `APPROVAL_AWAITED`.

Check the **smanley@trekai.org** inbox, including spam and quarantine, for an approval request
from `pam-noreply@google.com`. Confirm the email refers to the correct project, requester,
entitlement and grant. Google identities configured as approvers are automatically included
as notification recipients; effective PAM notification settings can disable these emails.

In the console as smanley, select the lab project and check **Approve grants → Pending
approval**. Seeing the pending request proves the workflow is working; receiving its email
separately confirms delivery. Existing secret access does not prove either result.

### 4. If the email does not arrive

- Confirm the grant is `APPROVAL_AWAITED` and the entitlement names smanley as approver.
- Check **PAM → Settings → Notifications** for the effective, possibly inherited setting
  for **Grant requires approval → Approver**. Inspect it first; agree on any changes before
  modifying settings.
- Check spam/quarantine and mail filtering for `pam-noreply@google.com`. Allow some time for
  delivery; avoid submitting duplicate requests while checking.

No extra notification email address is normally needed for this Google-account approver.

### 5. Close the test request

Approval is not necessary to test the request email. Once delivery has been checked, smanley
can deny this test request in the console with the reason `Email notification test complete`.
Do not approve it unless you also want to activate its temporary IAM grant.

## Record the outcome

- Grant ID: pending
- Request reached `APPROVAL_AWAITED`: not yet checked
- Visible to smanley in Pending approval: not yet checked
- Email received at smanley: not yet checked
- Request closed: not yet checked

## References

- [PAM notification recipients and events](https://docs.cloud.google.com/iam/docs/pam-overview#notifications)
- [Configure PAM notification preferences](https://docs.cloud.google.com/iam/docs/pam-configure-settings)
- [Request temporary access](https://docs.cloud.google.com/iam/docs/pam-request-temporary-elevated-access)
- [Allow the PAM sender address](https://docs.cloud.google.com/iam/docs/pam-permissions-and-setup)
