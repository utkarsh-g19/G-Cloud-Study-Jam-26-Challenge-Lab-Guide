# ARC131: Using the Google Cloud Speech API (Challenge Lab)

A complete walkthrough with the mistakes we made and how we fixed them.

| | |
|---|---|
| **Lab ID** | ARC131 |
| **Official title** | Using the Google Cloud Speech API: Challenge Lab |
| **Scoring** | 100 points across 3 checkpoints |
| **Time** | 45 minutes of access, about 10 to 15 minutes of actual work |
| **Official page version used** | Live lab page, manual and tests last updated October 20, 2026 |
| **Where you work** | The Google Cloud console and Cloud Shell (no VM, no notebook) |

| Checkpoint | Points |
|---|---|
| Task 1: Create an API key | 30 |
| Task 2: Speech-to-Text transcription in English | 30 |
| Task 3: Speech-to-Text transcription in Spanish | 40 |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake we made** show a real error from our run and the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values written in `UPPERCASE` are placeholders.** Your lab assigns its own project ID, bucket and key, and shows the exact file names inside your running lab. The public version of the lab page leaves some of these blank. In our run the file names were `request.json`, `speech_response_en.json`, `request_sp.json` and `response_speech_sp.json`. **Always copy the names from your own lab page.**

---

# Part 1: Conceptual

## What this lab is about

You are given two pre-recorded audio files stored in Google Cloud Storage: one English question and one Spanish clip. Your job is to turn each one into text using the **Cloud Speech-to-Text API**, without being shown the exact commands. In order, you:

1. Create an **API key** so you can call the API from the command line.
2. Write a small **request file** (JSON) that tells the API which audio to read and what language it is in.
3. Send that request with `curl` and save the **response** (the transcript) to a file.
4. Upload both the request and the response files to a **Cloud Storage bucket**, which is how the lab checks your work.
5. Repeat the same process for Spanish, changing the audio file, the encoding and the language.

Challenge labs give you the goal but not the commands, so you apply what the course taught.

## Important concepts, explained simply

**Speech-to-Text API.** A Google service that listens to an audio file and returns the words as text. This lab uses the `speech:recognize` method, which is meant for short audio (about a minute or less) and returns the result in one reply.

**REST API and `curl`.** Many Google services are used by sending a web request to an address. `curl` is a command-line tool that sends that request and prints the reply.

**API key.** A long string (it starts with `AIza`) that tells Google which project is making the request. It's the simplest way to authenticate, which is why the lab uses it. Treat it like a password and never share it in screenshots.

**API restrictions.** A setting on the key that limits which APIs it can call. For this lab, the key only needs to call the Speech-to-Text API.

**Cloud Shell.** A terminal that runs in your browser with `gcloud` already installed and logged in.

**Environment variable and `export`.** A named value the terminal remembers, like `API_KEY`. `export API_KEY="..."` stores it and `$API_KEY` uses it. **It disappears if Cloud Shell restarts**, so you may need to set it again.

**Request JSON.** The file that describes what you want transcribed. It has two parts:
- `config`: how to read the audio (`encoding`, `languageCode`, and for stereo audio `audioChannelCount`).
- `audio`: where the audio is (`uri`).

**Encoding.** The audio file's format. A `.wav` file uses `LINEAR16` and a `.flac` file uses `FLAC`. If you pick the wrong one, you get an error or an empty result.

**Language code.** A short tag such as `en-US` (US English) or `es-ES` (Spanish). It tells the API which language to listen for.

**Audio channels.** Stereo audio has two channels. The English file is stereo, so the request sets `audioChannelCount` to `2`.

**Cloud Storage and `gs://` addresses.** Google's file storage. A file's address looks like `gs://bucket-name/file-name`. A *bucket* is the top-level container that holds files.

**The response.** The API replies with JSON containing `results`, then `alternatives`, then a `transcript` (the text) and a `confidence` number.

