# ARC132: Implement Speech and Language Solutions with Pre-trained APIs (Challenge Lab)

A complete walkthrough with the mistakes we made and how we fixed them.

| | |
|---|---|
| **Lab ID** | ARC132 |
| **Official title** | Implement Speech and Language Solutions with Pre-trained APIs: Challenge Lab |
| **Scoring** | 100 points across 5 checkpoints |
| **Time** | 45 minutes of access, about 15 minutes of actual work |
| **Official page version used** | Manual last updated October 20, 2026; lab last tested September 29, 2026 |
| **Where you work** | The Google Cloud console and Cloud Shell (no VM, no notebook) |

| Checkpoint | Points |
|---|---|
| Task 1: Create an API key | 20 |
| Task 2: Create synthetic speech from text (Text-to-Speech) | 20 |
| Task 3: Speech-to-Text transcription in French | 20 |
| Task 4: Translate text with the Cloud Translation API | 20 |
| Task 5: Detect language with the Cloud Translation API | 20 |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake we made** show a real error from our run and the fix. Blocks marked **Pitfall to avoid** describe likely errors that we saw in sibling labs or that follow from the lab's own rules. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values written in `UPPERCASE` are placeholders.** Your lab assigns its own bucket name, and the public version of the lab page shows a stand-in (`speech_bucket`) until the lab starts. The file names and sentences below are the defaults on the official page, and your lab may show different ones. **Always copy the names and sentences from your own lab page.**

---

# Part 1: Conceptual

## What this lab is about

You are a junior cloud architect on a team that uses Google's pre-built speech and language services. You use three APIs, without writing an application, to prove you can:

1. Create an **API key** so you can call the services from the command line.
2. Turn a sentence of text into a spoken **MP3** with the **Text-to-Speech API**.
3. Turn a pre-recorded **French** audio file into text with the **Speech-to-Text API**.
4. **Translate** a Japanese sentence into English with the **Cloud Translation API**.
5. **Detect** which language a sentence is written in, also with the Cloud Translation API.

After each task you upload the files you produced to a **Cloud Storage bucket**, which is how the lab checks your work. Challenge labs give you the goal and the commands, but you have to adapt and troubleshoot them, so you apply what the course taught.

## Important concepts, explained simply

**Text-to-Speech API.** A Google service that reads text aloud and returns audio. The reply isn't a playable file straight away. It's the audio encoded as text, which is why the lab has you decode it into an MP3.

**Speech-to-Text API.** The reverse: it listens to an audio file and returns the words as text.

**Cloud Translation API.** Translates text from one language to another, and can also tell you which language a piece of text is in.

**REST API and `curl`.** You use these services by sending a web request to an address. `curl` is a command-line tool that sends the request and prints the reply.

**API key.** A long string (it starts with `AIza`) that tells Google which project is calling. It's a simple way to authenticate. Treat it like a password and don't share it in screenshots.

**API restrictions.** A setting on the key that limits which APIs it can call. This lab's key must be able to call all three services.

**Request JSON.** A small file that describes what you want done. For example, "this text, in this voice, as an MP3".

**Base64 and decoding.** A way of writing audio data using only text characters. The Text-to-Speech API returns audio this way, and `tts_decode.py` turns it back into a real MP3 file.

**Encoding and language code.** The audio file's format (`FLAC`, `MP3`) and a short language tag (`fr-FR` for French, `en-gb` for British English).

**Header vs query string.** There are two ways to send your key. Text-to-Speech in this lab sends it in a header (`X-Goog-Api-Key: ...`). The other calls put it in the web address (`?key=...`). Both work, and the lab uses each in a specific place.

**Cloud Storage and `gs://` addresses.** Google's file storage. A file's address looks like `gs://bucket-name/file-name`. A *bucket* is the top-level container.

**Environment variable and `export`.** A named value the terminal remembers, like `API_KEY`. `export API_KEY="..."` stores it and `$API_KEY` uses it. **It disappears if Cloud Shell restarts.**

**Heredoc and `EOF`.** A way to write several lines into a file at once (`cat <<'EOF' > file ... EOF`). It only works if the closing `EOF` sits alone at the very start of its line. If you paste it with indentation, the terminal never sees the end and waits forever.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. On the lab page, find the **bucket name** (the upload commands end with `gs://BUCKET_NAME/`) and write it down.
5. Don't click **End Lab** until all five checkpoints are green.

