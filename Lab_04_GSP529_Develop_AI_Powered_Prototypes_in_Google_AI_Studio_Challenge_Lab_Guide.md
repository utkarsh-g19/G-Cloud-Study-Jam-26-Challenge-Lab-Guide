# GSP529: Develop AI-Powered Prototypes in Google AI Studio (Challenge Lab)

A complete walkthrough, with the pitfalls to watch for and how to fix them.

| | |
|---|---|
| **Lab ID** | GSP529 |
| **Official title** | Develop AI-Powered Prototypes in Google AI Studio: Challenge Lab |
| **Scoring** | 100 points across 5 checkpoints |
| **Time** | 60 minutes of access, about 35 minutes of actual work |
| **Official page version used** | Manual last updated and lab last tested May 5, 2026 |
| **Where you work** | Google AI Studio in a browser (no Cloud Shell, no commands, no code) |

| Checkpoint | Points |
|---|---|
| Task 1: Generate the core application from a prompt | 20 |
| Task 2: Add the character portraits feature | 20 |
| Task 3: Refine the UI | 20 |
| Task 4: Add the character backstory feature | 20 |
| Task 5: Add player card, stats, and "save to deck" functionality | 20 |
| Task 6: Save your prototype and download the ZIP (no separate checkpoint, the lab checks your saved app) | none |

> **How to read this guide.** Steps are numbered in the order you do them. Blocks marked **Pitfall to avoid** describe errors that are likely in this lab, with the fix. Blocks marked **Note** are things the official page doesn't spell out.
>
> **An honest note about this guide.** This lab has no commands, so there are no terminal errors to quote. We went through Task 1 together and didn't record an error in this lab, so the pitfalls below come from the lab's own rules and how AI Studio behaves, not from a failure in our run. The menu names in AI Studio can differ slightly between versions. If a button isn't where this guide says, look for the closest match.
>
> **The prompts in this guide are written to cover the official requirements.** The official page describes what each feature must do but doesn't give prompts, so you can reword them as long as every required item stays in.

---

# Part 1: Conceptual

## What this lab is about

You play a developer at a weekend game jam. You build a small **Fantasy Character Creator** web app without writing any code: you only describe what you want in plain English and an AI builds it. This style is called **vibe coding**. You start with a simple app, then keep asking for more features, one task at a time:

1. A button that generates a random fantasy character (a Name and a Class).
2. Buttons that generate and regenerate a cartoon-style portrait.
3. A fantasy look and feel for the whole page.
4. A button that writes a short backstory.
5. A bordered player card with stats and a "Save to Deck" feature.
6. Saving the app under the right name and downloading its source code.

The lab checks the **saved** version of your app after each task, so saving matters as much as building.

## Important concepts, explained simply

**Google AI Studio.** A Google tool where you try Gemini models and build small AI apps from a text prompt. In this lab you use its **Build** area.

**Vibe coding.** Describing the app you want in everyday language and letting the AI write the code. You test the result, then ask for changes.

**Prompt.** The text you give the AI. The more specific it is (button names, what should appear, what must not change), the more likely the app matches the lab's requirements.

**Code Assistant.** A chat panel inside the Build view that changes your **existing** app when you describe a change. After Task 1, you use it for every change. Using it, instead of starting a new app, keeps your earlier features.

