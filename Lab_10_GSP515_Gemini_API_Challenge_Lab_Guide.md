# GSP515: Explore Generative AI with the Gemini API in Agent Platform (Challenge Lab)

A complete walkthrough with the mistakes we made and how we fixed them.

|                                |                                                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| **Lab ID**                     | GSP515                                                                                                  |
| **Official title**             | Explore Generative AI with the Gemini API in Agent Platform: Challenge Lab (formerly "...in Vertex AI") |
| **Scoring**                    | 100 points across 3 checkpoints                                                                         |
| **Time**                       | 90 minutes of access, about 25 minutes of actual work                                                   |
| **Official page version used** | Manual and tests last updated August 19, 2026                                                           |
| **Where you work**             | Cloud Shell (Task 1), then a JupyterLab notebook (Tasks 3 and 4)                                        |

| Checkpoint                                              | Points |
| ------------------------------------------------------- | ------ |
| Enable the required API and call Gemini fom Cloud Shell | 20     |
| Create a function call with Gemini (notebook)           | 40     |
| Describe video contents with Gemini (notebook)          | 40     |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Mistake we made** show a real error from our run and the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **Values in `UPPERCASE` or `<angle brackets>` are placeholders.** Your lab fills in its own project ID, model ID and so on. Always copy those from your own lab page, not from this guide.

---

# Part 1: Conceptual

## What this lab is about

Challenge labs don't teach you new concepts. They give you a scenario and expect you to apply what you learned in the course. In GSP515 you use Google's **Gemini** generative AI model in three ways:

1. **Call the model directly** over its REST API from the command line (Cloud Shell).
2. **Give the model a tool** (function calling) so it can ask your code to fetch live information, here the weather.
3. **Send it a video** and ask it to describe what's in it (multimodal prompting).

The notebook is pre-written with cells that contain blanks marked `INSERT`. Your job is to fill in the blanks correctly and run the cells in order.

## Important concepts, explained simply

**Agent Platform (formerly Vertex AI).** Google's managed platform for building with AI models. Google renamed Vertex AI to _Gemini Enterprise Agent Platform_. Some pages, links and docs still say "Vertex AI". The underlying API is still `aiplatform.googleapis.com`. This matters later, because searching for "Vertex AI API" in the API Library may not find it.

**Gemini and the model ID.** Gemini is the AI model. A _model ID_ (for example `gemini-3.6-flash` in our run) is the exact name you send to say which version you want. A wrong or misspelled ID gives a "model not found" error.

**API, endpoint and location.** An _API_ is a way for programs to talk to a service. The _endpoint_ is its web address. The _location_ says which region handles the request. This lab uses `global`.

**Enabling an API.** Google Cloud services are switched off in a project until you enable them. If the API isn't enabled, your calls fail with a 403 error.

**REST call with `curl`.** `curl` sends a web request from the command line. You send your question as JSON and get a JSON answer back. The access token (`gcloud auth print-access-token`) proves who you are.

**Jupyter notebook and kernel.** A notebook is a document made of _cells_. You run each code cell with Shift+Enter. The _kernel_ is the Python process that runs your cells and remembers variables between them. Restart the kernel and it forgets everything, so you have to rerun the setup cells.

**Function calling.** A language model can't look up the weather by itself. With function calling, you _describe_ a function to the model (its name, what it does, what inputs it takes). When you ask "What is the weather like in Boston?", the model doesn't answer directly. Instead it replies, "Please call `get_current_weather` with location = Boston." Your code would then run that function and hand the result back. In this lab you only see the model's request, and that's the checkpoint.

**Function declaration vs tool.** A _declaration_ describes one function. A _tool_ is a bundle of declarations you hand to the model. So you build the declaration first, wrap it in a tool, then pass the tool when you call the model.

**Multimodal.** The model accepts more than text: images, audio and video too. Here you send text plus a video file stored in Cloud Storage.

**URI and MIME type.** A _URI_ such as `gs://bucket/file.mp4` is the video's address in Cloud Storage. The _MIME type_ (`video/mp4`) tells the model what kind of file it is.

**Streaming.** A streamed response arrives in pieces instead of all at once. The notebook's output loop prints each piece as it arrives, so the call has to be a streaming call.