**Single-line `echo` vs a heredoc.** A *heredoc* (`cat > file <<'EOF' ... EOF`) writes several lines into a file, but only works if the closing `EOF` sits alone at the very start of its line. If you paste it with indentation, the terminal never sees the end and waits forever. Writing the JSON in one `echo` line avoids this (see the first mistake below).

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. **Write down from your lab page** the file names it asks for: the English request file, the English response file, the Spanish request file and the Spanish response file.
5. Don't click **End Lab** until all three checkpoints are green.

> **Note: a new project each lab.** Every lab creates a new project, so an API key from a previous lab does not carry over. Create a fresh one in this lab.

---

## Task 1: Create an API key (30 points)

### 1.1 Open the Credentials page

1. Click the **Navigation menu** (☰) at the top left.
2. Click **APIs & Services**, then **Credentials**.
3. Check that the project selector at the top shows your `qwiklabs-gcp-...` project.

### 1.2 Create the key

1. Click **+ Create credentials** at the top, then **API key**.
2. Leave **Name** as it is.
3. Click the **Select API restrictions** dropdown.
4. Click the filter box in the dropdown and type `speech`.
5. Tick **Cloud Speech-to-Text API**.
6. Click **OK**.
7. Leave **Application restrictions** on **None**.
8. Click **Create**.

> **Note: the form won't submit without an API choice.** The current Console shows a red "API selection required" message under **Select API restrictions** until you tick at least one API. That is the field to fix if **Create** doesn't work.
>
> The blue "Service account required" box is about the Agent Platform and Gemini APIs. **Ignore it.** Don't tick those APIs and don't bind the key to a service account.

> **Mistake we made (seen in the sibling lab ARC132, which uses the same form): scrolling the API list.** The dropdown list is alphabetical and starts with names like "Actions API" and "Analytics Hub API", so the API we wanted wasn't visible and we didn't know which box to tick.
>
> **Fix.** Don't scroll. Click the **Type to filter** box at the top of the dropdown and type `speech`, then tick **Cloud Speech-to-Text API**.

> **If Cloud Speech-to-Text API isn't in the list,** it isn't enabled in this project yet. Close the form, go to **APIs & Services > Library**, search for `Cloud Speech-to-Text API`, open it and click **Enable**, then redo steps 1.2.1 to 1.2.8.

### 1.3 Copy the key

1. In the "API key created" dialog, click the copy icon next to the key (it starts with `AIza`).
2. Click **Close**.

> **Note: if you closed the dialog before copying,** go to **APIs & Services > Credentials**, find your key under **API Keys**, and click **Show key**.
>
> **Don't** paste your real key into screenshots, chats or documents you share.

### 1.4 Open Cloud Shell and save the key

1. Click the **Activate Cloud Shell** icon (`>_`) at the top right.
2. Click **Continue** and **Authorize** if asked. Wait for the `$` prompt.
3. Type this, pasting your key between the quotes, and press Enter:

```bash
export API_KEY="PASTE_KEY_HERE"
```

4. Check it saved:

```bash
echo $API_KEY
```

It must print your key. If it prints a blank line, repeat step 3.

### 1.5 Check progress

Click **Check my progress** under Task 1 (30 points).

---

## Task 2: Transcribe the English audio (30 points)

The audio is `gs://spls/arc131/question_en.wav`.

### 2.1 Set the project and bucket variables

Run the two lines from your lab page:

```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)
export BUCKET_NAME="${GOOGLE_CLOUD_PROJECT}-speech"
```

### 2.2 Check them and check the bucket exists

```bash
echo $GOOGLE_CLOUD_PROJECT
echo $BUCKET_NAME
```

The second should print your project ID followed by `-speech`. Then:

```bash
gcloud storage ls gs://$BUCKET_NAME
```

An empty result with no error is fine. If it says the bucket doesn't exist, go to **Cloud Storage > Buckets** and look for one ending in `-speech`. The lab setup should have created it.

### 2.3 Check your key is still set

```bash
echo $API_KEY
```

If it's blank, run `export API_KEY="..."` again.

### 2.4 Create the English request file

The official page tells you to create a file named `request.json` that transcribes `gs://spls/arc131/question_en.wav` to English, but it doesn't give you the JSON. Run this **single line**:

```bash
echo '{"config":{"encoding":"LINEAR16","languageCode":"en-US","audioChannelCount":2},"audio":{"uri":"gs://spls/arc131/question_en.wav"}}' > request.json
```

