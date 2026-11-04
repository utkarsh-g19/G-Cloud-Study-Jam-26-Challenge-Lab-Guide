# ARC122: Analyze Images with the Cloud Vision API (Challenge Lab)

A complete walkthrough with the mistakes we made and how we fixed them.

| | |
|---|---|
| **Lab ID** | ARC122 |
| **Official title** | Analyze Images with the Cloud Vision API: Challenge Lab |
| **Scoring** | 100 points across 3 checkpoints |
| **Time** | 45 minutes of access. The page estimates 10 minutes of work, and our file-name mistakes added some time |
| **Official page version used** | Manual last updated May 20, 2026 |
| **Where you work** | The Google Cloud console and Cloud Shell (no VM, no notebook) |

| Checkpoint | Points |
|---|---|
| Task 1: Create an API key (and verify your resources) | 20 |
| Task 2 and Task 3: Create and update the JSON file (text detection) | 40 |
| Task 3: Update the JSON file to use `LANDMARK_DETECTION` | 40 |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake we made** show a real error from our run and the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values written in `UPPERCASE` are placeholders.** Your lab assigns its own bucket name, and the public version of the lab page leaves the upload bucket blank until the lab starts. In our run the image was `manif-des-sans-papiers.jpg`, and the bucket ended in `-bucket`. Yours will differ, so **always copy the names from your own lab page and your own bucket.**

---

# Part 1: Conceptual

## What this lab is about

You are a junior machine learning engineer. A bucket already holds one photo, and your job is to ask Google's **Cloud Vision API** two questions about it: "what text is written in this picture?" and "is there a famous landmark in it?". You do it with plain web requests and no code. In order, you:

1. Verify your resources: an API key, a bucket with an image, and that image set to public.
2. Write a small request file (`request.json`) that points at the image.
3. Send it to the Vision API for **text detection** and save the answer to a file.
4. Upload that answer to Cloud Storage so the lab can check it.
5. Change the request to **landmark detection** and repeat.

Challenge labs give you the goal but not the commands, so you apply what the course taught.

## Important concepts, explained simply

**Cloud Vision API.** A Google service that looks at an image and tells you what's in it. It can read text, find landmarks, detect faces and more. Which job it does depends on the `type` you ask for.

**`TEXT_DETECTION`.** Reads text in a photo (OCR). The reply contains `textAnnotations`.

**`LANDMARK_DETECTION`.** Recognises famous places and buildings. The reply contains `landmarkAnnotations`. If nothing famous is recognised, the answer comes back nearly empty, which is not an error.

**REST API and `curl`.** You use the service by sending a web request to an address. `curl` is a command-line tool that sends that request and prints the reply.

**`images:annotate`.** The Vision API's one main address. You send it a request body and it replies with the analysis.

**Request JSON.** The file that says what to analyze. It has an `image` (where it is) and `features` (what to look for, and how many results at most).

**`gcsImageUri`.** The image's address in Cloud Storage, written as `gs://BUCKET_NAME/FILE_NAME`. The **file name includes its extension**, for example `.jpg`.

**API key.** A long string (it starts with `AIza`) that tells Google which project is calling. It's a simple way to authenticate, which is why the lab uses it. Treat it like a password.

**Public object.** A request made with just an API key has no signed-in user, so the Vision API can only read the image if the image is **publicly readable**. That's why the lab makes you set it up. You do it by giving `allUsers` the **Storage Object Viewer** role.

**Public access prevention.** A bucket setting that blocks public access completely. If it's enforced, you can't make the image public until you turn it off.

**Environment variable and `export`.** A named value the terminal remembers, like `API_KEY`. `export API_KEY="..."` stores it and `$API_KEY` uses it. **Variables disappear if Cloud Shell restarts.**

**`printf` with `%s`.** A command that fills values into a template. Each `%s` is replaced, in order, by the values listed after the template. We use it to write the request file without typing the bucket and image names by hand.

**`grep -c`.** Counts how many lines contain a word. We use it to see quickly whether a response contains `textAnnotations` or `"error"`.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. **Write down from your lab page:** the upload command under Task 3 (`gcloud storage cp text-response.json gs://...`). The name after `gs://` is your **upload bucket**.
5. Don't click **End Lab** until all three checkpoints are green.

> **Note: a new project each lab.** Every lab creates a new project, so an API key from another lab doesn't carry over.

---

## Task 1: Verify your resources (20 points)

The official page asks for three things: an API key saved as `API_KEY`, a bucket ending in `-bucket` with an image already inside, and that image set to public.

### 1.1 Check the Vision API is enabled

