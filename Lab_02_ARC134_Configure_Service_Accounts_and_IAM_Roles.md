# ARC134: Configure Service Accounts and IAM Roles for Google Cloud (Challenge Lab)

A complete walkthrough with the common mistakes to avoid and how to fix them.

| | |
|---|---|
| **Lab ID** | ARC134 |
| **Official title** | Configure Service Accounts and IAM Roles for Google Cloud: Challenge Lab |
| **Scoring** | 100 points across 5 checkpoints (20 points each) |
| **Time** | 2 minutes setup, 45 minutes of access, about 15 minutes of actual work |
| **Official page version used** | Manual and lab last tested October 27, 2026 |
| **Where you work** | Cloud Shell, plus two VMs you create and SSH into (and `lab-vm` if your lab provides one) |

| Checkpoint | Official task | Points |
|---|---|---|
| Create a service account using gcloud | Task 2 | 20 |
| Grant IAM permissions to the service account using gcloud | Task 3 | 20 |
| Create a compute instance with the service account attached using gcloud | Task 4 | 20 |
| Create a custom role using a YAML file | Task 5 | 20 |
| Use the client libraries to access BigQuery from a service account | Task 6 | 20 |

Task 1 on the official page (explore Gemini) is optional and has no checkpoint, which is why the scored tasks run from Task 2 to Task 6.

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake to avoid** describe a common error and its fix. Blocks marked **Note** cover things the official page doesn't spell out.
>
> **Values in `UPPERCASE` or starting with `YOUR_` are placeholders.** Every lab run gets its own Project ID, Region and Zone, shown in the lab panel. Always copy them from your own panel. Values such as `us-east4-a` or `us-east1-c` appear in different runs, so never copy a region or zone from a guide.
>
> **The live lab takes priority.** Challenge labs can be updated. If a VM name, role, region, zone or checkpoint in your active lab differs from this guide, follow your lab.

