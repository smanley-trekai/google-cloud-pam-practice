# PAM: create projects and virtual machines with temporary console access

Follow the same gcloud and impersonation workflow as [TRY-PAM.md](TRY-PAM.md),
then use the resulting access in the Google Cloud console. Commands use bash or
zsh. This guide creates an entitlement on **`trekai-rd` (folder `653186815477`)**
inside `trekai.org`. Its roles apply to that folder and its descendant projects.
The earlier lab's project-level setup does not configure this folder.

| Identity | What it does |
|---|---|
| `smanley@trekai.org` | Impersonates the existing admin service account for bootstrap, then manages PAM and approves grants directly |
| `terraform-org-admin@trekai-452018.iam.gserviceaccount.com` | Supplies permissions for bootstrap commands carrying the impersonation flag |
| `smanley-adm@trekai.org` | Requests a grant directly, then creates projects and VMs in its console session |

The grant gives roles directly to `smanley-adm@trekai.org`. Service-account
impersonation is for setup; it does not change the account signed in to the
console. PAM allows the two accounts above to request and approve separately.

## Before you start

- Have gcloud installed and both user accounts authenticated. Console login is
  separate from gcloud login; use separate browser profiles for the two users.
- `smanley@trekai.org` needs Service Account Token Creator on the existing admin
  service account. If you already completed the impersonation bootstrap in
  [TRY-PAM.md](TRY-PAM.md#lab-variables-and-discovery), reuse it.
- The admin service account needs IAM administration permissions on `trekai-rd`,
  permission to enable APIs in `trekai-452018`, and permission to grant billing
  access if you use the optional billing step below. Impersonation alone does
  not supply those permissions.
- This entitlement grants Compute administration across projects under `trekai-rd`.
  Review that scope before running the setup commands.

In macOS zsh, allow pasted comments first; skip this in bash:

```zsh
setopt interactivecomments
```

## 1. Load the identifiers and verify impersonation

Load the verified environment saved during setup:

```bash
source ~/Documents/Google-Cloud/PAM/google-cloud-pam/PAM-CONSOLE/.env
APPROVER=smanley@trekai.org
REQUESTER=smanley-adm@trekai.org
ADMIN_SA=terraform-org-admin@trekai-452018.iam.gserviceaccount.com
ADMIN_PROJECT=trekai-452018
ENTITLEMENT=console-project-vm-admin
```

If you did not create that file, use the assignments below instead, checking
these IDs against your intended folder and billing account:

```bash
ORG_ID=470598084059
FOLDER_ID=653186815477
BILLING_ACCOUNT=018378-808B11-17982B
APPROVER=smanley@trekai.org
REQUESTER=smanley-adm@trekai.org
ADMIN_SA=terraform-org-admin@trekai-452018.iam.gserviceaccount.com
ADMIN_PROJECT=trekai-452018
ENTITLEMENT=console-project-vm-admin
```

Choose one loading method. Repeat it in each new terminal. Do not use the lab
`PROJECT_ID` as the organization ID; organization IDs are numeric.

```bash
gcloud auth list
gcloud auth print-access-token \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA" > /dev/null

gcloud resource-manager folders describe "$FOLDER_ID" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
gcloud billing accounts list \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

printf 'Organization: %s\nFolder: %s\nBilling: %s\nSetup/approver: %s\nRequester: %s\n' \
  "$ORG_ID" "$FOLDER_ID" "$BILLING_ACCOUNT" "$APPROVER" "$REQUESTER"
```

The token check should finish without an error. Confirm the IDs identify your
intended folder, organization, and billing account, with no placeholders left. If
impersonation fails, use the bootstrap instructions in `TRY-PAM.md` before
continuing. Tokens are discarded rather than printed.

## 2. Reuse the two gcloud configurations

Inspect the configurations created by `TRY-PAM.md`:

```bash
gcloud config configurations list
gcloud config configurations describe pam-admin
gcloud config configurations describe pam-requester
```

If either configuration is missing, create only that configuration:

```bash
# Run only if pam-admin is missing.
gcloud config configurations create pam-admin

# Run only if pam-requester is missing.
gcloud config configurations create pam-requester
```

Set each account and clear persistent impersonation:

```bash
gcloud config configurations activate pam-admin
gcloud config set account "$APPROVER"
gcloud config unset auth/impersonate_service_account

gcloud config configurations activate pam-requester
gcloud config set account "$REQUESTER"
gcloud config unset auth/impersonate_service_account

gcloud config configurations activate pam-admin
gcloud config list
```

If an account is missing from `gcloud auth list`, authenticate it with
`gcloud auth login EMAIL` using its exact address. Check for environment overrides
such as `CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT`, `CLOUDSDK_CORE_ACCOUNT`,
`CLOUDSDK_ACTIVE_CONFIG_NAME`, `CLOUDSDK_AUTH_ACCESS_TOKEN`, and
`CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE`; remove conflicting overrides in this
terminal. Application Default Credentials are not required.

All folder PAM commands below explicitly use `--folder="$FOLDER_ID"`;
any lab project still saved in the configurations is not their target.

## 3. Bootstrap folder PAM permissions with impersonation

**Who:** `smanley@trekai.org`, impersonating the existing admin service account.
Run these setup commands in order and resolve any error before continuing:

```bash
gcloud config configurations activate pam-admin

# Enable the PAM API in the existing administration project.
gcloud services enable privilegedaccessmanager.googleapis.com \
  --project="$ADMIN_PROJECT" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

# Let smanley administer PAM on trekai-rd, matching the lab pattern.
for ROLE in roles/privilegedaccessmanager.admin roles/resourcemanager.folderAdmin; do
  gcloud resource-manager folders add-iam-policy-binding "$FOLDER_ID" \
    --member="user:$APPROVER" --role="$ROLE" --condition=None \
    --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
done

# Let the requester browse the resource hierarchy in the console.
gcloud resource-manager folders add-iam-policy-binding "$FOLDER_ID" \
  --member="user:$REQUESTER" --role="roles/browser" --condition=None \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
```

These are persistent setup and visibility bindings. The requester receives its
project-creation and Compute administration roles only through the entitlement
below. The setup administrator's PAM Admin and Folder IAM Admin roles let it
manage sensitive access; they are not requester roles.

**Billing prerequisite:** if `smanley-adm` does not already have access to the
approved billing account, an authorized billing administrator can run this
command using the same impersonation pattern, provided `ADMIN_SA` can change
that billing account's IAM policy:

```bash
gcloud billing accounts add-iam-policy-binding "$BILLING_ACCOUNT" \
  --member="user:$REQUESTER" --role="roles/billing.user" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
```

This is standing Billing Account User access, separate from the PAM grant. Skip
it if access already exists. If the service account lacks billing IAM
permissions, have the billing administrator arrange this binding.

## 4. Turn on PAM for the folder

**Who:** `smanley@trekai.org`, directly, as in Part 2 of `TRY-PAM.md`.
Allow time for the setup role bindings to propagate.

1. Open the [PAM console page](https://console.cloud.google.com/iam-admin/pam/entitlements/)
   in the browser profile signed in as `smanley@trekai.org`.
2. Select **`trekai-rd` (folder `653186815477`)**, rather than the organization or lab project.
3. If prompted, click **Set up PAM → Grant role → Complete setup**.
4. If already configured, continue with the check below.

The console initializes Google's organization-level PAM service agent and grants
its service-agent role at the selected scope. Do not create a replacement service
account. Its identity is
`service-org-ORG_ID@gcp-sa-pam.iam.gserviceaccount.com`.

```bash
gcloud config configurations activate pam-admin
gcloud pam check-onboarding-status \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"
```

Resolve any reported onboarding issue before creating the entitlement. See
[Google's setup guide](https://docs.cloud.google.com/iam/docs/pam-permissions-and-setup).

## 5. Create the entitlement from YAML

**Who:** `smanley@trekai.org`, directly. As in `TRY-PAM.md`, bootstrap used
impersonation; PAM administration now uses the setup administrator's own roles.

```bash
mkdir -p ~/Documents/Google-Cloud/PAM/google-cloud-pam/PAM-CONSOLE
cd ~/Documents/Google-Cloud/PAM/google-cloud-pam/PAM-CONSOLE
gcloud config configurations activate pam-admin

cat > console-project-vm-admin.yaml <<EOF
privilegedAccess:
  gcpIamAccess:
    resourceType: cloudresourcemanager.googleapis.com/Folder
    resource: //cloudresourcemanager.googleapis.com/folders/$FOLDER_ID
    roleBindings:
    - role: roles/resourcemanager.projectCreator
    - role: roles/compute.instanceAdmin.v1
    - role: roles/serviceusage.serviceUsageAdmin
maxRequestDuration: 3600s
eligibleUsers:
- principals:
  - user:$REQUESTER
approvalWorkflow:
  manualApprovals:
    requireApproverJustification: true
    steps:
    - approvalsNeeded: 1
      approvers:
      - principals:
        - user:$APPROVER
requesterJustificationConfig:
  unstructured: {}
EOF

gcloud pam entitlements create "$ENTITLEMENT" \
  --entitlement-file=console-project-vm-admin.yaml \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"
```

| Role | What the active grant permits |
|---|---|
| Project Creator | Create projects under trekai-rd |
| Compute Instance Admin (v1) | Create and manage VMs in projects under trekai-rd |
| Service Usage Admin | Enable the Compute Engine API in new projects |

The maximum grant is one hour, the requester must give a reason, and `smanley`
must approve it with a reason. Organization Administrator alone would not
provide all these permissions. See [Create entitlements](https://docs.cloud.google.com/iam/docs/pam-create-entitlements).

If the entitlement already exists, describe and review it rather than rerunning
creation or deleting it:

```bash
gcloud pam entitlements describe "$ENTITLEMENT" \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"
```

## 6. Request and approve a grant with gcloud

**Requester: `smanley-adm@trekai.org`, directly, without impersonation.**

```bash
gcloud config configurations activate pam-requester
gcloud config list

gcloud pam entitlements search --caller-access-type=grant-requester \
  --folder="$FOLDER_ID" --location=global --account="$REQUESTER"

gcloud pam grants create --entitlement="$ENTITLEMENT" \
  --requested-duration=3600s \
  --justification="INC-1234: create a project in trekai-rd and provision a VM" \
  --folder="$FOLDER_ID" --location=global --account="$REQUESTER"
```

Save the returned grant ID (the last segment of the grant name). The request
should wait for approval. Do not submit a second request in the console.

**Approver: `smanley@trekai.org`, directly, without impersonation.**

```bash
gcloud config configurations activate pam-admin

gcloud pam grants search --entitlement="$ENTITLEMENT" \
  --caller-relationship=can-approve \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER" \
  --format="table(name.basename():label=GRANT_ID,requester,state)"

GRANT_ID=REPLACE_WITH_RETURNED_GRANT_ID

gcloud pam grants describe "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"

# After reviewing the requester, reason, and duration:
gcloud pam grants approve "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --reason="Approved for INC-1234; one hour to create the project and VM" \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"
```

Replace the grant ID placeholder before running describe or approve. The
approver is the user listed in YAML, so approving as the impersonated admin
service account would use the wrong identity. See [Approve grants](https://docs.cloud.google.com/iam/docs/pam-approve-deny-grants).

## 7. Verify access as the requester

```bash
gcloud config configurations activate pam-requester
gcloud pam grants search --entitlement="$ENTITLEMENT" \
  --caller-relationship=had-created \
  --folder="$FOLDER_ID" --location=global --account="$REQUESTER" \
  --format="table(name.basename():label=GRANT_ID,state)"
```

Wait for `ACTIVE`, then allow a few minutes for IAM propagation. Sign in to the
console as **`smanley-adm@trekai.org`** and continue below. A successful gcloud
request does not switch browser accounts; confirm the account avatar yourself.

## 8. Create a project in trekai-rd

With the grant **Active**, use **`smanley-adm@trekai.org`**:

1. Open **IAM & Admin → Manage resources** and click **Create project**.
2. Enter the project name and review the project ID.
3. Select the `trekai.org` organization and **`trekai-rd` (folder `653186815477`)**
   as the parent location. The folder grant does not permit creating projects
   directly under the organization or in sibling folders.
4. Select an approved billing account. `smanley-adm@trekai.org` needs **Billing Account
   User** (`roles/billing.user`) on that billing account to link it while
   creating the project. Arrange this with the billing administrator; the
   folder entitlement does not automatically provide billing-account access.
5. Click **Create** and wait for the project to finish provisioning.
6. Select the new project in the console's resource selector.

Project Creator on `trekai-rd` supplies the project-creation
permission. It does not itself grant VM administration. The additional Compute
role in this entitlement is inherited by projects under its scope. See Google's
[project creation guide](https://docs.cloud.google.com/resource-manager/docs/creating-managing-projects).

If billing is linked after project creation, the linking user also needs the
project-side billing permissions, typically **Project Billing Manager**
(`roles/billing.projectManager`), along with the billing-account permission.
See [billing-link permissions](https://docs.cloud.google.com/billing/docs/how-to/modify-project).

**Check for persistent access:** creating a project can automatically grant its
creator the Owner role on that project. Inspect **IAM & Admin → IAM** after
creation. Any such binding is independent of PAM and can remain after the grant
expires. Have an authorized administrator remove unneeded standing access once
an approved administration path is established.

## 9. Create a virtual machine in the new project

1. Sign in as **`smanley-adm@trekai.org`**, confirm its PAM grant is still
   **Active**, and select the new project.
2. Confirm billing is enabled for the project.
3. Open **Compute Engine → VM instances**. If prompted, enable the Compute Engine
   API. The Service Usage Admin role in this example permits API enablement.
4. Click **Create instance** and choose the name, region, zone, machine type,
   boot disk, and network settings.
5. In the security settings, review the attached service account. Attaching one
   requires `iam.serviceAccounts.actAs`, normally granted through **Service
   Account User** (`roles/iam.serviceAccountUser`) on the intended service
   account. Have an administrator arrange that permission if needed. Prefer
   access to the specific account over adding this role across the organization.
   If the VM needs no Google Cloud API identity, select no service account.
6. Review the configuration and click **Create**.
7. Confirm the VM appears in the instance list and reaches the running state.

Compute Instance Admin (v1) supports VM creation. Shared VPC subnets, shared
images, and other cross-project resources can require additional permissions on
those resources. See Google's
[VM creation guide](https://docs.cloud.google.com/compute/docs/instances/create-start-instance).

The VM continues running and accruing applicable charges after the PAM grant
expires. Expiry removes the grant's permissions; it does not delete the project
or VM.

## 10. Check expiry or revoke access early

When the hour ends, use the requester search command in step 7 to check that the
grant has ended. Test an operation requiring the granted roles again. Other
standing or inherited access can still authorize it, including a project Owner
binding assigned during creation.

To revoke early as the PAM administrator:

```bash
gcloud config configurations activate pam-admin
gcloud pam grants revoke "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --reason="Project and VM provisioning finished" \
  --folder="$FOLDER_ID" --location=global --account="$APPROVER"
```

Expiry or revocation does not remove the setup administrator's roles, Browser,
Billing Account User, or any other separately assigned roles. It also does not
delete the project or stop the VM.

## Troubleshooting

| Problem | What to check |
|---|---|
| `iam.serviceAccounts.getAccessToken` denied | Verify `smanley` can impersonate `ADMIN_SA`; follow the bootstrap in `TRY-PAM.md`. |
| Impersonated setup is denied | The service account itself needs permission for that operation. |
| Direct entitlement creation is denied | Verify smanley's folder-level PAM Admin and Folder IAM Admin bindings and allow for propagation. |
| The requester cannot see the entitlement | Check `FOLDER_ID`, requester email in YAML, active account, and propagation. |
| Approval is denied | Use `smanley@trekai.org` directly, with no impersonation or credential environment override. |
| Billing account is missing in the console | Check the requester's Billing Account User access to the intended account. |
| Active grant but VM creation is denied | Confirm the console account, project scope, IAM propagation, and permissions on any attached service account or shared resources. |
| Access still works after expiry | Inspect standing roles, inherited roles, other active grants, and the new project's Owner binding. |

## Folder setup added on 30 October 2026

The folder-level `console-project-vm-admin` entitlement is configured in
`trekai-rd`. The original organization-level entitlement is retained at the
user's request, along with its setup and Browser bindings. Its earlier successful
grant ended; the entitlement remains available for future organization-wide
requests. Use `--folder="$FOLDER_ID"` for the folder entitlement described here.
The organization-level PAM service-agent binding and separate billing access
also remain.

## Keep the entitlement in Terraform

For ongoing use, add the equivalent entitlement definition to Terraform and
import the existing entitlement as described in
[Adding an entitlement](README.md#adding-an-entitlement). Review the plan before
applying. Requests, approvals, and revocations remain runtime operations.