The official page's standards say any needed APIs must be enabled.

1. Click the **Navigation menu** (☰) > **APIs & Services** > **Library**.
2. Search for `Cloud Vision API` and open it.
3. If you see **Manage**, it's already enabled. If you see **Enable**, click it and wait for it to finish.

### 1.2 Create the API key

1. Navigation menu (☰) > **APIs & Services** > **Credentials**.
2. Click **+ Create credentials** > **API key**.
3. Leave **Name** as it is.
4. Click the **Select API restrictions** dropdown.
5. Click the filter box, type `vision`, and tick **Cloud Vision API**.
6. Click **OK**.
7. Leave **Application restrictions** on **None**.
8. Click **Create**.
9. Click the copy icon next to the key (it starts with `AIza`), then **Close**.

> **Note: the form won't submit without an API choice.** The current Console shows a red "API selection required" message until you tick at least one API. The blue "Service account required" box is about Agent Platform and Gemini APIs, so **ignore it**.
>
> If you closed the dialog before copying the key, go to **Credentials**, find it under **API Keys** and click **Show key**. Never paste your real key into screenshots or chats you share.

### 1.3 Save the key in Cloud Shell

1. Click the **Activate Cloud Shell** icon (`>_`) at the top right. Click **Continue** and **Authorize** if asked.
2. Run this (paste your key between the quotes):

```bash
export API_KEY="PASTE_KEY_HERE"
```

3. Check it:

```bash
echo $API_KEY
```

It must print your key.

### 1.4 Find the bucket and the image

```bash
gcloud storage ls
```

Find the bucket ending in `-bucket`. Then list what's inside:

```bash
gcloud storage ls gs://BUCKET_NAME
```

You'll see one image file. The image's name is the part after the last `/`, **including its extension**.

Save three values (bucket name only, no `gs://`, no trailing slash; the image file name with its extension; and the upload bucket from Step 0):

```bash
export IMG_BUCKET="PASTE_BUCKET_NAME"
export IMG_NAME="PASTE_IMAGE_FILE_NAME_WITH_EXTENSION"
export UPLOAD_BUCKET="PASTE_UPLOAD_BUCKET_NAME"
```

Check them, and check the image exists at exactly that path:

```bash
echo $IMG_BUCKET $IMG_NAME $UPLOAD_BUCKET
gcloud storage ls gs://$IMG_BUCKET/$IMG_NAME
```

The second command should print exactly one line: the image's path. If it says "No URLs matched", one of your values is wrong.

> **Mistake we made: a wrong or empty image name.** We had two separate problems here:
>
> 1. **`IMG_NAME` was empty.** The request file we built contained `gs://qwiklabs-gcp-.../-bucket/` with nothing after the last slash, so it didn't point to any image.
> 2. **`IMG_NAME` was missing `.jpg`.** After fixing the first problem, we typed `manif-des-sans-papiers` instead of `manif-des-sans-papiers.jpg`, so the path was still wrong.
>
> **Fix.** Read the file name from `gcloud storage ls gs://$IMG_BUCKET` and copy it exactly, extension included. Then run the check above with `$IMG_NAME` in it, which proves the path exists **before** you build the request.

> **Mistake we made: putting the image name in the bucket variable.** When it came to upload, we had set `UPLOAD_BUCKET` to the image's file name (`manif-des-sans-papiers.jpg`) instead of the bucket name.
>
> **Fix.** `UPLOAD_BUCKET` is a **bucket name only**. In our run it was the same bucket that held the image, ending in `-bucket`. Check yours against the upload command on your lab page.

### 1.5 Make the image public

1. In the Console, go to **Navigation menu** (☰) > **Cloud Storage** > **Buckets**.
2. Click your bucket (the one ending in `-bucket`).
3. Click the **Permissions** tab.
4. Click **Grant access**.
5. In **New principals**, type `allUsers`.
6. In **Select a role**, type `Storage Object Viewer` and pick **Storage Object Viewer** (under Cloud Storage).
7. Click **Save**. If a warning about making the bucket public appears, click **Allow public access**.

If the Console blocks this, try the command-line version:

```bash
gcloud storage buckets add-iam-policy-binding gs://$IMG_BUCKET --member=allUsers --role=roles/storage.objectViewer
```

Test that the image is public. This should print `HTTP/2 200`:

```bash
curl -sI https://storage.googleapis.com/$IMG_BUCKET/$IMG_NAME | head -n 1
```

> **Note: if you see `403`,** the image isn't public yet, so redo the steps above. If **Grant access** complains about public access prevention, look on the **Permissions** tab for a setting that says public access prevention is on, turn it off, and try again.

