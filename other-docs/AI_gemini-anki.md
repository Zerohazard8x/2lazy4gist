# Guide: Multi-Chat Notebook Synthesis for Anki Flashcards & Exam Preparation Pipeline

**NOTE: This guide was created using AI (LLM / Gemini) assistance**

## 1. Step-by-Step Workflow & Chat Trigger Rules

1. PRE-PROCESSING
- Speed up audio recordings >1 hr (1.25x - 1.5x) with `ffmpeg -itsscale 0.5 -i input.mp4 -c:v copy -af "atempo=2.0" -c:a aac -c:s copy -movflags +faststart output_2x.mp4`
- Calculate card targets (~10 cards/PDF, min 5 cards/PDF)

2. CHAT 1 BASE SETUP
- Attach core foundational PDFs/notes
- Send Prompt 1 (Master Initial Setup Prompt)
- Upload Recording 1, Recording 2, Recording 3 (Send Prompts 2 & 3)

TRIGGER: CHAT 1 REACHES 3–4 AUDIO FILES ──► OPEN CHAT 2
- Open Chat 2 within the SAME Notebook Workspace
- Upload Recording 4, Recording 5, etc.
- Execute Cross-Chat Synthesis Prompt (Prompt 4B)

> *Use exam preparation pipeline in another chat in the notebook if necessary.*

3. ANKI IMPORTATION
- Import generated .tsv/.csv into Anki (Tab-delimited, Allow HTML) 

### Exact Trigger Rules for Creating New Chats
1. Upload baseline PDFs and attach up to 3 (or maximum 4) audio lecture recordings.
2. As soon as Chat 1 reaches its 3rd or 4th audio recording upload, **do not attempt to upload additional audio files in Chat 1**.
3. Create a new chat session **inside the exact same Notebook workspace**.
4. Attach Lecture Recording 4, Recording 5, and any additional PDFs inside Chat 2. The Notebook's workspace memory preserves all context from Chat 1.
5. If relevant, use **Prompt 4B (Cross-Chat Synthesis)** in Chat 1 (as the last non-video recording) to explicitly instruct the AI to pull context across all chats in the notebook.

---

## Exam Preparation Pipeline
1. Upload Past Exams -> Send Prompt 5 (Exam Conversion + Prerequisites)
2. Send Prompt 6 (Generate Full Predicted Exam)
3. Send Prompt 7 (Condense to 50% Half-Length Exam)
4. Send Prompt 8 (Convert Predicted Exam to Anki + Prerequisites)

---

## 4. Complete Exact Prompts

### Prompt 1: Master Initial Setup Prompt
> *Use in Chat 1 with foundational PDFs/documents.*

```text
Please convert the attached into an Anki .tsv / .csv with around ____ cards and at least ____ cards, with standalone card fronts for use in "Random" mode, with no similar cards, and which are fully usable with only the Anki application without other programs or materials. The Anki .tsv / .csv should have complete informational parity with the attached. 

Prepare to "bias" those cards to ____ lecture recordings which I will send in my next message.

```

---

### Prompt 2: First Audio Recording Prompt (With Speed-Up Note)

> *Use in Chat 1 when attaching the first audio file.*

```text
Attached is the first lecture recording. Note that the recording has been sped up as gemini.google.com has a maximum upload length of 1 hour

```

---

### Prompt 3: Subsequent Audio Recording Prompts

> *Use when attaching recordings 2, 3, and 4 in Chat 1.*

```text
Attached is the second lecture recording. It has also been sped up

```

```text
Attached is the third lecture recording.

```

```text
Attached is the fourth and final lecture recording.

```

---

### Prompt 4A: Master Synthesis Prompt (When Final Generation is in Same Chat)

> *Use if generating the final deck within the chat that holds all recordings.*

```text
Now I would like you to create the final Anki .tsv / .csv . Create an Anki .tsv / .csv with around __ cards and at least __ cards, with standalone card fronts for use in "Random" mode, with no similar cards, and which are fully usable with only the Anki application without other programs or materials. 

Just in case you forgot, your first response is below. What you make is supposed to be a version of it which is "biased" to the ___ lecture recordings. Do not remove information from it. However, do not just re-output it, as that means what you made is not "biased" to the lecture recordings. The Anki .tsv / .csv should have complete informational parity with any other source here.

```

---

### Prompt 4B: Master Cross-Chat Synthesis Prompt (When Final Chat Lacks Direct Recordings)

> *CRITICAL: Use this prompt in the first chat (as the last non-video prompt) when the chat generating the deck does NOT contain all audio files directly in its own upload history.*

