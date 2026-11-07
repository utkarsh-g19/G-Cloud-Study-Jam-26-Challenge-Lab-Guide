# ARC100: Store, Process, and Manage Data on Google Cloud (Challenge Lab)

A complete walkthrough with the mistakes we made and how we fixed them.

| | |
|---|---|
| **Lab ID** | ARC100 |
| **Official title** | Store, Process, and Manage Data on Google Cloud: Challenge Lab |
| **Scoring** | 100 points across 3 checkpoints |
| **Time** | 45 minutes of access. The page estimates 15 minutes of work, and our permission problems cost us extra time |
| **Official page version used** | Manual last updated and lab last tested October 28, 2026 |
| **Where you work** | The Google Cloud console, plus Cloud Shell for fixing permissions |

| Checkpoint | Points |
|---|---|
| Task 1: Create a bucket | 30 |
| Task 2: Create a Pub/Sub topic | 30 |
| Task 3 and Task 4: Create the Cloud Run function (verified by the thumbnail test) | 40 |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake we made** show a real error from our run and the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values written in `UPPERCASE` are placeholders.** Your lab assigns its own bucket name, topic name, function name and region, and the public version of the lab page leaves them blank. They only appear inside your running lab, so **always copy them from your own lab page.** In our run the region was `us-central1`, the bucket began with `memories-bucket-`, and the function was called `memories-thumbnail-creator`.

---

# Part 1: Conceptual

## What this lab is about

A team is building a photo-storing app called **Memories**. You set up the back end that makes a small thumbnail of every photo that's uploaded. In order, you:

1. Create a **Cloud Storage bucket** to hold the photos.
2. Create a **Pub/Sub topic** that the function uses to announce what it did.
3. Create a **Cloud Run function** that runs automatically whenever a new image lands in the bucket, shrinks it to 64x64 pixels, saves the thumbnail back into the bucket and publishes a message.
4. Upload a photo and check that a thumbnail appears.

Challenge labs give you the goal but not the clicks, so you apply what the course taught.

## Important concepts, explained simply

**Cloud Storage and bucket.** Google's file storage. A *bucket* is the top-level container that holds files (called *objects*).

**Location type: region, zone, multi-region.** Where the bucket's data lives. A **region** (like `us-central1`) is a geographic area. A **zone** is one data center inside a region and creates a special kind of bucket with limited features. A **multi-region** spans a large area. This lab asks for a bucket in a **region**.

**Public access prevention.** A bucket setting that blocks anyone on the internet from reading your files. It's on by default and this lab doesn't need it off.

**Pub/Sub topic.** A named channel for messages. One program publishes a message to the topic and others can receive it. Here, the function publishes the new thumbnail's name.

**Cloud Run function (2nd generation).** A small piece of code that Google runs for you whenever something happens, with no server for you to manage. The lab insists on the **2nd generation** type because it's built on Cloud Run and uses Eventarc triggers.

**Trigger and Eventarc.** A trigger says "run this function when X happens". **Eventarc** is the service that watches for the event. Here the event is "a new object was finished uploading to the bucket" (`google.cloud.storage.object.v1.finalized`).

**Entry point.** The name of the function inside your code that Google should start. It must match the name in `functions.cloudEvent('...')` in `index.js`. The editor's default, `helloGCS`, is a template placeholder.

**`index.js` and `package.json`.** `index.js` is the code. `package.json` lists the libraries it needs (including `sharp`, which does the resizing).

**Service account and role.** A *service account* is an identity a program uses, like a robot user. A *role* is a bundle of permissions you give it. The trigger needs specific roles to read the bucket and receive events.

**IAM condition.** A rule attached to a permission. The student account in these labs may only grant a short list of roles. If you try anything else, Google asks you to specify a condition or refuses (see the mistakes below).

**Permission propagation.** After you grant a role, it can take a few minutes to take effect everywhere. Acting too soon causes "permission denied" errors that disappear if you wait.

**`run-sources-...` bucket.** A second bucket that Cloud Run creates automatically to hold your function's source code. It's not your photo bucket.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. **Write down from your lab page** the bucket name (Task 1), the region, the Pub/Sub topic name (Task 2) and the function name and entry point (Task 3).
5. Don't click **End Lab** until all three checkpoints are green.

> **Note: naming and resources.** The official page asks for sensible names, to use the project's own network and to avoid oversized resources, and warns that excessive resource use can get the project terminated. Don't create extra buckets, topics or functions.

---

## Task 1: Create a bucket (30 points)

### 1.1 Open the bucket form

1. Click the **Navigation menu** (☰) > **Cloud Storage** > **Buckets**.
2. Click **Create**.

