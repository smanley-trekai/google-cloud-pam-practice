# Request temporary SSH access to a VM with PAM

Use this guide to let `contractor-ait-aravinds@trekai.org` request SSH access to
`instance-20261030-190831`. `smanley@trekai.org` approves the request. The employee
then connects using their own Google account, through IAP and OS Login.

These are documented commands, not a record of completed deployment. This guide
does not create a VM or reuse the broader project/VM administrator entitlement.
It follows the setup impersonation pattern in [TRY-PAM.md](../TRY-PAM.md).

| Setting | Value |
|---|---|
| Project | `qa-virtual-machine` |
| VM | `instance-20261030-190831` |
| Zone | `us-central1-a` |
| Internal IP, checked 2 November 2026 | `10.128.0.2` |
| Requester | `contractor-ait-aravinds@trekai.org` |
| Setup administrator and approver | `smanley@trekai.org` |
| Setup service account | `terraform-org-admin@trekai-452018.iam.gserviceaccount.com` |
| Entitlement | `qa-vm-ssh-aravinds` |
| Maximum grant | 24 hours; requester can ask for 4, 8, or 24 hours |

## 1. Understand what expires

The entitlement temporarily grants:

| Role | Scope of access |
|---|---|
| OS Login (`roles/compute.osLogin`) | Standard Linux login to this VM, restricted by its resource name; no sudo role |
| IAP-secured Tunnel User (`roles/iap.tunnelResourceAccessor`) | Tunnel to `10.128.0.2` on TCP port 22 |
| Compute Viewer (`roles/compute.viewer`) | Read Compute resources in this project, needed for discovery and SSH tooling |

The project is the PAM entitlement's parent. IAM conditions narrow login and
tunnel access to the intended destination; Compute read access covers the
project. The IP-based condition must be updated if the VM's internal IP changes.
That condition can also match another destination if the IP is later reassigned.

The VM's attached service account requires a separate **Service Account User**
binding on that specific account. This guide makes that a standing prerequisite,
not a PAM-granted role. It remains after SSH access expires. Review the runtime
account's permissions because OS Login users can access its credentials from the
VM. Do not substitute the privileged setup service account as the runtime account.

**Expiry or revocation blocks new authorized logins after IAM propagation. It
does not terminate an established SSH session or undo changes made inside the
VM.** Existing standing OS Login, sudo, alternative SSH credentials, or other
grants can bypass the intended time limit. Review them before relying on PAM.

## 2. Load variables in the setup administrator's terminal

Use bash or zsh. In macOS zsh, run `setopt interactivecomments` before pasting
blocks that contain comments.

```bash
cd ~/Documents/Google-Cloud/PAM/google-cloud-pam/PAM-SSH
cp -n .env.example .env
source .env
printf 'Project: %s\nVM: %s\nZone: %s\nRequester: %s\nApprover: %s\n' \
  "$PROJECT_ID" "$VM_NAME" "$ZONE" "$REQUESTER" "$APPROVER"
```

Inspect an existing `.env` before sourcing it; `cp -n` preserves it. This folder
has its own environment file. Do not source `PAM-CONSOLE/.env` for this workflow.
Repeat `source .env` in each new setup terminal.

Verify authentication and impersonation:

```bash
gcloud auth list
gcloud auth print-access-token \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA" > /dev/null
```

If smanley's login expired, run `gcloud auth login smanley@trekai.org` yourself.
If impersonation is denied, follow the bootstrap in `TRY-PAM.md`. The setup
service account must itself have permissions to enable APIs, manage project
IAM, and manage the runtime service account's IAM policy.

Inspect `gcloud config list`. Clear persistent impersonation in a dedicated
configuration before direct user commands, and remove conflicting environment
overrides such as `CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT`,
`CLOUDSDK_AUTH_ACCESS_TOKEN`, and `CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE`.
Command-specific `--account` flags do not cancel persistent impersonation.

## 3. Verify the VM and existing access

Run each command individually and resolve errors before continuing:

```bash
gcloud compute instances describe "$VM_NAME" \
  --project="$PROJECT_ID" --zone="$ZONE" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA" \
  --format='yaml(name,status,networkInterfaces[].networkIP,serviceAccounts[].email)'

gcloud compute instances describe "$VM_NAME" \
  --project="$PROJECT_ID" --zone="$ZONE" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA" \
  --flatten='metadata.items[]' \
  --filter='metadata.items.key:(enable-oslogin block-project-ssh-keys)' \
  --format='table(metadata.items.key,metadata.items.value)'

gcloud compute firewall-rules list --project="$PROJECT_ID" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA" \
  --filter='direction=INGRESS' \
  --format='table(name,disabled,sourceRanges,allowed,targetTags)'

gcloud compute instances get-iam-policy "$VM_NAME" \
  --project="$PROJECT_ID" --zone="$ZONE" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

gcloud projects get-iam-policy "$PROJECT_ID" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
```

