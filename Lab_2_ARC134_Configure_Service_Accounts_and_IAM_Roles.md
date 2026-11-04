# ARC134 — Configure Service Accounts and IAM Roles for Google Cloud

> **A practical, step-by-step companion to the Google Skills challenge lab**  
> Lab ID: `ARC134`  
> Course: _Configure Service Accounts and IAM Roles for Google Cloud_  
> Lab duration shown by Google: 45 minutes  
> Public walkthrough location for the current variant: `us-east4` region, `us-east4-a` zone. **Always use the region and zone displayed in your own active lab instructions if they differ.**

---

## How to use this guide

This guide follows the same five scored checkpoints as the official ARC134 challenge lab. It does not replace the official lab page: keep the official instructions open, follow their task order, and use this guide when a task is underspecified, when you need a command, or when something fails.

The official page describes five outcomes:

1. Create a service account using `gcloud`.
2. Grant IAM permissions to that service account.
3. Create a Compute Engine VM with the service account attached.
4. Create a project-level custom role from a YAML file.
5. Use the BigQuery Python client library from a VM that uses a service account.

Each checkpoint is worth 20 points. The optional Gemini introduction is not one of the five scored checkpoints. The official task labels are numbered Task 2 through Task 6 because Task 1 is the optional Gemini activity; this guide labels the scored sections **Checkpoint 1–5** and shows the official task number beside each one.

