# Lab 7: Create Your First Gemini Enterprise Application

Questions, answers and explanations.

| | |
|---|---|
| **Course** | Create Your First Gemini Enterprise Application |
| **Topics covered** | Setting up Gemini Enterprise, connecting data, syncing data, access control, Gemini Notebook, Deep Research, image analysis |
| **Number of questions** | 8 |
| **Question type** | Multiple choice, one correct answer each |

> **How to use this file.** Each question is followed by the correct answer and an explanation of why it's right and why the other options are wrong. Answer options can appear in a different order in your quiz, so match the **wording**, not the letter.
>
> **Source basis.** Questions 1, 3 and 7 were checked against Google's Gemini Enterprise documentation. Questions 4, 5 and 8 were checked against how Gemini Notebook and its Deep Research feature are described by Google and by independent write-ups. Questions 2 and 6 rest on how Gemini Enterprise treats existing permissions and image input in general. If your course material words an answer differently, follow your course material.

## Answer key

| # | Topic | Correct answer |
|---|---|---|
| 1 | First setup task | Choose an identity provider |
| 2 | File permissions | Users can only access documents already shared with them via existing permissions |
| 3 | Importing PDFs with filterable data | Documents with Metadata (RAG) |
| 4 | Question not covered by the sources | It will generate an answer based only on the content of the uploaded documents |
| 5 | Report from hundreds of external websites | Deep Research agent |
| 6 | Understanding a pie chart saved as a PNG | Ask the AI assistant to "Summarize the customer sentiment pie chart" |
| 7 | Keeping Cloud Storage data current | Periodic |
| 8 | Video overview from a few specific text files | Gemini Notebook |

---

## Question 1

A company's Google Cloud administrator is performing the initial one-time configuration for a Gemini Enterprise application and needs to ensure the organization's workforce can securely authenticate and access their internal data. Which task should they perform first?

- Define a data synchronization frequency
- Connect the organization's data stores
- Enable the Gemini Code Assist API
- ✅ **Choose an identity provider**

**Explanation.** Setting up the identity provider comes first because it decides who the users are and how they sign in. It's also what lets Gemini Enterprise enforce access control on the data you connect later. Without it, the rest of the setup has no way of knowing who is allowed to see what.

**Why the others are wrong:**
- **Define a data synchronization frequency.** This is a choice you make while connecting a data store, so it comes after the data store exists.
- **Connect the organization's data stores.** This comes after identity is set up, so permissions can be applied to the data as it's ingested.
- **Enable the Gemini Code Assist API.** Code Assist is a separate product for developers. It has nothing to do with workforce sign-in for Gemini Enterprise.

---

## Question 2

An administrator is concerned that connecting a shared Google Drive to the Gemini Enterprise application will allow everyone in the company to read private, unshared documents. How does Gemini Enterprise handle file permissions?

- It automatically grants read access to all employees for indexed files
- ✅ **Users can only access documents already shared with them via existing permissions**
- It requires the IT department to manually re-assign permissions for every file
- All connected files become public to any user with the application URL

**Explanation.** Gemini Enterprise respects the access controls that already exist in the source system. A user only gets answers drawn from files they could already open in Drive. Connecting a data source doesn't widen anyone's access. This is also the reason the identity provider matters: the platform has to know who is asking in order to apply those permissions.

**Why the others are wrong:**
- **Automatically grants read access to all employees.** Indexing doesn't change who can read a file.
- **Requires IT to re-assign permissions for every file.** Existing permissions are reused, so no manual re-assignment is needed.
- **All connected files become public.** Nothing becomes public to anyone with the URL. Access still depends on the signed-in user.

---

## Question 3

A retail company has detailed product descriptions in PDF files and also has a corresponding set of structured data (product ID, price, stock level) for each PDF. To ensure that users can filter search results by price or stock level, which data import option should they choose?

- Documents
- ✅ **Documents with Metadata (RAG)**
- Tabular/E-commerce
- Media/Catalog Assets

**Explanation.** The PDFs are unstructured content, and the product ID, price and stock level are structured values that belong to each PDF. Importing the PDFs *with metadata* attaches those values to each document, so users can filter search results by them. In Cloud Storage this is done with a file that has one record per document, containing the structured fields and a pointer to the PDF.

**Why the others are wrong:**
- **Documents.** Imports the files only, so there are no price or stock fields to filter on.
- **Tabular/E-commerce.** Meant for structured, table-like data such as a product catalog. It has no PDF content for the search to read.
- **Media/Catalog Assets.** Meant for media content such as video or other catalog items, not documents with attributes.

---

## Question 4

You ask Gemini Notebook a question about a product that is not mentioned in any of the sources you uploaded to the current notebook. How will the tool respond?

- It will search your connected Google Drive to find the answer
- ✅ **It will generate an answer based only on the content of the uploaded documents**
- It will automatically browse the web to fill in the missing information
- It will ask you to add your Google Calendar as a source