**Gemini for text and for images.** The apps you build call Gemini models while they run: one generates text (the character's Name, Class and backstory) and one generates images (the portrait).

**Saving to Google Drive.** AI Studio stores your saved apps in the Google Drive of the account you're signed in with. The lab reads your saved app from there, which is why Task 1 asks you to give Drive access to your student account.

**State and "My Deck".** An app has to remember things while it's open, like the current character or the list of saved characters. "My Deck" is a list the app keeps in memory. The lab only needs it to exist and work inside the app.

**Iterative prompting.** Building in small steps. Each task adds one feature and keeps what came before. If a later change breaks an earlier feature, you ask the Code Assistant to restore it.

**Non-determinism.** The AI may give a slightly different app each time. The lab says the goal is to prototype, not to build a polished product.

---

# Part 2: Practical (step by step)

## Step 0: Start the lab safely

1. Open the lab in a **private/Incognito window** and click **Start Lab**. The credentials appear in the lab panel.
2. Open a **second private window** and go to `aistudio.google.com`. Put the lab and AI Studio windows side by side.
3. Click **Get Started**.
4. Copy the **Username** from the lab panel, paste it into the Email box and click **Next**.
5. Copy the **Password**, paste it and click **Next**.
6. Click **I understand** or **Accept** on any terms screen.
7. Check the account picture at the top right shows your `student-...@qwiklabs.net` account.
8. Don't click **End Lab** until all five checkpoints are green and you've downloaded the ZIP.

> **Pitfall to avoid: signed in with the wrong account.** If AI Studio shows your personal account, your app is saved to the wrong Drive and the lab can't find it, so every checkpoint stays red.
>
> **Fix.** Click the account picture, sign out, and sign in again with the **Username in your current lab panel**, in a private window.

---

## Task 1: Generate the core application from a prompt (20 points)

The official page asks for one text prompt that creates an app with a button. When clicked, it generates and displays a random fantasy character with a unique `Name` and a `Class` (for example Mage, Rogue or Warrior). You then **save** the app and enable Drive access.

### 1.1 Open the Build area

In AI Studio's left sidebar, click **Build**.

### 1.2 Enter the prompt

Click the main prompt box and paste this single prompt:

```
Build a simple Fantasy Character Creator web app for a fantasy game. The app has one button labeled "Generate Character". When the button is clicked, it uses Gemini to generate and display a random fantasy character with a unique Name and a Class (for example Mage, Rogue, or Warrior). Show the Name and Class clearly on screen.
```

### 1.3 Build it and test it

1. Click **Build** (or press Enter) and wait for the app to appear in the preview. It can take a minute or two.
2. Click **Generate Character** two or three times. Each click should show a different Name and Class.

> **Pitfall to avoid: a broken preview.** AI-generated apps sometimes show a blank page or an error on the first try.
>
> **Fix.** In the Code Assistant chat, type `Fix the error` and let it repair the app. Test again before saving.

### 1.4 Enable Drive access

1. Open the **Add files** menu (the **+** in the prompt area).
2. Click **Drive**.
3. In the pop-up window, select your student account.
4. Click **Allow** or **Continue** on every permission screen.

> **Pitfall to avoid: the permission pop-up is blocked.** The browser may hide it.
>
> **Fix.** Allow pop-ups for AI Studio and try the Drive step again. If it still doesn't appear, click **Save** first, which usually triggers the Drive permission request, then repeat the Drive step.

### 1.5 Save the app

1. Click **Save** at the top of the Build view.
2. If it asks for a name, type exactly `Fantasy Character Generator`. Task 6 needs this name, and using it from the start avoids having two copies.
3. Wait until the app shows as saved.

> **Pitfall to avoid: building without saving.** The official page says the lab verifies the **saved** app. An app that works in the preview but was never saved doesn't pass the checkpoint.

### 1.6 Check progress

Go to the lab page, wait about 20 seconds, and click **Check my progress** under Task 1 (20 points).

---

## Task 2: Add the character portraits feature (20 points)

The official page asks you to use the **Code Assistant** to add buttons to generate and regenerate a "cartoon/video game style" portrait of the currently displayed character, then save.

1. Stay in the **same app**. Don't start a new one.
2. Find the **Code Assistant** chat box.
3. Paste this prompt:

```
Add two buttons, "Generate Portrait" and "Regenerate Portrait". "Generate Portrait" should use a Gemini image generation model to create a cartoon/video game style portrait of the currently displayed character, based on their Name and Class, and show it on screen. "Regenerate Portrait" should create a new portrait for the same character in the same cartoon/video game style, replacing the previous one. Keep all existing features working.
```

4. Send it and wait for the update.
5. Test: click **Generate Character**, then **Generate Portrait**, then **Regenerate Portrait**. The portrait should appear, and the second one should differ from the first.
6. Click **Save**.
7. Click **Check my progress** under Task 2 (20 points).

> **Pitfall to avoid: image generation fails or is rate-limited.** You may see an error or a placeholder instead of a portrait.
>
> **Fix.** Wait a minute and click **Generate Portrait** again. If it keeps failing, type `The image generation is failing. Fix it.` in the Code Assistant. The checkpoint checks that the feature is in your saved app, so don't spend long on image quality.

> **Pitfall to avoid: starting a new app instead of using the Code Assistant.** A new app would lose your Task 1 work and might be saved under a different name.
>
> **Fix.** Always type changes into the Code Assistant of the **same** app. If you made a second app by mistake, go back to your first app (look under **Build** or the **By you** tab on the AI Studio home page) and continue there.

---

## Task 3: Refine the UI (20 points)

The official page asks for the look and feel of an "ancient alchemist's workbench", with a **dark, textured background** and a **stylized, fantasy-style font for the character's name**.

1. In the Code Assistant, paste:

```
Give the whole app the look and feel of an ancient alchemist's workbench. Use a dark, textured background (for example aged dark wood or stained parchment texture). Display the character's name in a stylized, fantasy-style font, such as Cinzel or MedievalSharp. Keep all existing buttons and features working.
```

2. Send it and wait for the update.
3. Check the preview. The background should be dark and textured, and the character name should use a fantasy-style font.
4. Click **Save**.
5. Click **Check my progress** under Task 3 (20 points).

> **Pitfall to avoid: the AI changes the look but drops a feature.** Restyling can remove the portrait buttons or the character display.
>
> **Fix.** Click each button in the preview. If something is gone, type `Restore the [missing feature] and keep every earlier feature working`, then test again before saving.

---

## Task 4: Add the character backstory feature (20 points)

The official page asks for a **"Generate Backstory"** button that creates a unique one-to-two-sentence origin story for the character on screen.

1. In the Code Assistant, paste:

```
Add a "Generate Backstory" button. When clicked, it uses Gemini to generate a unique origin story of one to two sentences for the character currently on screen, based on their Name and Class, and displays it below the character details. Keep all existing features working.
```

2. Send it and wait for the update.
3. Test: click **Generate Character**, then **Generate Backstory**. A story of one to two sentences should appear. Generate a new character and check the story changes.
4. Click **Save**.
5. Click **Check my progress** under Task 4 (20 points).

> **Pitfall to avoid: the button label doesn't match.** The checker looks for the feature the page names. If the AI called the button something else, such as "Create Story", rename it.
>
> **Fix.** Type `Rename the button to "Generate Backstory"` in the Code Assistant and save again.

---

## Task 5: Player card, stats, and "Save to Deck" (20 points)

The official page asks you to reformat the **entire UI into a bordered player card** with random `Health`, `Mana` and `Strength` stats, and to add a **"Save to Deck"** feature that saves favorite characters into a **"My Deck"** list inside the app.

1. In the Code Assistant, paste:

```
Reformat the entire UI into a bordered player card. The card must show the character's portrait, Name, Class, backstory, and randomly generated stats for Health, Mana, and Strength (new random values each time a character is generated). Add a "Save to Deck" button that saves the current character onto a list called "My Deck", shown in the app, so the user can keep their favorite characters. Keep all existing features working.
```

2. Send it and wait for the update.
3. Test it:
   - Click **Generate Character** and check the card shows **Health**, **Mana** and **Strength**.
   - Click **Save to Deck** and check the character appears under **My Deck**.
   - Generate another character, save it, and check both are listed.
4. Click **Save**.
5. Click **Check my progress** under Task 5 (20 points).

> **Pitfall to avoid: stats that don't change, or a deck that doesn't keep characters.** The AI may use fixed numbers, or replace the deck's contents each time you save a character.
>
> **Fix.** Type `Generate new random Health, Mana and Strength values for every new character` or `My Deck should keep every saved character and show them all`, test again, and save.

---

## Task 6: Save your prototype and download it

1. Check that the app's name is exactly `Fantasy Character Generator`. If it isn't, rename it from the app title or the Save dialog.
2. Click **Save** one last time.
3. Click the **Download** icon (or the three-dot menu, then **Download app**) to get the source code as a ZIP file.
4. Check that the ZIP is in your Downloads folder.

> **Pitfall to avoid: a different name.** The official page says to save with the name `Fantasy Character Generator`, and the lab checks your saved app. A typo or an extra copy can leave a checkpoint red.
>
> **Fix.** Open the **By you** tab (or your list of saved apps) and check there's one app with that exact name.

---

## Finish

1. Confirm all five checkpoints are green and the score reads **100**.
2. Check that the ZIP was downloaded.
3. Only then click **End Lab** and confirm. Ending the lab wipes the lab account and you can't resume it.

If a checkpoint stays red, save again, wait 20 to 30 seconds and click **Check my progress** again.

### Quick fixes for other problems you might see

| What you see | Likely cause | Fix |
|---|---|---|
| Checkpoint stays red after building | The change wasn't saved | Click **Save**, wait 20 to 30 seconds, then check again |
| Still red after saving | Wrong account, or several apps | Check the account at the top right and make sure you're editing one app |
| Preview blank or shows an error | The AI wrote faulty code | Type `Fix the error` in the Code Assistant, then save |
| Drive permission pop-up doesn't appear | Pop-up blocked | Allow pop-ups, or click **Save** first and then repeat **Add files > Drive** |
| Portrait generation errors or says quota exceeded | Image model busy | Wait a minute and retry |
| An earlier feature disappeared | A later prompt overwrote it | Type `Restore the [feature] and keep every earlier feature working`, test, then save |
| A button has the wrong label | The AI chose its own wording | Type `Rename the button to "..."` using the exact label the lab names |
| App differs from the lab's sample picture | AI results vary on each run | The lab only needs the required features, so don't chase an exact match |
| Time running low | Too much testing | Save after each prompt and keep to one prompt per task |

---

# Part 3: How this guide compares with the official instructions

The official lab page is the source of truth, and this guide follows its task order, requirements and checkpoints. It adds the prompts, the testing steps and the pitfalls below.

| Official lab | This guide |
|---|---|
| Sign in to AI Studio with the student account in a private window | Same steps. Adds a check of the account picture and a fix for the wrong account. |
| Task 1: one text prompt that makes an app with a button for a random character with a `Name` and a `Class`, then save, and enable Drive access via **Add files > Drive** | Same requirements. Supplies a ready prompt, tests, and fixes for a broken preview and a blocked Drive pop-up. |
| Task 2: use the Code Assistant to add `Generate` and `Regenerate` buttons for a "cartoon/video game style" portrait, then save | Same. Supplies the prompt and warns against starting a new app. |
| Task 3: "ancient alchemist's workbench", dark textured background, fantasy-style font for the name, then save | Same. Supplies the prompt and a check that no feature was lost. |
| Task 4: a "Generate Backstory" button with a one-to-two-sentence origin story, then save | Same. Supplies the prompt and the exact button label. |
| Task 5: bordered player card, random `Health`, `Mana` and `Strength`, "Save to Deck" into a "My Deck" list, then save | Same. Supplies the prompt and tests for random stats and a deck that keeps characters. |
| Task 6: save as `Fantasy Character Generator` and download the ZIP | Same. Suggests using that name from the first save so there's only one app. |

> **What the official page doesn't show.** It describes each feature but gives no prompts, no button locations and no testing advice, because the lab is meant to be worked out from the course labs. The prompts here are one way to meet the requirements, and any wording that keeps every required item works. The page also notes that AI results vary on each run, so your app won't look exactly like the sample.