What the fields mean:
- `encoding` is `LINEAR16` because the file is a `.wav`.
- `languageCode` is `en-US` because the speech is English.
- `audioChannelCount` is `2` because the file is stereo.
- `uri` is the audio file's address from the lab page.

> **Mistake we made: the pasted file never finished.** We first used the multi-line form:
>
> ```
> cat > request.json <<'EOF'
>   { ...several indented lines... }
>   EOF
> ```
>
> The terminal kept showing `>` prompts and never returned to `$`. The lines we pasted were **indented, including the closing `EOF`**, and a heredoc only ends when `EOF` is alone at the very start of its line.
>
> **Fix, in order:**
> 1. Click inside the terminal and press **Ctrl+C**. You should get the `$` prompt back.
> 2. **Don't** type `EOF` now. That would save the indented `EOF` into the file and break the JSON.
> 3. Create the file with the single-line `echo` command above instead. Indentation can't break it.
> 4. Check it with `cat request.json` and `python3 -m json.tool request.json` (next step).

### 2.5 Check the file

```bash
cat request.json
```

It should show the JSON on one line. Then check it's valid:

```bash
python3 -m json.tool request.json
```

It should print the neatly formatted JSON with no error. If it prints an error, the quotes or braces are wrong, so run the `echo` line again.

### 2.6 Call the API

Run the command from your lab page. It uses your key and saves the result in `speech_response_en.json`:

```bash
curl "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > speech_response_en.json
```

> **Mistake we almost made: wrong file names.** The public version of the lab page leaves the file names blank, and older guides on the internet use different ones (for example `speech_request_sp.json`). We only found the real names, `request.json` and `speech_response_en.json`, by reading the lab page inside our running lab.
>
> **Fix.** Always take the names from **your own lab page**. The checker looks for those exact names.

### 2.7 View the result

```bash
cat speech_response_en.json
```

You want `results` containing a `transcript` of an English question. If you see `"error"`, use the fixes at the end of this guide.

### 2.8 Upload both files to the bucket

```bash
gcloud storage cp request.json gs://${BUCKET_NAME}/request.json
gcloud storage cp speech_response_en.json gs://${BUCKET_NAME}/speech_response_en.json
```

### 2.9 Confirm the upload

```bash
gcloud storage ls gs://$BUCKET_NAME
```

Both files should be listed.

### 2.10 Check progress

Click **Check my progress** under Task 2 (30 points).

---

## Task 3: Transcribe the Spanish audio (40 points)

The audio is `gs://spls/arc131/multi_es.flac`. Use the **same Cloud Shell**, so your variables are still set.

### 3.1 Check the variables

```bash
echo $API_KEY
echo $BUCKET_NAME
```

Both should print a value. If either is blank, set it again with the `export` lines from Task 1.4 and Task 2.1.

### 3.2 Create the Spanish request file

The official page names the files for this task as `request_sp.json` (request) and `response_speech_sp.json` (response). Run this single line:

```bash
echo '{"config":{"encoding":"FLAC","languageCode":"es-ES"},"audio":{"uri":"gs://spls/arc131/multi_es.flac"}}' > request_sp.json
```

What changed from English: the encoding is `FLAC` (the file is `.flac`), the language is `es-ES`, and the channel count line is gone.

> **Note: the names don't follow one pattern.** The English files are `request.json` and `speech_response_en.json`, but the Spanish ones are `request_sp.json` and `response_speech_sp.json`. It's easy to type `speech_response_sp.json` by habit. Copy the names from your lab page.
>
> **Don't reuse `request.json` for Spanish.** It would overwrite your English request, and the file you uploaded in Task 2 would no longer match what's on your machine.

### 3.3 Check the file

```bash
cat request_sp.json
python3 -m json.tool request_sp.json
```

The second command should print the formatted JSON with no error.

### 3.4 Call the API

```bash
curl "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request_sp.json > response_speech_sp.json
```

If the lab page's command shows different text below the first line, use the lab page's version.

### 3.5 View the result

```bash
cat response_speech_sp.json
```