**Official lab:** [ARC134 — Google Skills](https://www.skills.google/focuses/67219?locale=en&parent=catalog)  
**Additional walkthrough:** [ePlus.dev ARC134 walkthrough](https://eplus.dev/configure-service-accounts-and-iam-roles-for-google-cloud-challenge-lab-arc134)  
**Command reference:** [Cloud-Wala-Banda ARC134 notes](https://github.com/Cloud-Wala-Banda/Google-Cloud-Lab-Solutions/blob/main/Configure%20Service%20Accounts%20and%20IAM%20for%20Google%20Cloud%20Challenge%20Lab.md)

> **Important:** Challenge labs can be updated. Resource names and required permissions below follow the publicly available ARC134 instructions and walkthroughs. Before running a command, compare it with the exact wording in your currently active lab. If Google changes a VM name, role, region, zone, or checkpoint, the live lab takes priority.

# Part A — Conceptual overview

## 1. What is this lab about?

ARC134 teaches you how to let applications and virtual machines access Google Cloud resources **without using a person's Google account for every API call**.

You create service accounts, grant them the permissions needed for a task, attach them to Compute Engine virtual machines, define a small custom IAM role, and use a service account to run a query against a public BigQuery dataset from Python.

## 2. Important concepts, explained simply

### Google Cloud project

A project is the container that holds cloud resources such as VMs, service accounts, APIs, and IAM policies. Every command in this lab must target the temporary project created for your lab session.

### IAM (Identity and Access Management)

IAM answers three questions:

- **Who?** A user, group, or service account.
- **What can they do?** A role containing permissions.
- **Where?** A project or a particular resource.

For example, ARC134 grants the `devops` service account roles that allow it to use service accounts and manage Compute Engine instances.

### Service account

A service account is an identity for software, a VM, or an application—not a person. A VM can use its attached service account when it calls Google Cloud APIs.

A service account email normally has this format:

```text
SERVICE_ACCOUNT_NAME@PROJECT_ID.iam.gserviceaccount.com
```

For this lab, the two service accounts are:

- `devops` — used by `vm-2` to access Compute Engine.
- `bigquery-qwiklab` — used by `bigquery-instance` to query BigQuery.

### IAM role and permission

A **permission** is one specific action, such as `compute.instances.list`. A **role** is a collection of permissions. You assign a role to a principal (for example, a service account) through an IAM policy binding.

ARC134 uses predefined roles such as:

- `roles/iam.serviceAccountUser`
- `roles/compute.instanceAdmin.v1`
- `roles/bigquery.dataViewer`
- `roles/bigquery.user`

The lab instructions may describe the Compute role in words as `compute.instanceAdmin`; the full predefined role ID used by `gcloud` is `roles/compute.instanceAdmin.v1`.

### Predefined role vs custom role

A predefined role is created and maintained by Google. A custom role is one you define by selecting the exact permissions needed. In ARC134, the custom role contains only:

- `cloudsql.instances.connect`
- `cloudsql.instances.get`

The YAML file is simply a structured text file describing the custom role.

### Attaching a service account to a VM

When a service account is attached to a VM, applications running on that VM can obtain credentials from the VM's metadata server and use the service account's permissions. The VM does not need a downloaded JSON key file for this lab.

### OAuth access scopes

A VM's access scopes can limit which Google APIs its attached service account can call. IAM roles and VM scopes work together: the service account needs the correct IAM permission, and the VM must have an access scope that allows the API call. A broad `cloud-platform` scope does not grant extra IAM permissions by itself; IAM still controls what the identity can do.

### BigQuery client library

The Python BigQuery client library lets Python code submit SQL queries to BigQuery. In this lab, the code uses Compute Engine credentials so that it authenticates as the `bigquery-qwiklab` service account attached to the VM.

# Part B — Practical walkthrough

## 0. Before starting the timer

1. Open the ARC134 lab page and read the task names and environment values before pressing **Start Lab**.
2. Use a private/incognito browser window and sign in with the temporary student credentials supplied by Google Skills.
3. Do not use your personal Google account for the lab's cloud resources.
4. Note the project ID and the region/zone shown in the lab. The public ARC134 walkthrough for the current variant specifies:
   - Region: `us-east4`
   - Zone: `us-east4-a`
5. Keep the official task page open beside Cloud Shell so you can click **Check my progress** after each checkpoint.
6. Do not press **End Lab** until you have finished checking all five scores. Ending the lab deletes its temporary environment.

### Important terminal distinction

There are two command-line environments in this lab:

- **Cloud Shell:** used to connect to the pre-created `lab-vm`.
- **The `lab-vm` SSH terminal:** the official tasks direct you to run the `gcloud` resource commands from this VM. Later, you SSH into `vm-2` and `bigquery-instance` for their specific tests.

A shell variable created in Cloud Shell does not automatically exist inside an SSH session. Set or export the variables again after connecting to `lab-vm`.

## 1. Connect to the pre-created `lab-vm`

### 1.1 Confirm the active project in Cloud Shell

Open Cloud Shell from the Google Cloud Console and run:

```bash
gcloud auth list
gcloud config get-value project
```

The active account should be the temporary lab account, and the project should be the temporary project shown by the lab page.

If Cloud Shell is not authenticated as the lab user, follow the lab's sign-in flow. Do not switch to a personal account.

### 1.2 Set the project and location variables in Cloud Shell

Replace the region and zone only if your live lab instructions show different values.

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-east4"
export ZONE="us-east4-a"

printf 'Project: %s\nRegion: %s\nZone: %s\n' "$PROJECT_ID" "$REGION" "$ZONE"
```

**Expected:** the project ID is a non-empty value, the region is `us-east4`, and the zone is `us-east4-a` for the currently documented lab variant.

If `PROJECT_ID` is blank or prints `(unset)`, stop and select the lab project before continuing:

```bash
gcloud config set project YOUR_LAB_PROJECT_ID
```

### 1.3 SSH into `lab-vm`

Run this from Cloud Shell:

```bash
gcloud compute ssh lab-vm --project="$PROJECT_ID" --zone="$ZONE"
```

Accept the SSH key prompts if they appear. Wait until the prompt changes to the remote VM shell.

### 1.4 Configure `gcloud` inside `lab-vm`

The official Task 2 asks you to authenticate `gcloud`, configure it for a user, and then switch to the `default` configuration. First inspect the existing configuration:

```bash
gcloud auth list
gcloud config configurations list
gcloud config list
```

If the remote VM needs interactive authentication, run:

```bash
gcloud auth login --no-launch-browser
```

Open the URL printed by the command, sign in with the same temporary lab student account, copy the verification code, and paste it back into the SSH terminal. Never share that code.

Then activate the default configuration and point it at the lab project:

```bash
gcloud config configurations activate default
gcloud config set project YOUR_LAB_PROJECT_ID
gcloud config set compute/region us-east4
gcloud config set compute/zone us-east4-a
```

Replace `YOUR_LAB_PROJECT_ID` with the exact project ID shown on the lab page. Verify:

```bash
gcloud config list
```

> **Configuration note:** the ARC134 task wording does not publish a fixed name for an additional user configuration. Do not blindly create a `user2` configuration from a different IAM lab (GSP647). If the live ARC134 hint explicitly asks for a named configuration, use the name and account shown in that live hint, then return to `default` as instructed.

### 1.5 Set reusable variables inside `lab-vm`

Run these after SSHing into `lab-vm`:

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="us-east4"
export ZONE="us-east4-a"

printf 'Project: %s\nRegion: %s\nZone: %s\n' "$PROJECT_ID" "$REGION" "$ZONE"
```

These variables are used throughout the remaining tasks in this SSH session.

### Optional official Task 1 — Enable and explore Gemini

This is an introductory activity and is not one of the five scored checkpoints. To mirror the official page, open the Gemini icon in the Google Cloud Console, enable Gemini Cloud Assist if prompted, click **Start Chatting**, and try these prompts:

```text
What is a service account?
What is the difference between predefined roles and custom roles?
```

You can skip this optional activity and continue directly to the scored service-account task.

---

## 2. Checkpoint 1 — Create the `devops` service account

**Official task:** Task 2 — _Create a service account using the gcloud CLI_  
**Target name:** `devops`

A service account named `devops` will represent the DevOps workload. The VM created later will use this identity.

### Step 2.1 — Create the service account

Inside the SSH session on `lab-vm`, run:

```bash
gcloud iam service-accounts create devops \
  --display-name="devops" \
  --project="$PROJECT_ID"
```

### Step 2.2 — Retrieve its email address

This is the command we discussed in chat when you asked how to find the ARC134 service account email:

```bash
gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)"
```

Store the result in a variable for later steps:

```bash
export SA="$(gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)")"

echo "$SA"
```

It should look similar to:

```text
devops@YOUR_LAB_PROJECT_ID.iam.gserviceaccount.com
```

Do not copy the example literally; use the email printed for your project.

### Step 2.3 — Verify the account exists

```bash
gcloud iam service-accounts describe "$SA" --project="$PROJECT_ID"
```

Return to the lab page and click **Check my progress** for **Create a service account using gcloud CLI**. Confirm the checkpoint awards 20/20 before proceeding.

### Common problems and fixes

| Problem             | Why it happens                                                                                                       | Fix                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `ALREADY_EXISTS`    | The service account was created in an earlier attempt in the same temporary project.                                 | Do not create a second account. Run the list/filter command, store its email in `SA`, and continue.                          |
| `PERMISSION_DENIED` | The command is running under the wrong Google account or in the wrong project.                                       | Run `gcloud auth list` and `gcloud config get-value project`; switch to the temporary student account and lab project.       |
| `SA` prints blank   | The variable assignment was run in a different SSH session, the filter didn't match, or the account was not created. | Re-run the list command without a filter, inspect the exact display name/email, then assign `SA` again in the current shell. |

**Useful listing command:**

```bash
gcloud iam service-accounts list --format="table(displayName,email)"
```

---

## 3. Checkpoint 2 — Grant IAM permissions to `devops`

**Official task:** Task 3 — _Grant IAM permissions to a service account using the gcloud CLI_  
**Principal:** the `devops` service account  
**Roles:** Service Account User and Compute Instance Admin

The service account needs permission to use service accounts and manage Compute Engine instances for the next task.

### Step 3.1 — Reconfirm project and service account variables

Run this in the same `lab-vm` SSH session:

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export SA="$(gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)")"

echo "PROJECT_ID=$PROJECT_ID"
echo "SA=$SA"
```

If either value is blank, fix that before assigning roles.

### Step 3.2 — Grant Service Account User

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA" \
  --role="roles/iam.serviceAccountUser"
```

### Step 3.3 — Grant Compute Instance Admin

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA" \
  --role="roles/compute.instanceAdmin.v1"
```

The full role ID `roles/compute.instanceAdmin.v1` is the predefined role identifier used by `gcloud`. It corresponds to the Compute Instance Admin role described in the task.

### Step 3.4 — Verify the bindings

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:$SA" \
  --format="table(bindings.role,bindings.members)"
```

Confirm that the output includes:

```text
roles/iam.serviceAccountUser
roles/compute.instanceAdmin.v1
```

Return to the lab page and click **Check my progress** for **Grant IAM permissions to service account using gcloud CLI**.

### Common problems and fixes

- **Wrong member type:** the member string must start with `serviceAccount:`. Do not use `user:` for this service-account binding.
- **Wrong role spelling:** use the full role ID `roles/compute.instanceAdmin.v1`, not the display name `Compute Instance Admin`.
- **Wrong project:** IAM project bindings apply to the project ID in the first argument. Check `gcloud config get-value project` before running the command.
- **Variable not set:** if `SA` is blank, re-run Step 3.1 in the current SSH terminal. Shell variables do not carry over automatically to a newly opened terminal.
- **Existing binding:** if the role is already present, verify it with the policy command rather than repeatedly adding the same binding.

---

## 4. Checkpoint 3 — Create `vm-2` with the `devops` service account attached

**Official task:** Task 4 — _Create a compute instance with a service account attached using gcloud_  
**VM name:** `vm-2`  
**Attached service account:** `devops`

### Step 4.1 — Create the VM from `lab-vm`

Make sure you are still connected to `lab-vm` and that `$SA` contains the `devops` service account email.

```bash
gcloud compute instances create vm-2 \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --service-account="$SA"
```

The public command reference for ARC134 uses the same essential arguments: VM name, service-account email, and zone. If your live lab hint specifies a machine type, image, or additional scope flag, include those exact values too.

### Step 4.2 — Verify the VM is running and has the correct identity

```bash
gcloud compute instances describe vm-2 \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --format="yaml(name,status,serviceAccounts)"
```

Confirm:

- `name` is `vm-2`.
- `status` is `RUNNING`.
- The `serviceAccounts` section contains the `devops` service account email.

### Step 4.3 — SSH into `vm-2`

From `lab-vm`, run:

```bash
gcloud compute ssh vm-2 --project="$PROJECT_ID" --zone="$ZONE"
```

Inside `vm-2`, verify the attached service account through the Compute Engine metadata server, then list instances:

```bash
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email

gcloud compute instances list --project="YOUR_LAB_PROJECT_ID"
```

Replace `YOUR_LAB_PROJECT_ID` with the project ID. The metadata endpoint should return the `devops` service account email, and the instance list should work without a permissions error.

The official task also asks you to try creating an instance from `vm-2`. Create a small temporary test VM, confirm it appears in the list, and delete that test VM afterward. Use the zone shown by your active lab:

```bash
gcloud compute instances create permission-test-vm \
  --project="YOUR_LAB_PROJECT_ID" \
  --zone="us-east4-a" \
  --machine-type="e2-micro"

gcloud compute instances list --project="YOUR_LAB_PROJECT_ID"
```

After verifying it, delete only the temporary test VM:

```bash
gcloud compute instances delete permission-test-vm \
  --project="YOUR_LAB_PROJECT_ID" \
  --zone="us-east4-a" --quiet
```

Do not delete `vm-2`. If the lab's live hint gives a different test VM name or zone, follow that instead.

Return to `lab-vm` (run `exit` in the `vm-2` SSH session), then click **Check my progress** for **Create a compute instance with service account attached using gcloud**.

### Common problems and fixes

| Problem                                                         | Likely cause                                                                                     | Fix                                                                                                                                     |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `vm-2` creation says the service account does not exist         | `$SA` is empty or has a typo.                                                                    | Print `echo "$SA"`, list service accounts, and use the exact email.                                                                     |
| `iam.serviceAccounts.actAs` or service-account permission error | The required IAM binding is missing or was granted to the wrong member.                          | Repeat Checkpoint 2 and verify the role binding is attached to `serviceAccount:$SA`.                                                    |
| VM is in the wrong zone                                         | The command used a default zone or an old value.                                                 | Use `--zone="$ZONE"` explicitly and verify the live lab's zone.                                                                         |
| SSH fails immediately after VM creation                         | The VM may still be booting or SSH key setup may not have finished.                              | Check `gcloud compute instances describe vm-2 ...`; wait until `RUNNING`, then retry SSH.                                               |
| The VM is already present                                       | A previous attempt created it.                                                                   | Describe it first. Reuse it if its attached service account is correct; do not delete it just to clear an error.                        |
| `gcloud compute instances list` returns 403                     | IAM role binding or VM access scope is insufficient, or the command targets a different project. | Verify the `devops` roles, confirm the project ID, and inspect the VM's access scopes. Follow the live lab hint before recreating a VM. |

---

## 5. Checkpoint 4 — Create a custom IAM role from YAML

**Official task:** Task 5 — _Create a custom role using a YAML file_  
**YAML filename:** `role-definition.yaml`  
**Custom role ID:** `customRole`  
**Required permissions:** `cloudsql.instances.connect`, `cloudsql.instances.get`

### Step 5.1 — Create the YAML file

Run this from `lab-vm`:

```bash
cat > role-definition.yaml <<'ROLE_YAML'
title: "Cloud SQL Instance Connector"
description: "Custom role with Cloud SQL connect and get permissions"
stage: "GA"
includedPermissions:
  - cloudsql.instances.connect
  - cloudsql.instances.get
ROLE_YAML
```

Check the file and its indentation:

```bash
cat role-definition.yaml
```

YAML indentation matters. Keep the two permissions nested under `includedPermissions` with consistent spacing.

### Step 5.2 — Create the project-level role

```bash
gcloud iam roles create customRole \
  --project="$PROJECT_ID" \
  --file="role-definition.yaml"
```

### Step 5.3 — Verify the custom role

```bash
gcloud iam roles describe customRole --project="$PROJECT_ID"
```

Confirm that the returned role includes both required permissions:

```text
cloudsql.instances.connect
cloudsql.instances.get
```

Return to the lab page and click **Check my progress** for **Create a custom role using a YAML file**.

### Common problems and fixes

- **YAML parsing error:** check indentation, colons, and hyphens. The permissions must be a YAML list below `includedPermissions`.
- **Invalid permission:** use the exact permission IDs shown in the challenge: `cloudsql.instances.connect` and `cloudsql.instances.get`. Do not substitute display names.
- **Role already exists:** inspect it first with `gcloud iam roles describe customRole --project="$PROJECT_ID"`. If it exists from an earlier attempt and the definition needs correction, update it with:

  ```bash
  gcloud iam roles update customRole \
    --project="$PROJECT_ID" \
    --file="role-definition.yaml"
  ```

- **Wrong scope:** the challenge requires a project-level custom role. Keep `--project="$PROJECT_ID"`; do not use `--organization`.

---

## 6. Checkpoint 5 — Query BigQuery from a VM using a service account

**Official task:** Task 6 — _Use the client libraries to access BigQuery from a service account_  
**Service account:** `bigquery-qwiklab`  
**VM name:** `bigquery-instance`  
**Required roles:** `roles/bigquery.dataViewer`, `roles/bigquery.user`

This final checkpoint creates another service account, attaches it to another VM, installs Python packages on that VM, and queries the public USA names dataset.

### Step 6.1 — Create the BigQuery service account

Run this on `lab-vm`:

```bash
gcloud iam service-accounts create bigquery-qwiklab \
  --display-name="bigquery-qwiklab" \
  --project="$PROJECT_ID"
```

Retrieve and store its email:

```bash
export BQ_SA="$(gcloud iam service-accounts list \
  --filter="displayName=bigquery-qwiklab" \
  --format="value(email)")"

echo "$BQ_SA"
```

Expected format:

```text
bigquery-qwiklab@YOUR_LAB_PROJECT_ID.iam.gserviceaccount.com
```

### Step 6.2 — Grant the two BigQuery roles

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$BQ_SA" \
  --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$BQ_SA" \
  --role="roles/bigquery.user"
```

Verify both roles:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:$BQ_SA" \
  --format="table(bindings.role,bindings.members)"
```

### Step 6.3 — Create `bigquery-instance` with the service account attached

```bash
gcloud compute instances create bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --service-account="$BQ_SA" \
  --scopes="https://www.googleapis.com/auth/bigquery"
```

The BigQuery OAuth scope allows the VM to request BigQuery API access. IAM roles still determine what the service account is authorized to do.

Verify the VM identity:

```bash
gcloud compute instances describe bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --format="yaml(name,status,serviceAccounts)"
```

### Step 6.4 — SSH into `bigquery-instance`

From `lab-vm`:

```bash
gcloud compute ssh bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE"
```

You should now be at a shell prompt on `bigquery-instance`.

### Step 6.5 — Install Python and the BigQuery dependencies

Use a Python virtual environment. This prevents Ubuntu/Debian system Python restrictions and ensures the packages are installed into the same Python environment that runs the query.

```bash
sudo apt-get update
sudo apt-get install -y python3 python3-venv python3-pip

python3 -m venv env
source env/bin/activate

python -m pip install --upgrade pip
python -m pip install google-cloud-bigquery pandas pyarrow db-dtypes
```

Verify that the environment is active and packages are available:

```bash
which python
python --version
python -m pip show google-cloud-bigquery pandas pyarrow db-dtypes
```

The path returned by `which python` should include `/env/bin/python`.

### Step 6.6 — Create `query.py`

Before writing the file, note the project ID and service account email printed during Steps 6.1–6.2. Substitute them below. The example uses placeholders deliberately; do not leave them unchanged.

```bash
cat > query.py <<'PYTHON_FILE'
from google.auth import compute_engine
from google.cloud import bigquery

# Use the service account attached to this Compute Engine VM.
credentials = compute_engine.Credentials(
    service_account_email="bigquery-qwiklab@YOUR_LAB_PROJECT_ID.iam.gserviceaccount.com"
)

query = """
SELECT
  name,
  SUM(number) AS total_people
FROM `bigquery-public-data.usa_names.usa_1910_2013`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
"""

client = bigquery.Client(
    project="YOUR_LAB_PROJECT_ID",
    credentials=credentials,
)

print(client.query(query).to_dataframe())
PYTHON_FILE
```

Edit the two occurrences of `YOUR_LAB_PROJECT_ID` so they contain the exact project ID from your lab. The service account email must also contain that same project ID.

One simple way to edit the file is:

```bash
nano query.py
```

Save with `Ctrl+O`, press Enter, then exit with `Ctrl+X`.

> **SQL syntax note:** The public table name is enclosed in backticks. Backticks identify a fully qualified BigQuery table name in Standard SQL. Do not replace them with single quotes.

### Step 6.7 — Run the Python query

Keep the virtual environment active and run:

```bash
python query.py
```

The script should print a dataframe containing up to 20 popular first names in Texas from the public USA names dataset.

If the output appears, the Python client authenticated with the attached service account and successfully ran the BigQuery query.

Return to the lab page and click **Check my progress** for **Use the client libraries to access BigQuery from a service account**.

### Common problems and fixes

| Error or symptom                                             | Likely cause                                                                                                                                                                                                              | Fix                                                                                                                                                                                            |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ModuleNotFoundError: No module named 'google.cloud'`        | BigQuery package was not installed in the active virtual environment.                                                                                                                                                     | Activate it with `source env/bin/activate`, then run `python -m pip install google-cloud-bigquery`.                                                                                            |
| `ModuleNotFoundError: No module named 'db_dtypes'`           | `to_dataframe()` needs the `db-dtypes` package.                                                                                                                                                                           | Run `python -m pip install db-dtypes pandas pyarrow` inside the active environment.                                                                                                            |
| `ModuleNotFoundError: No module named 'pandas'` or `pyarrow` | A dataframe dependency is missing.                                                                                                                                                                                        | Run `python -m pip install pandas pyarrow db-dtypes`.                                                                                                                                          |
| `externally-managed-environment` during pip install          | Packages are being installed into system Python on a modern Debian/Ubuntu image.                                                                                                                                          | Create and activate `env` with `python3 -m venv env`; install packages using `python -m pip` inside it. Do not force-install into system Python.                                               |
| `403` / `bigquery.jobs.create` permission denied             | The service account may be missing BigQuery User, or the wrong identity/project is being used.                                                                                                                            | Verify both BigQuery role bindings and the service-account email in the Python file.                                                                                                           |
| `403` / insufficient authentication scopes                   | The VM was created without a BigQuery-compatible OAuth scope.                                                                                                                                                             | Inspect its service account scopes. If the scope is wrong, recreate the VM with the scope in Step 6.3, after checking whether the lab allows deletion/recreation.                              |
| `Access Denied: Table ...`                                   | Table identifier/query or dataset access is incorrect.                                                                                                                                                                    | Use the exact public table name and backticks shown in Step 6.6; verify the query is running against the expected project.                                                                     |
| Python traceback after `source env/bin/activate`             | Activating a virtual environment does not install dependencies into it. The exact traceback from our earlier run was not preserved in the chat record, so its precise root cause cannot be confirmed from that log alone. | Check the final exception line, run `which python`, then `python -m pip show google-cloud-bigquery pandas pyarrow db-dtypes`. Install any missing package in the active environment and retry. |

**Important lesson from our run:** we activated `env` and then ran `python3 query.py`, but the command still produced a traceback. For a reproducible setup, activate the environment, install the dependencies using `python -m pip`, verify the package list, and use the environment's `python query.py`. If a traceback remains, diagnose its final exception line rather than repeatedly reinstalling packages blindly.

# Part C — Final verification checklist

Before ending the lab, return to the official ARC134 page and check each of the five checkpoint cards. The expected score is 20/20 for each item.

- [ ] **Checkpoint 1:** `devops` service account exists.
- [ ] **Checkpoint 2:** `devops` has `roles/iam.serviceAccountUser` and `roles/compute.instanceAdmin.v1`.
- [ ] **Checkpoint 3:** `vm-2` exists in the required zone and has the `devops` service account attached.
- [ ] **Checkpoint 4:** project-level role `customRole` exists and contains both Cloud SQL permissions.
- [ ] **Checkpoint 5:** `bigquery-qwiklab` has both BigQuery roles, `bigquery-instance` uses that service account, and `query.py` returns query results.
- [ ] The official page shows **100/100**.

Useful final inspection commands (run from `lab-vm` while the lab is still active):

```bash
# Project and location
gcloud config get-value project
gcloud config get-value compute/zone

# Service accounts
gcloud iam service-accounts list --format="table(displayName,email)"

# VM identities
gcloud compute instances describe vm-2 --zone="$ZONE" \
  --format="value(name,status,serviceAccounts[0].email)"
gcloud compute instances describe bigquery-instance --zone="$ZONE" \
  --format="value(name,status,serviceAccounts[0].email)"

# Custom role
gcloud iam roles describe customRole --project="$PROJECT_ID"
```

When all five progress checks are green and the score is 100/100, the challenge lab is complete. The course Skill Badge/credential is a separate course-level outcome; follow the course page's credential-claim flow if it displays one.

# Part D — What we learned from the issues in our chat

These notes are included because they are easy points to lose when following short command dumps.

## 1. Finding the service account email

You asked how to retrieve the service account email for ARC134. The command is:

```bash
gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)"
```

To list every service account and its email:

```bash
gcloud iam service-accounts list --format="table(displayName,email)"
```

To see the currently authenticated human/user account (not the service account):

```bash
gcloud auth list
```

This distinction matters: `gcloud auth list` and `gcloud iam service-accounts list` answer different questions.

## 2. The BigQuery Python traceback

In our earlier ARC134 session, the terminal showed the virtual environment being activated with `source env/bin/activate`, followed by a traceback from `python3 query.py`. The full traceback text is not available in the retained chat context, so this guide does **not** claim a specific confirmed root cause. The robust resolution path is to verify the active interpreter, install all four dependencies inside that interpreter, use the exact service account email/project ID in the script, and inspect the final traceback line if the run still fails.

## 3. Avoid using a helper script with a hard-coded region

The ePlus article links to a quick automated shell script, but the public script currently sets `us-central1` / `us-central1-a`, while the current walkthrough text for ARC134 says `us-east4` / `us-east4-a`. The script also contains logic that can delete and recreate VMs when it detects a mismatch. For this reason, this guide uses explicit manual commands and the current lab's stated region/zone instead of asking readers to run the script blindly.

## 4. Never carry values from another lab session into a new one

Every challenge-lab session can create a different temporary project and temporary student account. Always rediscover the project ID and service account email during the current run. Do not reuse a previous session's ID or credentials.

# References

1. **Official Google Skills ARC134 challenge lab** — task sequence, five scored checkpoints, lab requirements, and final BigQuery query:  
   https://www.skills.google/focuses/67219?locale=en&parent=catalog
2. **ePlus.dev ARC134 walkthrough** — challenge scenario, `lab-vm`, service account names, task flow, region/zone note, and manual walkthrough link:  
   https://eplus.dev/configure-service-accounts-and-iam-roles-for-google-cloud-challenge-lab-arc134
3. **Cloud-Wala-Banda command reference** — sample commands for service accounts, IAM bindings, `vm-2`, custom role YAML, and BigQuery setup:  
   https://github.com/Cloud-Wala-Banda/Google-Cloud-Lab-Solutions/blob/main/Configure%20Service%20Accounts%20and%20IAM%20for%20Google%20Cloud%20Challenge%20Lab.md
4. **Google Cloud CLI: create a service account**:  
   https://cloud.google.com/sdk/gcloud/reference/iam/service-accounts/create
5. **Google Cloud CLI: add a project IAM policy binding**:  
   https://cloud.google.com/sdk/gcloud/reference/projects/add-iam-policy-binding
6. **Google Cloud IAM: create and manage custom roles**:  
   https://cloud.google.com/iam/docs/creating-custom-roles
7. **Google Cloud Compute Engine: service accounts and scopes**:  
   https://cloud.google.com/compute/docs/access/service-accounts
8. **BigQuery: query a public dataset with client libraries**:  
   https://cloud.google.com/bigquery/docs/quickstarts/quickstart-client-libraries
9. **ePlus ARC134 helper script (review only; do not blindly execute)** — its published copy hard-codes a different region and includes VM delete/recreate logic:  
   https://raw.githubusercontent.com/ePlus-DEV/storage/refs/heads/main/labs/ARC134/lab.sh

---

_Guide prepared as a practical companion to ARC134. Always follow the active lab page if its task wording, project values, resource names, region, or zone differ from this document._