The earlier check found `enable-oslogin=TRUE`, the runtime service account shown
in `.env.example`, and the `qa-workstation` tag. Firewall rule
`default-allow-ssh` allowed `35.235.240.0/20` to TCP 22 for that tag. Recheck these
values; no new firewall rule is needed if they still match.

If OS Login is disabled, arrange an access migration before enabling it: enabling
OS Login changes SSH authentication and stops accepting metadata SSH keys.
Also review inherited and group-based permissions, IAP permissions, and access
inside the guest OS. Do not remove another employee's access while doing this
setup. The read-only policy commands above do not expand group membership.

## 4. Bootstrap project PAM and the runtime-account prerequisite

**Who: smanley, impersonating the existing setup service account.**

```bash
gcloud services enable privilegedaccessmanager.googleapis.com iap.googleapis.com \
  --project="$PROJECT_ID" \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="user:$APPROVER" --role=roles/privilegedaccessmanager.admin \
  --condition=None \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="user:$APPROVER" --role=roles/resourcemanager.projectIamAdmin \
  --condition=None \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"

gcloud iam service-accounts add-iam-policy-binding "$VM_SERVICE_ACCOUNT" \
  --project="$PROJECT_ID" \
  --member="user:$REQUESTER" --role=roles/iam.serviceAccountUser \
  --condition=None \
  --account="$APPROVER" --impersonate-service-account="$ADMIN_SA"
```

Skip bindings already supplied by an approved existing policy. The setup roles
belong to smanley; do not grant them to the employee.