**429 errors.** "Too many requests". It's a temporary rate limit. Wait a minute and rerun the same cell.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**.
2. Click **Open Google Cloud console**. Sign in with the **Username and Password from the lab panel**, accept the terms and skip the recovery page.
3. Check that the project selector shows a `qwiklabs-gcp-...` project.
4. Don't click **End Lab** until all three checkpoints are green.

> **Note.** Use only the student account in that window. If you have several student accounts open, the Console and Cloud Shell can end up using different accounts, which causes sign-outs and permission errors. We hit this in later labs.

---

## Task 1: Enable the API and call Gemini from Cloud Shell (20 points)

### 1.1 Open Cloud Shell and set the variables

1. Click **Activate Cloud Shell** (the `>_` icon, top right). Click **Continue**, then **Authorize** if asked.
2. On the lab page, find the **Task 1, step 1** code block. It starts with `PROJECT_ID=...` and also sets `LOCATION`, `API_ENDPOINT` and `MODEL_ID`.
3. Hover over it and click the **copy** icon, paste it into Cloud Shell, and press **Enter**.
4. Verify all four values:

```bash
echo $PROJECT_ID $LOCATION $API_ENDPOINT $MODEL_ID
```

In our run this printed a `qwiklabs-gcp-...` project ID, `global`, `aiplatform.googleapis.com`, and `gemini-3.6-flash`.

> **Note.** Copy the block from the lab page instead of typing it. It carries your project and the exact model ID. Your model ID may differ from ours.

### 1.2 Enable the API

The official lab tells you to enable the API through the Console (APIs & Services). We hit a problem there:

> **Mistake we made.** We searched the API Library for **"vertex ai api"** and got one result, **AlloyDB API**, which is the wrong service. A banner on the page said _"Vertex AI is now Gemini Enterprise Agent Platform."_ The API had been renamed, so the old search term no longer found it.
>
> **Fix.** Skip the library and enable it from Cloud Shell using the service name, which didn't change:
>
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```
>
> Wait for `Operation ... finished successfully`. Confirm it:
>
> ```bash
> gcloud services list --enabled | grep aiplatform
> ```
>
> You should see `aiplatform.googleapis.com`. If you prefer the Console, search for **Agent Platform** instead, and don't click the AlloyDB result.

If `gcloud services enable` returns "permission denied", use the Console route above.

### 1.3 Call the model

Wait 30 to 60 seconds after enabling the API, then paste this as **one command**:

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://${API_ENDPOINT}/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/google/models/${MODEL_ID}:generateContent" \
  -d '{"contents":[{"role":"user","parts":[{"text":"Why is the sky blue?"}]}]}'
```

The question text must be **exactly** `Why is the sky blue?`.

You should get JSON back with `"candidates"` and an explanation (Rayleigh scattering).

| Symptom                    | Cause                                     | Fix                                                      |
| -------------------------- | ----------------------------------------- | -------------------------------------------------------- |
| `403` or "API not enabled" | API not enabled yet, or still propagating | Redo 1.2, wait a minute, retry                           |
| `404` or "model not found" | Wrong `MODEL_ID` or `LOCATION`            | `echo $MODEL_ID $LOCATION` and compare with the lab page |
| Variables are blank        | Cloud Shell reconnected                   | Paste the variable block from 1.1 again                  |

### 1.4 Check progress

Click **Check my progress** under Task 1. If it's red, wait 20 seconds and click again.

---

## Task 2: Open the notebook (no checkpoint)

1. **Navigation menu (☰) > Agent Platform > Notebooks > Workbench**. On some Consoles this is **Vertex AI > Workbench**. If you can't find it, type "Workbench" in the top search bar.
2. Wait until the instance shows as active (green tick).
3. Click **Open JupyterLab**. A new tab opens. Allow pop-ups if it's blocked.
4. In the file browser on the left, double-click the `.ipynb` notebook named on the lab page.

---

## Task 3: Create a function call (40 points)

### 3.0 Kernel and setup cells

1. In the **Select Kernel** dialog, choose **Python 3 (Local)** and click **Select**. If there's no dialog, use **Kernel > Change Kernel...**.
2. Run the **Getting Started** cells **one at a time, top to bottom** with Shift+Enter. Wait for `[*]` to turn into a number before running the next cell.
3. The cells do four things in order: install the Google Gen AI SDK, restart the kernel, create the client, and import classes.