**Official lab:** [ARC134 on Google Skills](https://www.skills.google/focuses/67219?locale=en&parent=catalog)

---

# Part 1: Conceptual

## What this lab is about

Challenge labs don't teach new concepts. They give you a scenario and expect you to apply what the course taught. In ARC134 you act as a junior cloud architect who must give programs and virtual machines access to Google Cloud **without using a person's account**. In order, you:

1. Create a **service account** named `devops`.
2. Grant it the **roles** it needs.
3. Create a VM that **runs as** that service account and test it.
4. Create a **custom role** from a YAML file.
5. Create a second service account, attach it to another VM, and run a Python program on that VM that queries BigQuery.

## Important concepts, explained simply

**Service account.** An identity for a program or VM, not a person. It has no password, and its email looks like `NAME@PROJECT_ID.iam.gserviceaccount.com`. This lab uses two: `devops` (for `vm-2`) and `bigquery-qwiklab` (for `bigquery-instance`).

**IAM, permission and role.** IAM decides who can do what on which resource. A *permission* is one action, like `compute.instances.list`. A *role* is a named bundle of permissions.

**IAM binding.** One line in a project's access list: "this member has this role". `add-iam-policy-binding` adds one. The member must start with its type, so a service account is written `serviceAccount:EMAIL`, not `user:EMAIL`.

**Predefined vs custom role.** A predefined role (such as `roles/bigquery.user`) is made by Google. A custom role is one you define by picking exact permissions, which follows the principle of least privilege: give only what's needed.

**`roles/iam.serviceAccountUser`.** The role that lets an identity *act as* a service account, which attaching one to a VM requires. Without it, creating a VM that runs as `devops` fails with an `actAs` error.

**Compute Instance Admin.** The role that lets an identity manage VMs. Google has two role IDs: `roles/compute.instanceAdmin.v1` (stable) and `roles/compute.instanceAdmin` (the older one the console labels "beta"). The lab says "compute.instanceAdmin" in words, so Task 3 explains what to do if the checker wants the other one.

**Attached service account and metadata server.** A VM created with `--service-account` runs as that identity. Programs on it get credentials from the *metadata server*, an internal address only the VM can reach, so no key file is needed.

**Access scopes.** A second limit on what a VM may call. A role says what an identity is *allowed* to do, and a scope says which Google APIs the *VM* may reach at all. A VM created without `--scopes` gets a small default set that doesn't include the Compute Engine API, so `gcloud compute instances list` run on it fails even if the role is right. Scopes never grant extra permission.

**Custom role YAML file.** A small text file with four fields: `title`, `description`, `stage` (`ALPHA`, `BETA` or `GA`) and `includedPermissions`. This lab's role contains only `cloudsql.instances.connect` and `cloudsql.instances.get`. YAML is whitespace-sensitive, so indentation mistakes break it.

**BigQuery roles.** `roles/bigquery.dataViewer` lets an account read table data. `roles/bigquery.user` lets it run query jobs. The query needs both.

**BigQuery client library.** Python's `google-cloud-bigquery` lets code run SQL without building web requests by hand. The line `compute_engine.Credentials(service_account_email=...)` tells it to get its token from the VM's metadata server as that service account.

**Virtual environment.** A private folder of Python packages (`python3 -m venv env`). Modern Debian and Ubuntu images block plain `pip install`, so you install into an environment instead. Activating it installs nothing. The packages must be installed after activating.

**IAM conditions.** Labs sometimes attach a condition to a student's access, such as `allow_only_curriculum_roles`, which limits which roles the student can hand out. It's a lab guardrail, not a bug to work around.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. **Copy** the Username, Password, Project ID, Region and Zone from the lab panel.
3. Click **Open Google Cloud console** and sign in with only that student account. Accept the terms and skip the recovery page.
4. Check that the project selector shows your `qwiklabs-gcp-...` project.
5. **Wait two to three minutes** after the lab starts before running commands. Permissions can take a moment to apply.
6. Keep the official task page open so you can click **Check my progress** after each task.
7. Don't click **End Lab** until all five checkpoints show 20/20.

> **Note: one lab at a time.** Google Skills runs only one lab at once. Starting this lab asks you to end all existing labs, so finish and score any other running lab first.

> **Mistake to avoid: several student accounts in one browser.** Opening labs back to back in the same browser profile leaves several student accounts signed in, and the Console and Cloud Shell can end up using different accounts. The account chooser lists them, some marked "Signed out".
>
> **Fix.**
> 1. Close every other Google window, including old Console tabs.
> 2. Open a **new private window** and sign in with **only** the account that matches the Username in your current lab panel.
> 3. Check your identity: click your profile picture in the Console, and in Cloud Shell run `gcloud auth list`. Both must match your lab panel.

> **Mistake to avoid: sharing credentials in screenshots.** The panel shows the temporary password. It only works for the lab account, but crop it out when asking for help.

## Step 1: Set up your terminal

### 1.1 Open Cloud Shell and check your identity

Click **Activate Cloud Shell** (`>_`), then **Continue** and **Authorize**. Run:

```bash
gcloud auth list
gcloud config get-value project
```

The account marked `*` must match the lab panel's username, and the project must match the Project ID. Don't switch to a personal account.

### 1.2 Set the project, region and zone

Use the values from your lab panel:

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="REGION_FROM_PANEL"
export ZONE="ZONE_FROM_PANEL"
gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"

printf 'Project: %s\nRegion: %s\nZone: %s\n' "$PROJECT_ID" "$REGION" "$ZONE"
```

All three lines must print real values. If the project is blank or `(unset)`, run `gcloud config set project YOUR_LAB_PROJECT_ID`.

> **Mistake to avoid: empty variables.** Shell variables vanish when Cloud Shell reconnects and never carry into an SSH session. A blank `$PROJECT_ID`, `$ZONE` or `$SA` causes confusing errors later, such as `--member serviceAccount:` with nothing after the colon. Rerun this step and `echo` the values whenever a command complains about a missing argument.

### 1.3 Where to run the commands

| Terminal | Prompt looks like | Used for |
|---|---|---|
| **Cloud Shell** | `student_...@cloudshell:~$` | The default place for every `gcloud` command in Tasks 2 to 6 |
| **`lab-vm`** (only if your lab provides it) | `...@lab-vm:~$` | A pre-created VM. Use it only if your VM list shows it **and** your live instructions tell you to run the commands there |
| **`vm-2`** | `...@vm-2:~$` | Only the test in Task 4 |
| **`bigquery-instance`** | `...@bigquery-instance:~$` | Only the Python query in Task 6 |

To check for `lab-vm`, open **Navigation menu > Compute Engine > VM instances**. If it exists and your instructions send you there, connect and set it up:

```bash
gcloud compute ssh lab-vm --project="$PROJECT_ID" --zone="$ZONE"
```

If the VM asks you to sign in, run `gcloud auth login --no-launch-browser`, open the printed link, sign in with the **lab's student account** and paste the verification code back (never share it). Then rerun the variable block from 1.2 on the VM.

> **Note.** If your live lab's hint tells you to switch to a specifically named gcloud configuration, use the exact name and account it shows and return to `default` as instructed. Don't copy configuration names (such as `user2`) from other IAM labs.

---

## Task 1: Enable and explore Gemini (optional, no checkpoint)

This activity isn't scored, and you can skip it. Current versions of the lab let you open Gemini Cloud Assist inside the console. To try it, click the Gemini icon at the top right, enable Gemini Cloud Assist if prompted and click **Start Chatting**. This guide lists the commands Gemini is expected to suggest.

> **Mistake to avoid: waiting on Gemini.** Earlier versions of the lab had Gemini unavailable. If it fails to enable or respond, move on, since nothing in the five checkpoints depends on it. Gemini's answers can look right and still be wrong, so check any command against this guide before running it.

---

## Task 2: Create the `devops` service account (20 points)

### 2.1 Create it

```bash
gcloud iam service-accounts create devops \
  --display-name="devops" \
  --project="$PROJECT_ID"
```

Expected output: `Created service account [devops].`

### 2.2 Get its email and store it

```bash
export SA="$(gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)")"