You want `results` with a Spanish `transcript`. If you get `{}` or an error, see the fixes below.

### 3.6 Upload both files

```bash
gcloud storage cp request_sp.json gs://${BUCKET_NAME}/request_sp.json
gcloud storage cp response_speech_sp.json gs://${BUCKET_NAME}/response_speech_sp.json
```

### 3.7 Confirm the bucket

```bash
gcloud storage ls gs://$BUCKET_NAME
```

You should see four files: `request.json`, `speech_response_en.json`, `request_sp.json` and `response_speech_sp.json`.

### 3.8 Check progress

Click **Check my progress** under Task 3 (40 points).

---

## Finish

1. Confirm all three checkpoints are green and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the file names in the bucket against your lab page, letter by letter.

### Quick fixes for other errors you might see

| What you see | Likely cause | Fix |
|---|---|---|
| Terminal shows `>` and never returns to `$` | Pasted heredoc with an indented `EOF` | Press Ctrl+C and use the single-line `echo` command |
| `"API key not valid"` | `API_KEY` is empty or mistyped, or the key is brand new | Run `echo $API_KEY`, re-export it, and wait a minute if the key was just created |
| `"API has not been used... or it is disabled"` | Speech-to-Text API not enabled | **APIs & Services > Library**, enable **Cloud Speech-to-Text API**, wait a minute, retry |
| `API_KEY_SERVICE_BLOCKED` | The key's API restrictions don't include Speech-to-Text | **Credentials**, open the key, tick **Cloud Speech-to-Text API**, save |
| English call errors about channel count or sample rate | `audioChannelCount` doesn't match the file | Keep it at `2`, and if the error still mentions it, try `1` |
| Response is `{}` or has no `results` | Wrong `encoding` or `languageCode` | English must be `LINEAR16` with `en-US`, Spanish `FLAC` with `es-ES` |
| Variables are blank after Cloud Shell reconnects | Variables don't survive a restart | Re-run the `export` lines from Task 1.4 and Task 2.1 |
| Upload says bucket not found | Wrong or empty `BUCKET_NAME` | `echo $BUCKET_NAME` and compare with the bucket in **Cloud Storage > Buckets** |
| Checkpoint red though output looks right | File name mismatch in the bucket | Run `gcloud storage ls gs://$BUCKET_NAME` and compare names with the lab page |

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, file names and checkpoints. It adds the warnings and fixes below.

| Official lab | This guide |
|---|---|
| Task 1: create an API key in the console and export it as `API_KEY` in Cloud Shell | Same steps. Adds the API-restrictions field, the filter-box tip, the "ignore the service account box" note and the echo check. |
| Task 2 step 1: set `GOOGLE_CLOUD_PROJECT` and `BUCKET_NAME` | Same two lines. Adds the checks that the values and the bucket exist. |
| Task 2: create `request.json` to transcribe the English `.wav` | Same file. The official page doesn't give the JSON, so this guide supplies it, explains each field, and writes it as one line to avoid the heredoc problem. |
| Task 2: call the API with `curl` and store the result in `speech_response_en.json` | Same command. Adds a result check and what a good or bad response looks like. |
| Task 2: upload the files to the bucket | Same goal. Adds the `gcloud storage ls` check to confirm both files arrived. |
| Task 3: create the Spanish request file, call the API, store `response_speech_sp.json` | Same steps. Adds the `FLAC` and `es-ES` change, the naming-pattern warning and the "don't reuse `request.json`" rule. |

> **What the official page doesn't show.** The public version leaves the file names blank, and doesn't give the request JSON, because the course labs are meant to have taught it. The values here (`LINEAR16` with `audioChannelCount: 2` for the English `.wav`, and `FLAC` with `es-ES` for the Spanish `.flac`) are the standard settings for those two files. If the API ever complains, check them against the error message.
>
> **Older versions of this lab.** Some public pages and third-party guides describe an older version of ARC131 that had you connect over SSH to a VM named `lab-vm`, and used file names such as `speech_request_sp.json`. The current version uses Cloud Shell and uploads the files to a bucket, with the file names shown on your own lab page. If a guide tells you to SSH into a VM, it's out of date for this lab.