**Who: smanley directly.** In the console, open
[Privileged Access Manager](https://console.cloud.google.com/iam-admin/pam/entitlements/),
select **`qa-virtual-machine`**, and complete **Set up PAM → Grant role → Complete
setup** if prompted. An inherited service-agent binding might already satisfy
onboarding. Do not create a project-level PAM service account.

```bash
gcloud pam check-onboarding-status \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"
```

Resolve onboarding errors before creating the entitlement.

## 5. Generate and create the SSH entitlement

**Who: smanley directly, without impersonation.** The supplied
[vm-ssh.yaml](vm-ssh.yaml) contains the known values. Regenerate it after changing
the environment so the requester, destination, and duration stay consistent:

```bash
cat > vm-ssh.yaml <<EOF
privilegedAccess:
  gcpIamAccess:
    resourceType: cloudresourcemanager.googleapis.com/Project
    resource: //cloudresourcemanager.googleapis.com/projects/$PROJECT_ID
    roleBindings:
    - role: roles/compute.osLogin
      conditionExpression: resource.name == "projects/$PROJECT_ID/zones/$ZONE/instances/$VM_NAME"
    - role: roles/iap.tunnelResourceAccessor
      conditionExpression: destination.ip == "$VM_INTERNAL_IP" && destination.port == 22
    - role: roles/compute.viewer
maxRequestDuration: $MAX_REQUEST_DURATION
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
  --entitlement-file=vm-ssh.yaml \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"
```

If it already exists, inspect it with `gcloud pam entitlements describe` using
the same project, location, and account flags. Do not delete it to add a user.
Changing `.env` or YAML alone does not update an existing Cloud entitlement.

## 6. Employee requests 4, 8, or 24 hours

**Who: the employee, in their own terminal.** Do not share credentials or use the
setup administrator's service-account impersonation.

```bash
gcloud auth login contractor-ait-aravinds@trekai.org
gcloud config list
gcloud config unset auth/impersonate_service_account

gcloud pam entitlements search --caller-access-type=grant-requester \
  --project=qa-virtual-machine --location=global \
  --account=contractor-ait-aravinds@trekai.org

gcloud pam grants create --entitlement=qa-vm-ssh-aravinds \
  --requested-duration=14400s \
  --justification="QA work requiring SSH to instance-20261030-190831" \
  --project=qa-virtual-machine --location=global \
  --account=contractor-ait-aravinds@trekai.org
```

Choose one duration per request:

| Duration | `--requested-duration` |
|---|---|
| 4 hours | `14400s` |
| 8 hours | `28800s` |
| 24 hours | `86400s` |

The 24-hour maximum also permits other supported shorter durations. If different
people should have hard caps of 4 or 8 hours, create separate entitlements with
those lower maximums instead. Save the returned grant ID and send it to the
approver through your normal process. Do not approve from the employee's account.

## 7. Smanley reviews and approves

**Who: smanley directly, in the setup administrator's terminal.**

```bash
gcloud pam grants search --entitlement="$ENTITLEMENT" \
  --caller-relationship=can-approve \
  --project="$PROJECT_ID" --location=global --account="$APPROVER" \
  --format='table(name.basename():label=GRANT_ID,requester,state,requestedDuration)'

GRANT_ID=REPLACE_WITH_THE_RETURNED_GRANT_ID

gcloud pam grants describe "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"

gcloud pam grants approve "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --reason="Approved QA SSH work for the requested duration" \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"
```

Replace the ID placeholder and review the actual requester, reason, and duration
before approval. Grant duration starts at activation, not when YAML is created.

## 8. Employee verifies activation and connects

```bash
gcloud pam grants search --entitlement=qa-vm-ssh-aravinds \
  --caller-relationship=had-created \
  --project=qa-virtual-machine --location=global \
  --account=contractor-ait-aravinds@trekai.org \
  --format='table(name.basename():label=GRANT_ID,state)'

gcloud compute ssh instance-20261030-190831 \
  --project=qa-virtual-machine --zone=us-central1-a \
  --tunnel-through-iap \
  --account=contractor-ait-aravinds@trekai.org
```

Wait for `ACTIVE` and allow time for IAM propagation before SSH. The same user
can use the console's SSH button after activation; confirm the console is signed
in as the employee. Test a fresh connection before the grant and after it ends:
new login should fail outside the active grant if no other access authorizes it.
The normal grant does not provide sudo. Use a separately reviewed OS Admin Login
entitlement if administrative Linux access is required.

## 9. Revoke early when needed

**Who: smanley directly.** Use the ID of the specific active grant:

```bash
gcloud pam grants revoke "$GRANT_ID" --entitlement="$ENTITLEMENT" \
  --reason="QA work completed; end SSH authorization" \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"
```

Do not delete the entitlement just to end one grant. Established SSH sessions
may remain connected; session termination is a separate VM administration task.

## 10. Add another employee

For a separate entitlement per employee, copy `.env.example` to another local
environment file, change `REQUESTER`, choose a unique `ENTITLEMENT`, and repeat
the runtime-account prerequisite and entitlement generation/creation steps.
Use a distinct YAML filename to keep each employee's definition available.
Adapt the employee-side commands to the new email and entitlement ID.

To share an entitlement, add the new `user:EMAIL` to `eligibleUsers.principals`
in the YAML, retaining existing entries. Apply it with:

```bash
gcloud pam entitlements update "$ENTITLEMENT" \
  --entitlement-file=vm-ssh.yaml \
  --project="$PROJECT_ID" --location=global --account="$APPROVER"
```

Each eligible employee still requests their own grant. A dedicated Google group
can replace individual requester entries when group membership is appropriately
managed. Adding a user to an entitlement does not create an active grant.

## Troubleshooting and references

| Problem | Check |
|---|---|
| PAM API disabled | Run the impersonated API-enablement step for `qa-virtual-machine`. |
| User cannot see the entitlement | Verify requester email, project, propagation, and absence of impersonation overrides. |
| `iam.serviceAccounts.actAs` denied | Check Service Account User on the VM's actual attached runtime account. |
| IAP tunnel denied | Check active grant, destination IP, port 22, firewall target tag, and network policy. |
| OS Login denied | Check VM metadata, resource-name condition, account identity, supported guest OS, and group/organization restrictions. |
| No sudo | Expected with OS Login; OS Admin Login is a different privilege. |
| SSH works after expiry | Test a new session; inspect standing, inherited, group, and guest-OS access. |

- [PAM permissions and onboarding](https://docs.cloud.google.com/iam/docs/pam-permissions-and-setup)
- [Create PAM entitlements and role conditions](https://docs.cloud.google.com/iam/docs/pam-create-entitlements)
- [Request grants](https://docs.cloud.google.com/iam/docs/pam-request-temporary-elevated-access)
- [Approve grants](https://docs.cloud.google.com/iam/docs/pam-approve-deny-grants)
- [OS Login requirements](https://docs.cloud.google.com/compute/docs/oslogin/set-up-oslogin)
- [IAP SSH and destination conditions](https://docs.cloud.google.com/iap/docs/using-tcp-forwarding)
- [IAM resource-name formats](https://docs.cloud.google.com/iam/docs/conditions-resource-attributes)