echo "$SA"
```

It should print something like `devops@YOUR_LAB_PROJECT_ID.iam.gserviceaccount.com`, using your own project ID. If it prints nothing, set it directly:

```bash
export SA="devops@${PROJECT_ID}.iam.gserviceaccount.com"
```

To see every service account and its email:

```bash
gcloud iam service-accounts list --format="table(displayName,email)"
```

> **Note: two different "list" commands.** `gcloud auth list` shows the *human* account you're signed in as. `gcloud iam service-accounts list` shows the *service accounts in the project*.

### 2.3 Verify it exists and check progress

```bash
gcloud iam service-accounts describe "$SA" --project="$PROJECT_ID"
```

Click **Check my progress** under Task 2 and confirm 20/20.

> **Mistake to avoid: `IAM_PERMISSION_DENIED` on the very first command.** Some runs return:
>
> ```
> ERROR: (gcloud.iam.service-accounts.create) [student-...@qwiklabs.net] does not have
> permission to access projects instance ...: Permission 'iam.serviceAccounts.create' denied
> reason: IAM_PERMISSION_DENIED
> ```
>
> In one run, the account and project both matched the lab panel, so it wasn't a wrong-account problem. The student account's roles (visible on **IAM & Admin > IAM**) were BigQuery Admin, Compute Instance Admin (beta), Compute Network Admin, Project IAM Admin (with the `allow_only_curriculum_roles` condition), Service Usage Consumer, Student Custom Role Creator and Viewer (Legacy). None of those includes creating service accounts.
>
> **Fix, in order. Stop as soon as one works:**
> 1. **Rule out the common causes.** Compare the profile-picture email and `gcloud auth list` with your lab panel. If another student account is signed in, start again from a clean private window.
> 2. **Wait and retry.** Give the lab a few minutes to finish provisioning, then rerun the command.
> 3. **Check whether `devops` already exists.** Run `gcloud iam service-accounts list`. If `devops@...` is listed, the lab setup may have created it, so store its email (2.2) and continue.
> 4. **Check your roles.** On **IAM & Admin > IAM**, find your student username. A working lab should include a role that allows creating service accounts, such as Service Account Admin.
> 5. **Try the other terminal.** If your lab has a `lab-vm` and your live instructions send you there (1.3), run the command from it.
> 6. **Refresh the login.** Close and reopen Cloud Shell and click **Authorize**, or run `gcloud auth login --no-launch-browser`.
> 7. **Restart the lab** only after the steps above: **End Lab**, then **Start Lab** for a fresh project, and sign in with the new credentials in a clean private window. Nothing is lost if you haven't created anything yet, but each restart uses lab time.
> 8. **Report it.** If a fresh project fails the same way, use **Send feedback** on the lab page with lab ID ARC134, the error text, the roles list and your project ID.
>
> Don't try to get around the `allow_only_curriculum_roles` condition (editing it, using another account and so on). It's a lab guardrail, and bypassing it could affect scoring.

> **Mistake to avoid: trusting a green tick without checking.** In one run, the first checkpoints showed as complete even though the create command had been denied. The lab setup may have created the account already, or the checker may not have refreshed. Always confirm with `gcloud iam service-accounts list` that `devops` really exists, because the next tasks depend on it.

> **Mistake to avoid: `ALREADY_EXISTS`.** If the account was created earlier in the same project, creating it again fails. Don't create a second one. List the accounts, store the email in `SA` and continue.

---

## Task 3: Grant IAM permissions to `devops` (20 points)

The official steps are: store the project ID and service account in variables, keep the email in `SA`, then give the service account the role of `iam.serviceAccountUser` with the permissions `compute.instanceAdmin`.

### 3.1 Reconfirm the variables

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export SA="$(gcloud iam service-accounts list \
  --filter="displayName=devops" \
  --format="value(email)")"

echo "PROJECT_ID=$PROJECT_ID"
echo "SA=$SA"
```