### 1.2 Name and location

1. In **Name your bucket**, type the bucket name from your lab page and click **Continue**.
2. Under **Location type**, click **Region**.
3. In the region dropdown, pick the region from your lab page (ours was `us-central1`) and click **Continue**.
4. Leave the storage class, access control and protection settings on their defaults, clicking **Continue** through each.
5. Click **Create**.

> **Mistake we made: choosing the wrong location type.** The form opened with **Zone** selected, and its extra dropdowns (Region and Zone) made it look like the right choice for a single region.
>
> **Fix.** Zone creates a special single-zone bucket with limited compatibility, and the lab wants a normal regional bucket. Click the **Region** option, then pick the region from the dropdown. After creation, the bucket's details should show **Location type: Region** and the correct location. A bucket's location can't be changed afterwards, so if it's wrong, delete the bucket and make it again.

> **Mistake we almost made: the public access dialog.** When we clicked **Create**, a dialog titled "Public access will be prevented" appeared with **Enforce public access prevention on this bucket** ticked.
>
> **Fix.** Leave it ticked and click **Confirm**. The function reads and writes objects with its own service account, so the bucket doesn't need to be public. Ticked is also the safer default.

### 1.3 Check progress

Click **Check my progress** under Task 1 (30 points).

---

## Task 2: Create a Pub/Sub topic (30 points)

1. Navigation menu (☰) > **Pub/Sub** > **Topics**. If a Pub/Sub API prompt appears, click **Enable**.
2. Click **Create topic**.
3. In **Topic ID**, type the topic name from your lab page.
4. Leave **Add a default subscription** ticked.
5. Click **Create**.
6. Click **Check my progress** under Task 2 (30 points).

---

## Task 3: Create the Cloud Run function (40 points)

### 3.1 Start creating the function

1. Navigation menu (☰) > **Cloud Run**. Enable any required API if prompted.
2. Click **Write a function** (it may appear as **Create function** or under a **Functions** tab).
3. In **Service name** (or **Function name**), type the function name from your lab page.
4. In **Region**, pick the region from your lab page.
5. Under **Runtime**, choose **Node.js 22**.
6. Leave **Authentication** on its default.
7. Make sure the environment is **Cloud Run function (2nd gen)**, as the official page requires.

### 3.2 Add the Cloud Storage trigger

1. Click **Add trigger**, then **Cloud Storage trigger**.
2. For **Event type**, choose `google.cloud.storage.object.v1.finalized`.
3. For **Bucket**, click **Browse** and pick the bucket from Task 1.
4. Set the trigger **Region** to the same region.
5. Click **Save trigger**.

### 3.3 Accept the permission prompts

1. If a banner asks you to **Grant** roles to service accounts, click **Grant** (or **Grant all**).
2. If it asks to enable APIs, click **Enable all required APIs**.
3. Wait **2 to 3 minutes** before clicking **Create**, as the official page says, so permissions can spread.

### 3.4 Create the function and wait for the trigger

1. Click **Create**.
2. At the top of the page, under **Creating service**, wait until **Creating Eventarc trigger** shows a green check. The official page says not to change the code before that.

> **Mistake we made: the trigger failed even though the function was created.** Our status list showed **Creating service**, **Creating revision** and **Routing traffic** as completed, but **Creating Eventarc trigger** showed a red error:
>
> ```
> Failed. Details: Permission "storage.buckets.get" denied on "Bucket \"memories-bucket-...\" could not be validated.
> Please verify that the bucket exists and that the Eventarc service account has permission.
> ```
>
> The bucket did exist. The Eventarc service account just didn't have permission to read it yet, because the roles hadn't been granted or hadn't spread.
>
> **Fix, in order:**
> 1. Don't edit the code yet.
> 2. Grant the roles from Task 3.5 below.
> 3. Wait 2 to 3 minutes.
> 4. Recreate the trigger from the function's **Triggers** tab (Task 3.6).

### 3.5 Grant the roles from Cloud Shell

1. Click the **Activate Cloud Shell** icon (`>_`) at the top right. Click **Continue** and **Authorize** if asked.
2. Set two variables:

```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```

3. Check both printed:

```bash
echo $PROJECT_ID $PROJECT_NUMBER
```

4. Give the Eventarc service agent its role. This is the account named in the error. Note the `--condition=None` at the end, which the mistake below explains:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com" --role="roles/eventarc.serviceAgent" --condition=None
```

5. If step 4 says the service account doesn't exist, create it and run step 4 again:

```bash
gcloud beta services identity create --service=eventarc.googleapis.com
```

6. Give the default compute service account its event receiver role:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" --role="roles/eventarc.eventReceiver" --condition=None
```

