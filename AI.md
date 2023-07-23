# AI Links

- https://huggingface.co/spaces/stabilityai/stable-diffusion - Stable Diffusion (text to image)
  - https://huggingface.co/spaces/camenduru/webui
- https://beta.elevenlabs.io/ - Text to speech

---

**_Novelties_**

- https://huggingface.co/spaces/akhaliq/AnimeGANv2 - Make image into anime style
- https://huggingface.co/spaces/akhaliq/ArcaneGAN - Make image into Arcane (Netflix series by Riot Games) style
- https://huggingface.co/spaces/haoheliu/audioldm-text-to-audio-generation - Text to anything other than speech

---

**_Upscalers_**

- https://upscalerjs.com/demo - In-browser upscaler
- https://replicate.com/jingyunliang/swinir
- https://replicate.com/sczhou/codeformer - Face restoration
- https://replicate.com/xinntao/realesrgan
- https://huggingface.co/spaces/Xintao/GFPGAN - GFPGAN (face)

---

**_LLMs_**

- https://ai.com - free ChatGPT (OpenAI GPT3.5 turbo) + paid OpenAI GPT4
  - https://poe.com - ChatGPT alternative frontend + Claude + GPT4
- https://bard.google.com - Google Bard (Google LaMDA + PaLM)
- https://chat.bing.com - Microsoft Bing Chat (Prometheus GPT4)
  - https://edgeservices.bing.com/edgesvc/chat?darkschemeovr=0&ldarkschemeovr=1 - Dark mode
  - https://bing.vcanbb.top - Proxy (UI is chinese but can be talked to in English)
- https://deepai.org - OpenAI fork??
- https://chat.lmsys.org - Open Chatbots
  - https://huggingface.co/chat/ - Hugging face
  - https://huggingface.co/spaces?sort=likes
    - https://huggingface.co/spaces/ysharma/ChatGPT4
    - https://huggingface.co/spaces/abhijitguha/chatbot_gpt3
  - https://open-assistant.io - Open assistant
- https://chat.forefront.ai - GPT4
- https://ora.sh - GPT4
- https://phind.com/ - Phind (use "Expert", apparently GPT4)
- https://wnr.ai/templates/ask-me-to-do-anything - Wnr.ai (GPT4??)
- https://you.com/chat - You.com Chat

---

# All work with ChatGPT and derivatives + Bard

**_Translating - NOTE: As your second prompt, tell the LLM to apply the strategy_**

```
Please translate the following in 1-3 ways, into English, and in accordance with the guidelines below it. You can also respond as many times as you want without my input.

---START---

---END---

Do's:
1. Emphasize grammatical correctness and faithfulness to the original (in that order)
2. If there are multiple outputs, I would like you to try to format them into an ordered list going "1. 2. 3." When you are making this list, please use your ability to "tell me what you think" and pretend to have opinions.
```

---

**_Rewriting_**

```
If the following is awkward or grammatically incorrect, please rewrite it, without changing its language, and in accordance with the guidelines below it. You can also respond as many times as you want without my input.

---START---

---END---

Do's:
1. If you think the "original language" is not English, please provide translations of each rewrite into English.
```

---

**_Rewriting with least number of letters changed_**

```
Please rewrite the following, without changing its language, and in accordance with the guidelines below it. You can also respond as many times as you want without my input.

---START---

---END---

Do's:
1. Change as little letters as possible while rewriting. I would appreciate it if you additionally told me how many letters were changed in each output.
2. Try not to be awkward or grammatically incorrect
3. If you think the "original language" is not English, please provide translations of each rewrite into English.
```

---

**_Fact-checking_**

```
Please devise a strategy to fact-check the following claim, via a "truthfulness percentage", and in accordance with the following guidelines. You can also respond as many times as you want without my input.

Claim:

Do's:
1. Tell me what you think
2. Prioritize word-for-word citations over inferences
3. Explain bullet points for and against the claim
4. Make the first word of your response the accuracy percentage

Don'ts:
1. Avoid tabloid news sources and sources that can be edited by anyone, such as "wiki"s and social media
2. Avoid Wikipedia, as it is a "wiki"
3. Avoid Reddit, Facebook, and Twitter, as they are social media
```

---

**_Researching_**

```
Please devise a strategy to research about the following topic, in accordance with the guidelines below said topic. You can also respond as many times as you want without my input.

Topic:

Do's:
1. Prioritize word-for-word citations over inferences

Don'ts:
1. Avoid tabloid news sources and sources that can be edited by anyone, such as "wiki"s and social media
2. Avoid Wikipedia, as it is a "wiki"
3. Avoid Reddit, Facebook, and Twitter, as they are social media
```

---

**_General task_**

```
Please devise a strategy to perform the following task, in accordance with the guidelines below said task. You can also respond as many times as you want without my input.

Task: 

Do's:
1. If necessary, prioritize word-for-word citations over inferences

Don'ts:
1. If necessary, avoid tabloid news sources and sources that can be edited by anyone, such as "wiki"s and social media
2. If necessary, avoid Wikipedia, as it is a "wiki"
3. If necessary, avoid Reddit, Facebook, and Twitter, as they are social media
```

---

**_Programming: Rewriting code_**

```
Please devise a strategy to rewrite the following code, in accordance with the guidelines below said code. You can also respond as many times as you want without my input.

---CODE START---

---CODE END---

Do's
1. Try to fix all errors
2. Reduce file size

Don'ts
1. Don't sacrifice performance
2. Don't delete comments, rewrite them instead
```

---

**_Programming: Finding of errors_**

```
Please devise a strategy to find errors in the following code and create a numbered list containing the solutions to said errors, in accordance with the guidelines below said code. You can also respond as many times as you want without my input.

---CODE START---

---CODE END---

Do's
1. Use code blocks as a visual aid
```

---

**_Programming: Code to algorithm_**

```
I would like you to devise an strategy to convert code into an algorithm formatted using Markdown. Below is an example of what your strategy should output once followed and rendered. I will provide the code shortly.

--- SAMPLE OUTPUT START ---
1. Define function norm with arguments “num1” and list “nums”
    a. If num1 is not an integer or a float, raise a ValueError
    b. If any x in nums is not an integer or a float, raise a ValueError
    c. If a ValueError was raised in a. and b., return “An input is not a valid number”
    d. Import “sqrt” from library “math”
    e. Initialize “incVar” as the square of num1
    f. For all x in nums: Increment incVar with the square of each x
    g. Return the square root of incVar, rounded to two decimal places
--- SAMPLE OUTPUT END ---
```