Both must print a value before you continue.

### 3.2 Grant Service Account User

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA" \
  --role="roles/iam.serviceAccountUser"
```

### 3.3 Grant Compute Instance Admin

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA" \
  --role="roles/compute.instanceAdmin.v1"
```

The lab's wording is loose. The usual reading is to grant **both** roles, because attaching `devops` to a VM needs `serviceAccountUser`, and the Task 4 test needs instance admin rights.

### 3.4 Verify and check progress

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:$SA" \
  --format="table(bindings.role,bindings.members)"
```

The output must include `roles/iam.serviceAccountUser` and `roles/compute.instanceAdmin.v1`. Click **Check my progress** under Task 3.

> **Mistake to avoid: a checkpoint that stays red although both bindings exist.** The lab text says `compute.instanceAdmin`, which matches the older **beta** role ID, while this guide uses `roles/compute.instanceAdmin.v1`. If the checkpoint stays red after you've confirmed the `.v1` binding, also grant the other role (and check the hint box in your lab for the exact role it names):
>
> ```bash
> gcloud projects add-iam-policy-binding "$PROJECT_ID" \
>   --member="serviceAccount:$SA" \
>   --role="roles/compute.instanceAdmin"
> ```
>
> Wait 20 seconds and click **Check my progress** again.

> **Mistake to avoid: wrong member type, spelling or project.** The member must start with `serviceAccount:`, not `user:`. Use full role IDs (`roles/...`), not display names like "Compute Instance Admin". The first argument must be your project ID. If a binding already exists, verify it with the policy command instead of adding it repeatedly.

---

## Task 4: Create `vm-2` with the `devops` service account attached (20 points)

### 4.1 Create the VM

Make sure `$SA` still holds the `devops` email (rerun 3.1 if not):

```bash
gcloud compute instances create vm-2 \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --service-account="$SA" \
  --scopes="https://www.googleapis.com/auth/cloud-platform"
```

- The name must be exactly `vm-2`, and the zone must be the one in your lab panel.
- If it asks to enable the Compute Engine API, answer **y**.
- If your live hint specifies a machine type, image or other flag, add those exact values too.

> **Mistake to avoid: leaving out `--scopes`.** Without it, the VM gets a small default set of scopes that doesn't include the Compute Engine API. The checkpoint (a VM named `vm-2` with `devops` attached) can still pass, but the test in 4.3 then fails with *"Request had insufficient authentication scopes"*, even though the IAM roles are correct.
>
> **If you already created `vm-2` without scopes,** don't delete it. Stop it, set the scopes and start it again:
>
> ```bash
> gcloud compute instances stop vm-2 --project="$PROJECT_ID" --zone="$ZONE"
> gcloud compute instances set-service-account vm-2 \
>   --project="$PROJECT_ID" --zone="$ZONE" \
>   --service-account="$SA" \
>   --scopes="https://www.googleapis.com/auth/cloud-platform"
> gcloud compute instances start vm-2 --project="$PROJECT_ID" --zone="$ZONE"
> ```

> **Mistake to avoid: leaving out `--service-account`.** Without it, the VM runs as the default Compute account instead of `devops`, and the checker looks for `devops`.

### 4.2 Verify and check progress

```bash
gcloud compute instances describe vm-2 \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --format="yaml(name,status,serviceAccounts)"
```

Confirm `name` is `vm-2`, `status` is `RUNNING`, and `serviceAccounts` contains the `devops` email and the `cloud-platform` scope. Click **Check my progress** under Task 4.

### 4.3 SSH in and test, as the official task asks

```bash
gcloud compute ssh vm-2 --project="$PROJECT_ID" --zone="$ZONE"
```

If it asks you to create an SSH key, press **Enter** for an empty passphrase (twice). Inside `vm-2`, variables from the other terminal don't exist, so replace the `YOUR_...` values below with your real ones from the lab panel. First check the attached identity:

```bash
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email
```

It must print the `devops` email. Then list instances:

```bash
gcloud compute instances list --project="YOUR_LAB_PROJECT_ID"
```

The official task also asks you to try **creating** an instance from `vm-2`. Create a small temporary test VM, confirm it appears and delete only that VM:

```bash
gcloud compute instances create permission-test-vm \
  --project="YOUR_LAB_PROJECT_ID" \
  --zone="YOUR_ZONE" \
  --machine-type="e2-micro"