7. Let Pub/Sub create tokens:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator" --condition=None
```

> **Mistake we made: the command stopped and asked for a condition.** Our first grant printed this and waited for an answer:
>
> ```
> [1] EXPRESSION=api.getAttribute('iam.googleapis.com/modifiedGrantsByRole', []).hasOnly([...]), TITLE=allow_only_curriculum_roles, DESCRIPTION=Restrict student IAM modifications to curriculum roles only
> [2] None
> [3] Specify a new condition
> The policy contains bindings with conditions, so specifying a condition is required when adding a binding. Please specify a condition.:
> ```
>
> The student account's permission to grant roles is limited by a rule (`allow_only_curriculum_roles`) to a short list of roles. Google asks which condition the **new** binding should carry.
>
> **Fix.** Type **2** and press **Enter** to choose **None**, which adds a normal binding. Or add `--condition=None` to the command so it never asks, as in the commands above. Don't choose option 1, which would attach the restriction to the new binding.

> **Note: Pub/Sub Publisher is not on the list.** The official page's note names four roles: Eventarc Service Agent, Eventarc Event Receiver, Service Account Token Creator and Pub/Sub Publisher. In our run the allowed list in the condition included only `eventarc.serviceAgent`, `eventarc.eventReceiver`, `cloudfunctions.serviceAgent` and `iam.serviceAccountTokenCreator`, so we skipped the `roles/pubsub.publisher` commands. The thumbnail was still created, because the code saves the thumbnail first and then publishes the message. If your **Grant** button in the Console adds Pub/Sub Publisher for you, that's fine too.
>
> If a command says permission denied, copy the exact message and check which role it names.

8. Wait **2 to 3 minutes** for the permissions to spread.

### 3.6 Recreate the trigger

1. Go to **Cloud Run** and click your function.
2. Open the **Triggers** tab.
3. Click **Add trigger**, then **Cloud Storage trigger**.
4. For **Event type**, choose `google.cloud.storage.object.v1.finalized`.
5. For **Bucket**, click **Browse** and pick your bucket from Task 1.
6. Set **Region** to your region.
7. For **Service account**, choose the default compute service account (it starts with your project number).
8. Click **Save trigger**.
9. Wait until the trigger shows a green check and appears in the **Triggers** tab, with the event type, your bucket and your function as the destination.

> **Note: the red error stays on screen.** After the trigger was recreated, the red message from the first attempt was still showing at the top of the page. It's left over from the earlier attempt. What matters is that the **Triggers** tab lists the trigger. You can also check from Cloud Shell:
>
> ```bash
> gcloud eventarc triggers list --location=us-central1
> ```
>
> Replace `us-central1` with your region.

### 3.7 Set the entry point and paste the code

1. Open the **Source** tab.

> **Mistake we made: no entry point field.** The **Source** tab showed only a message, "Start writing code and redeploy to finalize your service", and a **Start writing code** button. We couldn't find the **Function entry point** field the official page asks for.
>
> **Fix.** Click **Start writing code**. If it asks for a runtime, pick **Node.js 22**. The editor then opens, and the **Function entry point** field appears at the top next to **Save and redeploy**, pre-filled with the template name `helloGCS`.

2. Click inside the **Function entry point** field, select its content (Ctrl+A) and type the entry point from your lab page.
3. In the file list on the left, click `index.js`. Click inside the editor, press **Ctrl+A**, then **Delete**.
4. On your lab page, find the `index.js` code block in Task 3 step 3 and click its **copy icon**. This block has your function name and topic name already filled in.
5. Paste it into `index.js` with **Ctrl+V**.
6. Check that the name inside `functions.cloudEvent('...')` near the top matches the entry point from step 2, character for character.
7. Click `package.json` in the file list. Press **Ctrl+A**, then **Delete**.
8. On your lab page, copy the `package.json` block from Task 3 step 4 and paste it in.
9. Click **Save and redeploy**.
10. Wait for the deploy to finish. The status list should show **Building source**, **Updating service**, **Creating revision** and **Routing traffic** all completed.

> **Note: copy the code, don't retype it.** The official code uses your function and topic names. A single missing brace in `package.json` breaks the build, and a mismatch between the entry point and `functions.cloudEvent('...')` stops the function from starting.
>
> If the deploy fails with "permission denied... may take a few minutes", wait 3 minutes and click **Save and redeploy** again, as the official note says.

---

## Task 4: Test the infrastructure

### 4.1 Get an image

Download `https://storage.googleapis.com/cloud-training/arc101/travel.jpg` to your computer (right-click the image, then **Save image as**), or use any JPG or PNG.