**Explanation.** Gemini Notebook answers are grounded in the sources in the current notebook. When the sources don't cover the question, it doesn't go looking elsewhere. In practice it tells you the sources don't contain the answer instead of inventing one. To bring in more information, you have to add sources yourself (for example a document, a web link, or a Deep Research report).

**Why the others are wrong:**
- **Searches your connected Drive.** It only uses the sources in the notebook, not everything you have access to.
- **Automatically browses the web.** Web research happens only when you start it, for example with Deep Research.
- **Asks you to add Google Calendar.** Calendar isn't part of how a notebook answers questions.

---

## Question 5

Cymbal Foods needs a comprehensive report on global food marketing trends by analyzing hundreds of external websites. Which feature should they use?

- Google Calendar data sync
- Gemini Notebook grounded questions
- ✅ **Deep Research agent**
- Search internal sources enhancement

**Explanation.** Deep Research is an agent built for this job. It plans the research, browses a large number of websites, and compiles a cited report. The need to analyze hundreds of *external* websites points to it.

**Why the others are wrong:**
- **Google Calendar data sync.** Brings calendar data into the system. It does no web research.
- **Gemini Notebook grounded questions.** Answers questions from sources you've already uploaded, not from hundreds of new websites.
- **Search internal sources enhancement.** Improves searching of the company's own data, not external websites.

---

## Question 6

A marketing team lead needs to quickly understand the primary findings of a customer sentiment pie chart saved as a PNG file. How can they use Gemini Enterprise to achieve this?

- ✅ **Ask the AI assistant to "Summarize the customer sentiment pie chart."**
- Request a Deep Research report on pie chart design trends
- Transcribe the chart into a spreadsheet and connect it as a data store
- Use the Google Calendar data store to find the meeting where the chart was shown

**Explanation.** Gemini models can read images as well as text, so the assistant can look at the PNG directly and summarize what the chart shows. It's the quickest route and needs no setup.

**Why the others are wrong:**
- **Deep Research on pie chart design trends.** That would research how pie charts are designed, not what this chart says.
- **Transcribe into a spreadsheet and connect it as a data store.** A long detour for a one-off question, and it still needs someone to read the chart first.
- **Use the Calendar data store to find the meeting.** It finds meetings, not the content of an image file.

---

## Question 7

A company stores its weekly sales reports in a designated Cloud Storage bucket. To ensure their Gemini Enterprise application always reflects the latest data, which synchronization frequency should they use?

- ✅ **Periodic**
- Multiple ingestion types
- Streaming
- One time

**Explanation.** For Cloud Storage, there are two ingestion options. **One-time** ingestion imports the data once, and it doesn't change unless someone refreshes it manually. **Periodic** ingestion sets a sync frequency, and the data store then updates automatically on that schedule (every one, three or five days). New weekly reports keep arriving, so periodic sync is the right choice, and the shortest interval keeps the data closest to current.

**Why the others are wrong:**
- **One time.** The data would go stale as soon as a new weekly report is added.
- **Streaming.** Not a synchronization option for Cloud Storage.
- **Multiple ingestion types.** Not a frequency at all.

---

## Question 8

A researcher has a small set of specific text files and wants to create a video overview that summarizes only those sources. Which tool is best suited for this task?

- Idea Generation Agent
- Deep Research Agent
- General Gemini Enterprise Search
- ✅ **Gemini Notebook**

**Explanation.** Gemini Notebook works from the sources you put into it, and it can turn them into outputs such as a video overview or an audio overview. A small set of specific files that must be summarized on their own is exactly what it's designed for.

**Why the others are wrong:**
- **Idea Generation Agent.** Produces and ranks new ideas. It isn't a tool for summarizing given files into a video.
- **Deep Research Agent.** Looks outward at many websites, so it would bring in material beyond those files.
- **General Gemini Enterprise Search.** Finds information across connected data stores. It doesn't produce a video overview limited to chosen files.

---

## Quick reference: the ideas behind the answers

| Idea | What to remember |
|---|---|
| **Setup order** | Identity provider first, then connect data stores, then choose how often they sync |
| **Permissions** | Gemini Enterprise follows the source system's existing permissions, so it never widens access |
| **Documents vs. documents with metadata** | Documents alone have nothing to filter on. Metadata is what makes filtering by fields like price or stock possible |
| **Tabular/E-commerce** | For structured, table-like product data, not PDF content |
| **Gemini Notebook** | Grounded in the sources in the notebook only. Can create overviews such as video and audio from them |
| **Deep Research** | The agent for broad research across many external websites, producing a cited report |
| **Images** | The assistant can analyze images directly, so a chart in a PNG can be summarized with a simple prompt |
| **Cloud Storage sync** | One-time = data doesn't change unless refreshed. Periodic = automatic updates on a schedule. Streaming isn't an option for Cloud Storage |