gcloud compute instances list --project="YOUR_LAB_PROJECT_ID"

gcloud compute instances delete permission-test-vm \
  --project="YOUR_LAB_PROJECT_ID" \
  --zone="YOUR_ZONE" --quiet
```

Then return to your previous terminal:

```bash
exit
```

> **Mistake to avoid: deleting `vm-2`.** Delete only `permission-test-vm`. Never delete `vm-2` to clear an error, because the checkpoint needs it.
>
> **Mistake to avoid: typing commands in the wrong terminal.** The prompt tells you where you are: `...@vm-2:~$` is the test VM. Run `exit` before continuing.

| Problem | Likely cause | Fix |
|---|---|---|
| "service account does not exist" while creating `vm-2` | `$SA` is empty or has a typo | `echo "$SA"`, list the accounts and use the exact email |
| `iam.serviceAccounts.actAs` permission error | The `serviceAccountUser` binding is missing or on the wrong member | Repeat Task 3 and verify the binding |
| VM created in the wrong zone | A default or old value was used | Pass `--zone="$ZONE"` explicitly |
| SSH fails right after creation | The VM is still booting | Wait for `RUNNING` in the describe output and retry |
| `vm-2` already exists | A previous attempt created it | Describe it first and reuse it if its service account is correct |
| `instances list` returns 403 or "insufficient authentication scopes" inside `vm-2` | VM created without a compute-capable scope, or an IAM role is missing | Use the stop, set-service-account, start fix above, and verify the `devops` roles |

---

## Task 5: Create a custom role from a YAML file (20 points)

The file must be named `role-definition.yaml`, and the role needs the permissions `cloudsql.instances.connect` and `cloudsql.instances.get`. The official page doesn't name the role ID, so this guide uses `customRole`. If your lab's hint names a different ID, use that one.

### 5.1 Create the YAML file

```bash
cat > role-definition.yaml <<'ROLE_YAML'
title: "Cloud SQL Instance Connector"
description: "Custom role with Cloud SQL connect and get permissions"
stage: "GA"
includedPermissions:
  - cloudsql.instances.connect
  - cloudsql.instances.get
ROLE_YAML

cat role-definition.yaml
```

> **Mistake to avoid: YAML formatting errors.** `includedPermissions:` must start at the left edge, and each permission is a line starting with `- `, indented consistently under it. A missing space after the dash or mixed indentation causes a parsing error. Don't paste smart (curly) quotes from a document, since plain quotes are required.

### 5.2 Create the project-level role

```bash
gcloud iam roles create customRole \
  --project="$PROJECT_ID" \
  --file="role-definition.yaml"
```

> **Mistake to avoid: wrong level or reused ID.** The task requires a project-level role, so keep `--project="$PROJECT_ID"` and don't use `--organization`. If the role already exists from an earlier attempt and the definition needs correcting, update it instead of recreating it:
>
> ```bash
> gcloud iam roles update customRole \
>   --project="$PROJECT_ID" \
>   --file="role-definition.yaml"
> ```

### 5.3 Verify and check progress

```bash
gcloud iam roles describe customRole --project="$PROJECT_ID"
```

The output must list both `cloudsql.instances.connect` and `cloudsql.instances.get`. Click **Check my progress** under Task 5.

---

## Task 6: Query BigQuery from a service account (20 points)

The official steps are: create a service account named `bigquery-qwiklab` with the roles **BigQuery Data Viewer** and **BigQuery User**, create a VM named `bigquery-instance` using it, SSH in and install the dependencies, create `query.py`, replace the two placeholders, and run it.

### 6.1 Create the service account

```bash
gcloud iam service-accounts create bigquery-qwiklab \
  --display-name="bigquery-qwiklab" \
  --project="$PROJECT_ID"