> **Note: save the file, don't just open it.** Opening the image in a browser tab doesn't put a file on your computer. The upload dialog needs a file you've saved.

### 4.2 Open the right bucket

1. Navigation menu (☰) > **Cloud Storage** > **Buckets**.
2. Click your photo bucket, the one from Task 1 (it begins with your bucket name).

> **Mistake we made: looking for the Objects tab on the bucket list.** The page listed two buckets: ours and one beginning with `run-sources-`. The **Objects** tab doesn't exist on the list page, only inside a bucket.
>
> **Fix.** Click the blue link of your photo bucket. Don't use the `run-sources-...` bucket, which Cloud Run created for the function's source code. Inside your bucket, **Objects** is the first tab.

### 4.3 Upload and verify

1. On the **Objects** tab, click **Upload files**.
2. Choose `travel.jpg` and click **Open**. Upload it **after** the function has deployed, because only new uploads trigger it.
3. Wait about 30 seconds, then click **Refresh** (the circular arrow above the object list).
4. You should see two files: `travel.jpg` (about 909 KB) and `travel_64x64_thumbnail.jpg` (about 1.4 KB). In our run the thumbnail appeared one second after the original.

### 4.4 Check progress

Click **Check my progress** under Task 4 (this is the 40-point checkpoint). The official page says you don't need to wait for the thumbnail before clicking.

---

## Finish

1. Confirm all three checkpoints are green and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the names and region against your lab page.

### Quick fixes for other errors you might see

| What you see | Likely cause | Fix |
|---|---|---|
| Bucket details show Location type **Zone** | Chose Zone in the form | Delete the bucket and recreate it with **Region** |
| `Permission "storage.buckets.get" denied` on the Eventarc trigger | Eventarc service account lacks a role, or roles haven't spread | Grant the roles (Task 3.5), wait a few minutes, recreate the trigger |
| Command asks "Please specify a condition" | The student IAM condition | Type `2` for **None**, or add `--condition=None` |
| Cloud Shell says the Eventarc service account doesn't exist | The service identity wasn't created yet | Run `gcloud beta services identity create --service=eventarc.googleapis.com` and retry |
| No **Function entry point** field | The Source tab is in its empty state | Click **Start writing code** |
| Deploy fails with a package error | `package.json` pasted incompletely | Copy it again from the lab page |
| Deploy says "permission denied... may take a few minutes" | Permissions still spreading | Wait 3 minutes and click **Save and redeploy** again |
| Red trigger error still shows after recreating it | Message left from the first attempt | Check the **Triggers** tab or `gcloud eventarc triggers list` instead |
| No thumbnail appears | Trigger missing, wrong entry point, or image uploaded before the deploy | Check the **Triggers** tab, check the entry point matches `functions.cloudEvent('...')`, then upload a new image with a different name |
| Can't find the Objects tab | You're on the bucket list, or in the `run-sources-` bucket | Open your own photo bucket |
| Checkpoint red though the thumbnail exists | Checker hasn't caught up, or a name mismatch | Wait 20 seconds and click again, then compare names with the lab page |

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, requirements and checkpoints. It adds the warnings and fixes below.

| Official lab | This guide |
|---|---|
| Task 1: create a bucket for the photographs, in the lab's region | Same. Adds the **Region** (not Zone) choice and the public access prevention dialog. |
| Task 2: create a Pub/Sub topic for the function to send messages | Same. Adds the exact clicks. |
| Task 3: create a Cloud Run function (2nd gen) using Node.js 22 with a Cloud Storage trigger, enable the APIs and grant roles as requested, wait a few minutes, and wait for the green check on the Eventarc trigger | Same. Adds the permission commands, the IAM condition fix and how to recreate a failed trigger. |
| Task 3: set the entry point and add the given `index.js` and `package.json` | Same. Adds the **Start writing code** step that reveals the entry point field, and the copy-don't-retype rule. |
| Task 3 note: if permission is denied, wait a few minutes, and make sure the Eventarc, Token Creator and Pub/Sub roles are assigned | Same. Notes that the student account's condition only allowed some of those roles in our run. |
| Task 4: upload a JPG or PNG, refresh, and see a thumbnail; the trigger can be recreated if missing | Same. Adds finding the right bucket and the thumbnail's expected size and name. |

> **What the official page doesn't show.** The public version of the page leaves your bucket, topic, function names and region blank until your lab starts. It also doesn't say that the Console's **Source** tab hides the entry point until you click **Start writing code**, or that a student account may need `--condition=None` when granting roles. The values in this guide are the ones from our run. Copy yours from your own lab page.