> **Mistake we made: following an out-of-date guide.** We first worked from a popular community walkthrough of this lab. It was written for an older version, and it told us to open an **SSH window on a VM**, to put the API key in the web address for Text-to-Speech, and it never uploaded anything to a bucket. When we compared it with the live page, we got corrected: "you are not following the current ARC132 instructions, check the current page".
>
> **Fix.** Always work from the lab page inside your running lab. The current version uses **Cloud Shell for all five tasks**, sends the key in a header for Text-to-Speech, and ends every task by uploading files to the bucket. This guide follows that version.

---

## Task 1: Create an API key (20 points)

### 1.1 Open the Credentials page

1. Click the **Navigation menu** (☰) at the top left.
2. Click **APIs & Services**, then **Credentials**.
3. Check that the project selector shows your `qwiklabs-gcp-...` project.

### 1.2 Create the key

1. Click **+ Create credentials**, then **API key**.
2. Leave **Name** as it is.
3. Click the **Select API restrictions** dropdown.
4. Click the filter box and type `speech`, then tick **Cloud Speech-to-Text API**.
5. Clear the filter, type `text-to-speech`, then tick **Cloud Text-to-Speech API**.
6. Clear the filter, type `translation`, then tick **Cloud Translation API** (the plain one).
7. Clear the filter and check that all three are still ticked.
8. Click **OK**.
9. Leave **Application restrictions** on **None**.
10. Click **Create**.

> **Mistake we made: the red "Make sure all fields are correct to continue" banner.** The form wouldn't submit. A red message under **Select API restrictions** said "API selection required", because the current Console won't create a key until you choose at least one API.
>
> **Fix.** Choose the APIs in the dropdown (steps 3 to 8), and the banner disappears.

> **Note: ignore the blue "Service account required" box.** It says you need to bind the key to a service account if you use the **Agent Platform API** or the **Gemini API**. This lab uses neither. Don't tick them and don't bind the key to a service account.

> **Mistake we made: not knowing which box to tick in the dropdown.** The dropdown is an alphabetical list of every Google API, so the first thing you see is names like "Actions API", "Analytics Hub API", "App Engine Admin API" and "Artifact Registry API". None of the three we wanted is visible there, so it's easy to tick the wrong one or give up.
>
> **Fix.** Don't scroll. Use the **Type to filter** box at the top of the dropdown. The three you need are **Cloud Speech-to-Text API**, **Cloud Text-to-Speech API** and **Cloud Translation API**. Tick all three, then click **OK**.

> **Pitfall to avoid: an API missing from the list.** If one of the three doesn't appear when you filter, it isn't enabled yet. Close the form, go to **APIs & Services > Library**, search for it, open it, click **Enable**, then redo steps 1.2.1 to 1.2.10.

### 1.3 Copy the key

1. In the "API key created" dialog, click the copy icon next to the key (it starts with `AIza`).
2. Click **Close**.

If you closed the dialog before copying, go to **Credentials**, find your key under **API Keys** and click **Show key**.

### 1.4 Save the key in Cloud Shell

1. Click the **Activate Cloud Shell** icon (`>_`) at the top right. Click **Continue** and **Authorize** if asked. Wait for the `$` prompt.
2. Run this, pasting your key between the quotes:

```bash
export API_KEY="PASTE_KEY_HERE"
```

3. Check it:

```bash
echo $API_KEY
```

It must print your key.

4. Save the bucket name too, and check the bucket exists:

```bash
export BUCKET="PASTE_BUCKET_NAME_FROM_LAB_PAGE"
gcloud storage ls gs://$BUCKET
```

An empty result with no error is fine. If it says the bucket doesn't exist, compare the name with the one in your lab page's upload commands.

### 1.5 Check progress

Click **Check my progress** under Task 1 (20 points).

> **Note: a new key can take a minute.** If Task 2 says "API key not valid" right after you create the key, wait a minute and try again.

---

## Task 2: Create synthetic speech from text (20 points)

### 2.1 Create `synthesize-text.json`

The official page gives you the whole command, including the text to read aloud.

1. On your lab page, find Task 2 step 1, hover over the code block and click its **copy icon**.
2. Click inside Cloud Shell, paste with **Ctrl+V** (or **Ctrl+Shift+V**), and press **Enter**.
3. Check it:

```bash
cat synthesize-text.json
```

It should show an `input` with the sentence, a `voice` with `languageCode` `en-gb`, `name` `en-GB-Standard-A` and `ssmlGender` `FEMALE`, and an `audioConfig` with `audioEncoding` `MP3`.