```text
I have uploaded the remaining ___ lecture recordings in the other conversation/s in this notebook. Now I would like you to create the final Anki .tsv / .csv . Create an Anki .tsv / .csv with around __ cards and at least __ cards, with standalone card fronts for use in "Random" mode, with no similar cards, and which are fully usable with only the Anki application without other programs or materials. 

Just in case you forgot, your first response is below. What you make is supposed to be a version of it which is "biased" to the ___ lecture recordings. Do not remove information from it. However, do not just re-output it, as that means what you made is not "biased" to the lecture recordings. The Anki .tsv / .csv should have complete informational parity with any other source here.

```

---

### Prompt 5: Past Exam Conversion with Prerequisites Prompts

> *Use when uploading past exam PDFs to link exam questions back to review deck concepts.*

```text
Please keep in mind the attached quiz, ____. You may need to OCR it or use your image analysis tools

```

```text
Now please take note of this other quiz, checked by the same grader. Try to understand why the grader gave those grades, in both the first and second quizzes. In my next message, I will tell you to revise ____ into a version which you believe would have gotten a 100%

```

```text
Please create a revised version of ___.pdf which you believe would have gotten a 100% score from this grader. Change the least number of things - count/approximate the number of things you changed, with repositioning things not counting as changing any of them. If an item did not have a solution, please make a solution in the same way as the items which were already solved.

```

```text
Well done. Now, please create an Anki .tsv / .csv for ____, with everything which we did here. The .tsv / .csv should have at least 1 card per item from ___, should have standalone fronts for use in "Random" mode, and should be usable with only Anki without any other programs or files. If relevant, split the ______ questions and answers into smaller, more Anki-ready entries which would still be graded 100%. 

In addition, if you think the item cannot be answered without understanding certain things, and the things are in the below Anki file, respond with a separate code block / code file containing entries for those things. If those things cannot be understood without understanding some other things, add entries for those too if they are in the below Anki file. Do not add new data in the entries if they are not in the below Anki file.

Lastly, take note of how the grader creates questions for a later task. Be assisted by taking note of how effectively the Anki file, which I used to review for ____, corresponded to the actual content of ____.


```

---

### Prompt 6: Predicted Theoretical Exam Generation Prompt (Full Length)

> *Use to generate a new, unseen predicted exam based on past exam patterns and lecture bias.*

```text
Now, assuming the attached documents what we discussed for _____ and the below is my review Anki file, and taking into account how the grader creates questions, please create a theoretical ____ which is supposed to be solved within the same time as _____ and _____ (<time>).

```

---

### Prompt 7: Half-Length Exam Condensation Prompt

> *Use to compress the predicted exam into a rapid, high-yield practice test.*

```text
Well done. Now, please distill that theoretical ____ into half length.

```

---

### Prompt 8: Predicted Exams to Anki Conversion with Prerequisites Prompt

> *Use to convert predicted exams (full or half-length) into Anki format with prerequisite links.*

```text
Well done. Now, please create an Anki .tsv / .csv for this theoretical half-length _____, with everything which we did here. If relevant, split the theoretical ______'s questions and answers into smaller, more Anki-ready entries which would still be graded 100%. The .tsv / .csv should have full informational parity for all items in the theoretical _____, standalone fronts for use in "Random" mode, and should be usable with only Anki without any other programs or files. 

In addition, if you think the item cannot be answered without understanding certain things, and the things are in the below Anki file, respond with a separate code block / code file containing entries for those things. If those things cannot be understood without understanding some other things, add entries for those too if they are in the below Anki file. Do not add new data in the entries if they are not in the below Anki file.

```

---

### Prompt 9: Expansion & Rescaling Prompt (Adding New Sources Later)

> *Use when scaling up deck size with additional sources.*

```text
I would now like you to regenerate that file as if it was around ___ cards and at least ___ cards, converted from an additional ___ .pdfs, and was biased to ___ lecture recordings. Attached is the fifth lecture recording. In my next message, I will send the additional .pdf's. I will eventually tell you to convert those .pdf's into an Anki file of around ___ cards, which you will bias to the lecture recording.

```

---

## 5. Anki Import Guide for Beginners

1. **Save AI Output:** Copy the generated TSV/CSV text block from the AI and save it as `DSP_Study_Deck.tsv`.
2. **Open Anki:** Launch desktop Anki.
3. **Import File:** Navigate to **File** > **Import...** (`Ctrl+I` / `Cmd+I`) and select `DSP_Study_Deck.tsv`.
4. **Configure Import Settings:**
* **Card Type:** Basic (or custom 3-field note type: Front, Back, Prerequisites).
* **Field Separator:** Tab.
* **Allow HTML in fields:** **Checked / Enabled** (ensures LaTeX `$X(z)$` renders correctly).
* **Field Mapping:** Field 1 $\rightarrow$ **Front**, Field 2 $\rightarrow$ **Back / Prerequisites**.


5. **Complete Import:** Click **Import**.