> **Mistake we nearly made: panicking at red output.** The pip install cell printed yellow `WARNING: Ignoring invalid distribution ~oogle-auth` lines. The restart cell printed a pink `DeprecationWarning` box followed by `{'status': 'ok', 'restart': True}`. Both are **normal and harmless**. The restart worked.
>
> **Fix / rule.** Don't rerun the restart cell. A restart wipes all variables, and you would have to rerun the client and imports cells again. If you ever do restart, rerun from **Set Google Cloud project information and create client** downward.

After the restart, you'll see two cells that must have run:

```python
# Define project information
PROJECT_ID = "qwiklabs-gcp-..."   # pre-filled
LOCATION = "global"

# Create the API client
from google import genai
client = genai.Client(enterprise=True, project=PROJECT_ID, location=LOCATION)
```

```python
from google.genai.types import (
    FunctionDeclaration,
    GenerateContentConfig,
    Tool,
    Part
)
```

> **Note.** The imports cell tells you the exact names available. This mattered for Task 3.2, where the answer depends on how the class was imported.

If any cell gives **429**, wait one minute and rerun that same cell.

### 3.1 Load the model

Find the cell starting `# Task 3.1`. The comment says to load the Gemini 3.6 Flash model. Replace the **whole** placeholder, including `<<` and `>>`:

```python
model_id = "gemini-3.6-flash"
```

Use the same model ID you saw in Cloud Shell (`echo $MODEL_ID`). Press Shift+Enter.

> **Mistake we made.** We ran this cell once with `"<<INSERT CORRECT MODEL NAME HERE>>"` still in it, so `model_id` held junk text.
>
> **Fix.** Replace the placeholder and **run the cell again**. Later cells read `model_id`, so a stale value breaks them. A cell number such as `[3]` showing doesn't mean the cell is correct. It only means it ran.

### 3.2 Define the function declaration

The cell begins:

```python
get_current_weather_func = INSERT CORRECT CLASSNAME(
    name="get_current_weather",
    description="Get the current weather in a given location",
    parameters={ ... }
)
```

> **Mistake we made.** We ran the cell as-is and got:
>
> ```
> SyntaxError: invalid syntax
> get_current_weather_func = INSERT CORRECT CLASSNAME(
> ```
>
> `INSERT CORRECT CLASSNAME` is placeholder text, not Python.
>
> **Fix.** Replace only that phrase. Because the imports cell imported `FunctionDeclaration` directly, no `types.` prefix is needed:
>
> ```python
> get_current_weather_func = FunctionDeclaration(
> ```
>
> Leave everything else exactly as it was: `name`, `description`, the `parameters` block and the closing `)`.

**Which form to use depends on your imports:**

| Your import line                                            | What to write                |
| ----------------------------------------------------------- | ---------------------------- |
| `from google.genai.types import (FunctionDeclaration, ...)` | `FunctionDeclaration(`       |
| `from google.genai import types`                            | `types.FunctionDeclaration(` |

Run with Shift+Enter. There's no output when it works. The cell just gets a number.

### 3.3 Wrap it in a tool

```python
weather_tool = Tool(
    function_declarations=[get_current_weather_func],
)
```

Change only `INSERT CORRECT CLASSNAME` to `Tool`. Keep the variable name `get_current_weather_func`, since it must match what 3.2 defined.

> **Note.** If you get `NameError: name 'get_current_weather_func' is not defined`, cell 3.2 didn't run successfully. Go back and fix it first.

### 3.4 Call the model with the tool

Replace `INSERT TOOL TO USE` with the tool you just built:

```python
prompt = "What is the weather like in Boston?"
response = client.models.generate_content(
    model=model_id,
    contents=prompt,
    config=GenerateContentConfig(
        tools=[weather_tool],
        temperature=0,
    ),
)
response
```

Keep the square brackets, because `tools` expects a list. Press Shift+Enter.

**Expected output:** the response contains a `function_call` for `get_current_weather` with the location `Boston`. That's the weather-related data the lab says you must see. The model isn't answering the weather question. It's asking for your function to be called.

If you don't see the function call, one of the cells above has an error or a wrong `INSERT`. Don't continue until it appears.

### 3.5 Check progress

Go to the lab page and click **Check my progress** for **Create a function call** (you should be at 60 points in total).

---

## Task 4: Describe video contents (40 points)

### 4.1 Load the multimodal model