### 1.6 Check progress

Go to the lab page and click **Check my progress** under Task 1 (20 points).

---

## Task 2: Create the request file

The official page tells you to create a file named `request.json` and shows a template with blanks:

```
{ "requests": [ { "image": { "source": { "gcsImageUri": } }, "features": [ { "type": "maxResults": 10 } ] } ] }
```

As written, that's not valid JSON. You have to fill `gcsImageUri` with your image's address and `type` with the detection method. This command does that in one go, with `TEXT_DETECTION` first:

```bash
printf '{"requests":[{"image":{"source":{"gcsImageUri":"gs://%s/%s"}},"features":[{"type":"%s","maxResults":10}]}]}\n' "$IMG_BUCKET" "$IMG_NAME" TEXT_DETECTION > request.json
```

Check the content. The path must end in your image's file name and `type` must say `TEXT_DETECTION`:

```bash
cat request.json
```

Then check it's valid JSON:

```bash
python3 -m json.tool request.json
```

It should print the formatted JSON with no error.

> **Note: why one `printf` command.** It builds the file from your variables, so a typo in the bucket or image name is impossible to make while typing. If the path in `cat request.json` looks wrong, the variable is wrong, so fix the variable and run the command again.

---

## Task 3: Text detection, then landmark detection

### 3.1 Call the Vision API for text

The official page gives this command with `YOUR_JSON` as a placeholder. Replace `YOUR_JSON` with `request.json`:

```bash
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json https://vision.googleapis.com/v1/images:annotate?key=${API_KEY} -o text-response.json
```

> **Mistake we made: pasting the page's command without replacing the placeholder.** We pasted the command exactly as shown on the lab page, with `@YOUR_JSON` still in it. Cloud Shell replied:
>
> ```
> curl: option --data-binary: error encountered when reading a file
> curl: try 'curl --help' or 'curl --manual' for more information
> ```
>
> The words "error encountered when reading a file" give it away: curl looked for a file literally called `YOUR_JSON`, which doesn't exist.
>
> **Fix.** Change `@YOUR_JSON` to `@request.json` (the name of the file you created) and run it again. The commands in this guide already have this done.

### 3.2 Check the response before you upload it

```bash
grep -c textAnnotations text-response.json
grep -c '"error"' text-response.json
```

The first must print `1` or higher, and the second must print `0`. Then look at it:

```bash
cat text-response.json
```

You want `textAnnotations` with text found in the image.

> **Mistake we made: uploading the error response.** We uploaded `text-response.json` **before** checking what it contained. Because the request pointed at a wrong image path, the file was a small error message (187 bytes), not a real result. The lab accepted the upload but **Check my progress** stayed red, and the `grep` check showed `0` for `textAnnotations` and `1` for `"error"`.
>
> **Fix, in order:**
> 1. Correct `IMG_NAME` (see the Task 1.4 mistake) and run the `gcloud storage ls gs://$IMG_BUCKET/$IMG_NAME` check.
> 2. Rebuild `request.json` with the `printf` command from Task 2.
> 3. Run the curl command again, and check that `grep` now shows `1` for `textAnnotations` and `0` for `"error"`.
> 4. Upload again. The new file **overwrites** the bad one.

### 3.3 Upload the text result

```bash
gcloud storage cp text-response.json gs://$UPLOAD_BUCKET
```

Check that it landed, and that it's much bigger than the 187-byte error file:

```bash
gcloud storage ls -l gs://$UPLOAD_BUCKET
```

> **Mistake we made: upload failed with a 404.** With `UPLOAD_BUCKET` wrongly set to the image's file name, the upload said:
>
> ```
> ERROR: gs://manif-des-sans-papiers.jpg not found: 404.
> ```
>
> **Fix.** Reset the variable to the **bucket** name and upload again:
>
> ```bash
> export UPLOAD_BUCKET="PASTE_UPLOAD_BUCKET_NAME"
> gcloud storage cp text-response.json gs://$UPLOAD_BUCKET
> ```

### 3.4 Check progress for text detection

Go to the lab page, wait about 15 seconds, and click **Check my progress** under the first part of Task 3 (40 points).

> **Note: what we saw once it was right.** After we fixed the image name, rebuilt the request and re-uploaded a real response, the checkpoint turned green and showed "Assessment Completed!". If yours stays red, the usual cause is a file in the bucket that isn't a real Vision response, so check its size and content.

### 3.5 Switch to landmark detection

Rewrite `request.json` with the other method, using the same command and a different last value:

```bash
printf '{"requests":[{"image":{"source":{"gcsImageUri":"gs://%s/%s"}},"features":[{"type":"%s","maxResults":10}]}]}\n' "$IMG_BUCKET" "$IMG_NAME" LANDMARK_DETECTION > request.json
```

Check it. The path must still end in your image's file name and `type` must now say `LANDMARK_DETECTION`:

```bash
cat request.json
```

### 3.6 Call the API for landmarks

```bash
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json https://vision.googleapis.com/v1/images:annotate?key=${API_KEY} -o landmark-response.json
```

Check that it isn't an error:

```bash
grep -c '"error"' landmark-response.json
```

It should print `0`. Then view it:

```bash
cat landmark-response.json
```

If a landmark is recognised, you'll see `landmarkAnnotations`. If you only see `{ "responses": [ {} ] }`, no landmark was recognised, and the official page says the upload is for activity-tracking validation, so you can upload it either way.

### 3.7 Upload the landmark result

```bash
gcloud storage cp landmark-response.json gs://$UPLOAD_BUCKET
gcloud storage ls -l gs://$UPLOAD_BUCKET
```

You should see both `text-response.json` and `landmark-response.json`.

### 3.8 Check progress for landmark detection

Wait about 15 seconds and click **Check my progress** under the landmark part of Task 3 (40 points).

---

## Finish

1. Confirm all three checkpoints are green and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the file names and sizes in the upload bucket against your lab page.

### Quick fixes for other errors you might see

| What you see | Likely cause | Fix |
|---|---|---|
| `curl: option --data-binary: error encountered when reading a file` | `@YOUR_JSON` left in the command | Use `@request.json` |
| `request.json` path ends with `-bucket/` | `IMG_NAME` is empty | Re-export `IMG_NAME`, then rebuild the file |
| Response contains `"error"` and no `textAnnotations` | Wrong image path, wrong key, or image not public | Run `gcloud storage ls gs://$IMG_BUCKET/$IMG_NAME` and the public check, then redo the call |
| `"Failed to get image"` or `403` on the public check | Image isn't public yet | Redo Task 1.5 |
| `"API key not valid"` | `API_KEY` is empty or mistyped, or the key is new | `echo $API_KEY`, re-export it, wait a minute |
| `"Cloud Vision API has not been used... or it is disabled"` | API not enabled | Enable it as in Task 1.1, wait a minute |
| `API_KEY_SERVICE_BLOCKED` | Key not allowed to call Vision | **Credentials**, open the key, tick **Cloud Vision API**, save |
| Upload says `404 not found` | `UPLOAD_BUCKET` holds a file name, not a bucket name | Reset it to the bucket name |
| Variables are blank after Cloud Shell reconnects | Variables don't survive a restart | Re-run the `export` lines from Task 1.3 and Task 1.4 |
| Checkpoint red but output looks right | Bad or small file in the bucket | `gcloud storage ls -l gs://$UPLOAD_BUCKET`, check the sizes, and re-upload real responses |

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, commands and checkpoints. It adds the warnings and fixes below.

| Official lab | This guide |
|---|---|
| Task 1: have an API key saved as `API_KEY`, a bucket ending in `-bucket` with an image in it, and the image publicly accessible | Same three things. Adds the console clicks for the key, the way to find the image name, the public-access steps and the `curl -sI` test. |
| Standards: enable any needed APIs and create all resources in the region | Adds the **APIs & Services > Library** check for Cloud Vision. The lab doesn't have you create resources, so there is no region step. |
| Task 2: create `request.json` from a template with blanks | Same file. Fills the blanks with a `printf` command built from your variables, and adds a JSON validity check. |
| Task 3: fill in the bucket/image and `TEXT_DETECTION`, and call the API with `curl`, replacing `YOUR_JSON` | Same command, with `YOUR_JSON` already replaced by `request.json`. Adds `grep` checks on the response before uploading. |
| Task 3: upload `text-response.json` with `gcloud storage cp` to the bucket shown on the page | Same command. Adds the `-l` listing to confirm the file size and the warning about putting a bucket name, not a file name, in the variable. |
| Task 3: switch to `LANDMARK_DETECTION`, call again, save to `landmark-response.json` and upload | Same steps. Adds the note that an almost-empty landmark response isn't an error. |

> **What the official page doesn't show.** The public version of the page leaves the upload bucket blank until your lab starts, and the `request.json` template has blanks (it isn't valid JSON as shown). It also doesn't tell you the image must keep its file extension in the path, or how to find the image's name. Some older copies of the page and third-party guides use `gsutil cp` for the upload where the current page uses `gcloud storage cp`. They do the same job, so follow the version on your own lab page.