export BQ_SA="$(gcloud iam service-accounts list \
  --filter="displayName=bigquery-qwiklab" \
  --format="value(email)")"

echo "$BQ_SA"
```

Expected format: `bigquery-qwiklab@YOUR_LAB_PROJECT_ID.iam.gserviceaccount.com`. If the account already exists, skip creation and store the email. If creation returns `IAM_PERMISSION_DENIED`, use the fix list under Task 2.

### 6.2 Grant the two BigQuery roles

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$BQ_SA" \
  --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$BQ_SA" \
  --role="roles/bigquery.user"
```

Verify both:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:$BQ_SA" \
  --format="table(bindings.role,bindings.members)"
```

### 6.3 Create `bigquery-instance`

```bash
gcloud compute instances create bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --service-account="$BQ_SA" \
  --scopes="https://www.googleapis.com/auth/bigquery"
```

The BigQuery scope lets the VM request BigQuery API access, and IAM roles still decide what the service account may do. (The broader `cloud-platform` scope also works.) Verify the identity:

```bash
gcloud compute instances describe bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE" \
  --format="yaml(name,status,serviceAccounts)"
```

> **Mistake to avoid: attaching `devops` by accident.** The VM must use `bigquery-qwiklab`. Check that `$BQ_SA`, and not `$SA`, is in the command.
>
> **Mistake to avoid: a wrong or missing scope.** If the VM's scope doesn't cover BigQuery, the query fails with a 403 about insufficient authentication scopes. Fix it as in Task 4: stop the VM, run `gcloud compute instances set-service-account ... --scopes=...` and start it again.

### 6.4 SSH in and install the dependencies

```bash
gcloud compute ssh bigquery-instance \
  --project="$PROJECT_ID" \
  --zone="$ZONE"
```

The prompt should change to `...@bigquery-instance:~$`. The page just says "install the dependencies". Use a virtual environment (this setup is a suggestion, not given on the page):

```bash
sudo apt-get update
sudo apt-get install -y python3 python3-venv python3-pip

python3 -m venv env
source env/bin/activate

python -m pip install --upgrade pip
python -m pip install google-cloud-bigquery pandas pyarrow db-dtypes
```

Verify the environment and packages:

```bash
which python
python --version
python -m pip show google-cloud-bigquery pandas pyarrow db-dtypes
```

`which python` must include `/env/bin/python`, and the prompt should start with `(env)`.

> **Mistake to avoid: assuming activation installs anything.** `source env/bin/activate` only switches to an empty environment. Install the packages **after** activating, using `python -m pip install ...`, so they land in the same Python that runs the script.

### 6.5 Create `query.py`

This is the code from the official page. The two `YOUR_...` placeholders are replaced in 6.6.

```bash
cat > query.py <<'PYTHON_FILE'
from google.auth import compute_engine
from google.cloud import bigquery
credentials = compute_engine.Credentials(
    service_account_email='YOUR_SERVICE_ACCOUNT')
query = '''
SELECT name, SUM(number) as total_people
FROM `bigquery-public-data.usa_names.usa_1910_2013`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
'''
client = bigquery.Client(
    project='YOUR_PROJECT_ID',
    credentials=credentials)
print(client.query(query).to_dataframe())
PYTHON_FILE
```

> **Note: the backticks in the SQL.** The table name is wrapped in backticks, which mark a fully qualified BigQuery table name. Don't replace them with quotes.

### 6.6 Replace the two placeholders

Variables from the other terminal don't exist on this VM, so read the real values from the metadata server:

```bash
PROJECT_ID=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/project/project-id)
BQ_SA="bigquery-qwiklab@${PROJECT_ID}.iam.gserviceaccount.com"

sed -i "s/YOUR_PROJECT_ID/${PROJECT_ID}/" query.py
sed -i "s/YOUR_SERVICE_ACCOUNT/${BQ_SA}/" query.py