Run the import cell that appears just above (it imports `GenerationConfig`, `Image`, `Part`). Then in the `# Task 4.1` cell:

```python
multimodal_model = "gemini-3.6-flash"
```

The comment says "Load Gemini 3.6 Flash Model", so use the same ID as Task 3.1. Run it.

### 4.2 Run the helper cells

The next cell imports `http.client`, `typing`, `urllib.request`, `IPython.display` and `PIL`. Run it **as-is**. Keep running the cells below it, in order, until you reach the `# Task 4.2` cell. Don't skip any. One of them defines `print_multimodal_prompt`, which the final cell needs.

> **Note.** If you get `NameError: name 'print_multimodal_prompt' is not defined`, scroll up, run the helper cell, then rerun the Task 4.2 cell.

### 4.3 Describe the video

The Task 4.2 cell has **four** blanks. The URI is in the cell's own comment: `# Video URI: gs://github-repo/img/gemini/multimodality_usecases_overview/mediterraneansea.mp4`.

```python
# Task 4.2 Generate a video description
prompt = """
Describe the contents of this video in detail.
What is shown in this video?
Where should I go to see it?
What are the top 5 places in the world that look like this?
"""
video = Part.from_uri(
    file_uri="gs://github-repo/img/gemini/multimodality_usecases_overview/mediterraneansea.mp4",
    mime_type="video/mp4",
)
contents = [prompt, video]

responses = client.models.generate_content_stream(
    model=multimodal_model,
    contents=contents
)

print("-------Prompt--------")
print_multimodal_prompt(contents)

print("\n-------Response--------")
for response in responses:
    print(response.text, end="")
```

| Blank                        | What goes there                                                 | Why                                                                      |
| ---------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Prompt                       | Add a line asking Gemini to describe the video                  | The comment says "update the prompt to ask Gemini to describe the video" |
| `INPUT CORRECT CLASSNAME`    | `Part`                                                          | Already imported, and `.from_uri(...)` is a `Part` method                |
| `INSERT VIDEO URI`           | The `gs://...mediterraneansea.mp4` path from the cell's comment | Copy it from the comment so it isn't mistyped                            |
| `INSERT CORRECT METHOD NAME` | `generate_content_stream`                                       | The output loop (`for response in responses`) expects a stream           |

> **Mistake to avoid.** If you use plain `generate_content` here, the result is a single response, and looping over it with `response.text` can fail. We chose `generate_content_stream` to match the loop that's already in the cell. One user on the Google Cloud forum reported an error from that loop on an older version and said `print(responses.text)` worked instead. That's a single user's report, so only try it if you actually hit that error.
>
> **Don't edit** the `print_multimodal_prompt(contents)` line or the output loop.

Press Shift+Enter. This one can take **20 to 60 seconds**. Wait until `[*]` becomes a number. Expected output: the prompt printed first, then a text description of the Mediterranean Sea video.

If you get **429**, wait one minute and rerun this cell.

### 4.4 Check progress

Click **Check my progress** under **Describe video contents**.

---

## Finish

1. Confirm all three checkpoints are green and the score reads **100**.
2. Only then click **End Lab** and confirm. Ending the lab wipes everything and you can't resume.

---

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order and checkpoints. It adds the explanations and warnings below.

| Official lab                                                                    | This guide                                                                                                                    |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Task 1: set variables, enable API, call Gemini from Cloud Shell, check progress | Same steps. Adds the AlloyDB search problem and the `gcloud services enable` route.                                           |
| Task 2 / 3: open notebook, run Getting Started cells, fill the Task 3 cells     | Same steps. Adds the hint that harmless red warnings and restart output are normal, and a rule not to rerun the restart cell. |
| Task 3: INSERT cells for the model, declaration, tool, request                  | Same steps. Adds the exact class names and why, based on the imports cell.                                                    |
| Task 4: INSERT cells for the video request                                      | Same steps. Adds the table of four blanks and the streaming explanation.                                                      |
| The lab's own note about 429 errors                                             | Same advice: wait a minute and rerun.                                                                                         |

> **What the official page doesn't show.** Your exact model ID, project ID and the notebook's INSERT cell wording are filled in or shown only inside your own running lab. Where this guide says "copy from the lab page", do that. The class and method names in Part 2 matched our run, but if your notebook version differs, trust what is in your notebook and use the logic in the tables to decide.

---
