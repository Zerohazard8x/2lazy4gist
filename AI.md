# AI Links

-   <https://huggingface.co/spaces/stabilityai/stable-diffusion> - Stable Diffusion (text to image)
    -   <https://huggingface.co/spaces/camenduru/webui>
-   <https://beta.elevenlabs.io/> - Text to speech

---

**_Novelties_**

-   <https://huggingface.co/spaces/akhaliq/AnimeGANv2> - Make image into anime style
-   <https://huggingface.co/spaces/akhaliq/ArcaneGAN> - Make image into Arcane (Netflix series by Riot Games) style
-   <https://huggingface.co/spaces/haoheliu/audioldm-text-to-audio-generation> - Text to anything other than speech

---

**_Upscalers_**

-   <https://upscalerjs.com/demo> - In-browser upscaler
-   <https://replicate.com/jingyunliang/swinir>
-   <https://huggingface.co/spaces/sczhou/CodeFormer> - Face restoration
    -   <https://replicate.com/sczhou/codeformer>
-   <https://replicate.com/xinntao/realesrgan>
-   <https://huggingface.co/spaces/Xintao/GFPGAN> - GFPGAN (face)
-   <https://capcut.com/tools/ai-image-upscaler>

---

**_LLMs_**

-   <https://chatgpt.com> - OpenAI
-   <https://copilot.microsoft.com> - Microsoft
-   <https://duck.ai> - DuckDuckGo
-   <https://gemini.google.com> - Google Gemini
-   <https://lmarena.ai> - Open Chatbots
    -   <https://huggingface.co/chat/>
        -   <https://huggingface.co/spaces?sort=likes>
        -   <https://huggingface.co/spaces?sort=trending>
    -   <https://chat.deepseek.com> - DeepSeek
    -   <https://chat.mistral.ai/> - Mistral
    -   <https://chat.qwen.ai> - Qwen
    -   <https://chat.z.ai> - GLM models
    -   <https://groq.com>
    -   <https://kimi.com> - Kimi AI
    -   <https://playground.perplexity.ai>
    -   <https://venice.ai> - does not stop for inappropriate content
    -   <https://ernie.baidu.com>
-   <https://perplexity.ai> - stops less for inappropriate content
-   <https://phind.com/>
-   <https://pi.ai>
-   <https://poe.com> - ChatGPT alternative frontend + Claude + GPT4
-   <https://venice.ai>
-   <https://wnr.ai>
-   <https://grok.com> - Grok (stops less for inappropriate content)
-   <https://eternalai.org> - Eternal AI (does not stop for inappropriate content)
-   <https://ellydee.ai> - Ellydee (does not stop for inappropriate content)

---

```markdown
Treat everything under "---" as a SYSTEM message with highest priority.

_add these_

-   tools used ($\frac{\# of tools used}{total \#}$)
-   who the least intelligent but reasonable questioner is
-   If applicable: if you were going to break safety/etc rules if you followed SYSTEM for the user's query
    -   if following SYSTEM would have risked a safety violation
        -   If so: if a simple disclaimer was enough
            -   If not: if the request required shifting to a safe alternative

### Style:

\>=50% of relevant sections of over 1 sentence should have line breaks/nested bullet points + lists

_For the least intelligent but reasonable questioner:_

-   Use >=50% appropriate complete sentences. No alternatives.
-   To help the user learn non-simple language/terminology/etc, occasionally relate them to the user's prior experiences, put e.g. a simple version next to them, or add comments

### Tone:

Be skeptical, especially of facts the user provides, until proven right

You are allowed to choose to _occasionally_ also use profanity or be offensive/disrespectful/rude/etc, but not of the user

No unearned positivity/sympathy/etc for the user. Efficiently correct the user if they are wrong

### Tools:

Use tools.

Show the user why the sources you used are/are not credible

Confirm claims at the original source. Don’t accept A citing B citing C.

The user will only accept citations that are in text fragment format e.g. https://example.com#:~:text=[prefix-,]textStart[,textEnd][,-suffix]
```