> **Pitfall to avoid: a stuck terminal (`>` prompts).** The official command is a heredoc, so it only finishes when `EOF` is alone at the very start of a line. In another lab we typed the heredoc with indented lines, the closing `EOF` was indented too, and the terminal showed `>` forever.
>
> **Fix.** Press **Ctrl+C** to get the `$` prompt back, and don't type `EOF` to try to finish it, because that would save the indented `EOF` inside the file. Run the command again using the **copy icon** on the lab page, which pastes the lines with no indentation, then check with `cat`.

### 2.2 Call the Text-to-Speech API

Paste and run the lab's command (the key goes in a header):

```bash
curl -s -X POST -H "X-Goog-Api-Key: ${API_KEY}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data @synthesize-text.json \
  "https://texttospeech.googleapis.com/v1/text:synthesize" > synthesize-text.txt
```

If your lab page names a different response file than `synthesize-text.txt`, use the name from your page.

### 2.3 Check the response

```bash
cat synthesize-text.txt
```

**Success:** JSON with a long `audioContent` string. **Failure:** `"error"`, so use the fixes at the end.

### 2.4 Create the decoder script

On your lab page, copy the Task 2 step 3 block (the one starting `cat <<'EOF' > tts_decode.py`) with its copy icon, paste it into Cloud Shell and press Enter. Check it saved:

```bash
ls -l tts_decode.py
```

### 2.5 Create the MP3

```bash
python3 tts_decode.py --input "synthesize-text.txt" --output "synthesize-text-audio.mp3"
```

Nothing printed means success. Check the file:

```bash
ls -lh synthesize-text-audio.mp3
```

> **Note: a backup if the script won't paste.** This one-line command does the same decoding. Run it instead of 2.4 and 2.5:
>
> ```bash
> python3 -c "import json,base64; open('synthesize-text-audio.mp3','wb').write(base64.b64decode(json.load(open('synthesize-text.txt'))['audioContent']))"
> ```

### 2.6 Upload both files

```bash
gcloud storage cp synthesize-text.txt gs://$BUCKET/
gcloud storage cp synthesize-text-audio.mp3 gs://$BUCKET/
```

Check that they landed:

```bash
gcloud storage ls gs://$BUCKET
```

### 2.7 Check progress

Click **Check my progress** under Task 2 (20 points).

---

## Task 3: Speech-to-Text transcription in French (20 points)

The audio is `gs://cloud-samples-data/speech/corbeau_renard.flac`.

### 3.1 Create the request file

This one-line command writes the same file as the official heredoc:

```bash
echo '{"config":{"encoding":"FLAC","languageCode":"fr-FR"},"audio":{"uri":"gs://cloud-samples-data/speech/corbeau_renard.flac"}}' > request.json
```

Check it, and check it's valid JSON:

```bash
cat request.json
python3 -m json.tool request.json
```

The second command should print the neatly formatted JSON with no error.

### 3.2 Call the Speech-to-Text API

Here the key goes in the web address:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @request.json \
  "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" \
  -o response.json
```

If your lab page names the files differently, use its names for both `request.json` and `response.json`.

### 3.3 View the result

```bash
cat response.json
```

**Success:** `results` containing French `transcript` text.

### 3.4 Upload both files

```bash
gcloud storage cp request.json gs://$BUCKET/
gcloud storage cp response.json gs://$BUCKET/
```

### 3.5 Check progress

Click **Check my progress** under Task 3 (20 points).

---

## Task 4: Translate text with the Cloud Translation API (20 points)

### 4.1 Create the translation request

Check the sentence matches the one on your lab page. The default is `これは日本語です。`:

```bash
echo '{"q":"これは日本語です。","target":"en"}' > translation_request.json
```

Check it:

```bash
cat translation_request.json
python3 -m json.tool translation_request.json
```

### 4.2 Call the Translation API

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @translation_request.json \
  "https://translation.googleapis.com/language/translate/v2?key=${API_KEY}" \
  -o translation_response.txt
```

### 4.3 View the result

```bash
cat translation_response.txt
```

**Success:** `data` > `translations` > `translatedText` with the English version.

### 4.4 Upload it

```bash
gcloud storage cp translation_response.txt gs://$BUCKET/
```

### 4.5 Check progress

Click **Check my progress** under Task 4 (20 points).

---

## Task 5: Detect a language with the Cloud Translation API (20 points)

### 5.1 Create the detection request