cat query.py
```

Check that `cat query.py` shows your real project ID and the full `bigquery-qwiklab@...` email, with no `YOUR_...` text left. You can also edit by hand with `nano query.py` (save with `Ctrl+O`, **Enter**, then `Ctrl+X`).

> **Mistake to avoid: leaving a placeholder in the file.** A leftover `YOUR_SERVICE_ACCOUNT` or `YOUR_PROJECT_ID` makes the script ask for an account that doesn't exist and produces a traceback. Always run `cat query.py` before running it.

### 6.7 Run the query and check progress

With the virtual environment still active:

```bash
python query.py
```

The script should print a table of up to 20 popular first names in Texas with totals. That means the Python client authenticated as the attached service account and ran the query. Then leave the VM:

```bash
exit
```

Click **Check my progress** under Task 6.

Read the **last line** of any traceback, because it names the real problem:

| Error or symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'google.cloud'` | BigQuery package isn't installed in the active environment | `source env/bin/activate`, then `python -m pip install google-cloud-bigquery` |
| `ModuleNotFoundError: ... 'db_dtypes'`, `'pandas'` or `'pyarrow'` | `to_dataframe()` needs these | `python -m pip install pandas pyarrow db-dtypes` |
| `externally-managed-environment` while installing | Packages are going into the system Python | Create and activate `env`, and install with `python -m pip`. Don't force-install into system Python |
| `403` with `bigquery.jobs.create` denied | Missing BigQuery User role, or the wrong identity or project | Recheck both role bindings and the email in `query.py` |
| `403` with insufficient authentication scopes | VM created without a scope that covers BigQuery | Stop the VM, run `set-service-account` with a BigQuery-capable scope and start it again |
| Metadata server error, or `404` for the service account | A `YOUR_...` placeholder is still in the file, or the VM isn't using `bigquery-qwiklab` | `cat query.py` and check the describe output from 6.3 |
| `Access Denied: Table ...` | Wrong table name or project | Use the exact table name with backticks and your own project ID |
| Traceback right after `source env/bin/activate` | Activating an environment installs nothing | Run `which python`, then `python -m pip show google-cloud-bigquery pandas pyarrow db-dtypes`, install anything missing in the active environment and retry |

If a traceback remains after these checks, diagnose its final exception line instead of reinstalling packages repeatedly. A reliable order is: activate the environment, install with `python -m pip`, verify with `pip show`, check `cat query.py`, then run `python query.py`.

---

## Finish

1. Confirm all five checkpoints show 20/20 and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab clears your work and removes the temporary project, and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the exact names: `devops`, `vm-2`, `role-definition.yaml`, `bigquery-qwiklab` and `bigquery-instance`.

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, names and checkpoints. It adds the explanations and warnings below. If your live lab differs from anything here, the live lab wins.

| Official lab | This guide |
|---|---|
| Task 1: enable and explore Gemini (optional) | Marked optional and skippable, with a note to verify any command Gemini suggests. |
| Task 2: create service account `devops` | Same command, run in Cloud Shell. Adds the email lookup and a fix list for `IAM_PERMISSION_DENIED`, `ALREADY_EXISTS` and false green ticks. |
| Task 3: variables for project ID and `SA`, then grant `iam.serviceAccountUser` with `compute.instanceAdmin` | Same steps. Explains the loose wording (grant both roles), uses the full role ID `roles/compute.instanceAdmin.v1`, and says what to do if the checker wants the beta role. |
| Task 4: create `vm-2` with `devops` attached, SSH, create and list an instance | Same steps. Adds the `--scopes` flag, the metadata-server identity check and the cleanup of the temporary test VM. |
| Task 5: `role-definition.yaml` with two Cloud SQL permissions, create the role at project level | Same steps. Uses role ID `customRole`, and adds the YAML formatting and update-instead-of-recreate advice. |
| Task 6: `bigquery-qwiklab` with BigQuery Data Viewer and User, `bigquery-instance`, install dependencies, `query.py`, replace placeholders, run | Same steps. Adds the virtual-environment setup, the metadata-server shortcut for the placeholders, and an error table for the traceback. |

> **What the official page doesn't show.** Your project ID, region and zone appear only in your running lab's panel. The role ID for Task 5, the exact dependency list for Task 6 and the Compute Instance Admin role ID aren't fully specified on the page, so this guide suggests values that work. If a hint box in your lab says otherwise, follow your lab.