Use the sentence from your lab page. The default is `Este%é%japonês.`, with the percent signs in it:

```bash
echo '{"q":"Este%é%japonês."}' > detect_request.json
```

Check it:

```bash
cat detect_request.json
python3 -m json.tool detect_request.json
```

### 5.2 Call the API

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @detect_request.json \
  "https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" \
  -o detection_response.txt
```

### 5.3 View the result

```bash
cat detection_response.txt
```

**Success:** `data` > `detections` with a language code and a `confidence` number.

### 5.4 Upload it

```bash
gcloud storage cp detection_response.txt gs://$BUCKET/
```

### 5.5 Check progress

Click **Check my progress** under Task 5 (20 points).

---

## Finish

1. Confirm all five checkpoints are green and the score reads **100**.
2. Check the bucket holds all the files:

```bash
gcloud storage ls gs://$BUCKET
```

You should see `synthesize-text.txt`, `synthesize-text-audio.mp3`, `request.json`, `response.json`, `translation_response.txt` and `detection_response.txt`.

3. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume it.

If a checkpoint stays red, wait 20 seconds and click again, then check the file names in the bucket against your lab page, letter by letter.

### Quick fixes for other errors you might see

| What you see | Likely cause | Fix |
|---|---|---|
| Form won't create the key, red "API selection required" | No API chosen in **Select API restrictions** | Filter and tick the three APIs, then click **OK** |
| Can't find the right API in the dropdown | The list is alphabetical and starts at "A" | Use the filter box and type the API name |
| Terminal shows `>` and never returns to `$` | Heredoc pasted with an indented `EOF` | Press Ctrl+C and copy the command again from the lab page |
| `"API key not valid"` | `API_KEY` is empty or mistyped, or the key is new | `echo $API_KEY`, re-export it, wait a minute if the key was just created |
| `"API has not been used... or it is disabled"` | One of the three APIs isn't enabled | **APIs & Services > Library**, enable the API named in the error, wait a minute, retry |
| `API_KEY_SERVICE_BLOCKED` | The key's restrictions don't include that API | **Credentials**, open the key, tick all three APIs, save |
| JSON parse errors | The request file got mangled | `cat` the file and compare with the examples |
| `python3: command not found` | Python named differently | Try `python` instead |
| Empty `{}` from Speech-to-Text | Wrong audio address, encoding or language | Check `request.json` against the lab page |
| Variables blank after Cloud Shell reconnects | Variables don't survive a restart | Re-run the `export` lines from Task 1.4 |
| Upload says bucket not found | `BUCKET` is empty or wrong | `echo $BUCKET` and compare with the lab page |
| Checkpoint red though output looks right | File name mismatch in the bucket | `gcloud storage ls gs://$BUCKET` and compare names with the lab page |

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, commands and checkpoints. It adds the warnings and fixes below.

| Official lab | This guide |
|---|---|
| Task 1: create an API key in the console and export it as `API_KEY` | Same steps. Adds the API-restrictions field, the filter-box tip, the "ignore the service account box" note and the echo check. |
| Task 2 step 1: create `synthesize-text.json` with a heredoc | Same file. Says to copy it with the lab page's copy icon, and explains the `>` prompt problem. |
| Task 2 step 2: call Text-to-Speech with the key in an `X-Goog-Api-Key` header | Same command. Adds a result check. |
| Task 2 steps 3 and 4: create `tts_decode.py` and make the MP3 | Same steps. Adds a one-line backup decoder. |
| Task 2 step 5: upload the response and the MP3 | Same commands, using a `BUCKET` variable. Adds the listing check. |
| Task 3: French request, call, upload request and response | Same files and commands. Writes the request as one line and adds a JSON validity check. |
| Task 4: translation request, call, upload | Same. Adds validity and result checks. |
| Task 5: detection request, call, upload | Same. Adds validity and result checks. |

> **What the official page doesn't show.** The public version leaves your bucket name blank until the lab starts. It also doesn't tell you which APIs to tick in the key's restrictions, or how the Console's new "API selection required" rule works.
>
> **Older versions and third-party guides.** Many community walkthroughs of ARC132 describe an earlier version of the lab: they work on a VM over SSH, put the key in the web address for Text-to-Speech, and have no bucket upload. Some use `nano` to create files and an access token (`gcloud auth print-access-token`) instead of the API key. The current version uses Cloud Shell, the key in a header for Text-to-Speech, and uploads to a bucket. If a guide's steps don't match your lab page, trust your lab page.
