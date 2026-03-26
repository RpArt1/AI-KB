---
source: aidevs4-lesson4
---


---

## SCQA Structure

**Situation:** AI agents can now process and generate text, images, audio, and video through APIs — but true multimodal support is still uneven across providers and models.

**Complication:** Each modality introduces a separate set of API limitations, cost tradeoffs, and hidden gotchas (especially around how files are passed between tools). There's no single "best" model — the right choice depends on the problem.

**Question:** How do you build agents that work effectively with multimodal inputs/outputs in practice?

**Answer:** Use a set of composable patterns: explicit file references, dedicated vision/audio tools, JSON-based prompts, template cloning, and iterative feedback loops.

---

## 5W2H — What's covered and what's missing

| | |
|---|---|
| ✅ **What** | Images, audio, video — generation, editing, analysis, transcription |
| ✅ **Why** | Production systems need more than text; agents must handle real-world content |
| ✅ **How** | Specific patterns for file passing, agent prompts, tool design |
| ⚠️ **Who** | Providers mentioned (OpenAI, Gemini, ElevenLabs) but model selection logic is vague |
| ⚠️ **When** | Which patterns apply at which project stage? Not always explicit |
| ❌ **How much** | No cost benchmarks between providers — mentioned as important but not quantified |

---

## Critical Thinking

**Strong arguments:**
- The "media tag" pattern for file references is genuinely non-obvious and fills a real API gap
- Agent-vs-workflow heuristic is clear and actionable: *static process = workflow, dynamic data = agent*
- Template cloning + targeted edits is a smart way to reuse structured prompts without rewriting them

**Weaknesses / gaps:**
- VLM reliability is acknowledged but not deeply addressed — when is human review mandatory vs optional?
- "AI can learn a style from images" is presented optimistically; the lesson admits it "usually requires several iterations with human support"
- PDF *parsing* is explicitly deferred to later lessons — important gap for real pipelines

**Logical pattern to watch:** The lesson repeatedly combines techniques from previous lessons (file system access, augmented tool calls) — each example builds on prior patterns rather than replacing them.

---

## Inversion Analysis

**How would this fail?**

| Risk | Mitigation |
|---|---|
| Agent passes raw base64 to tool instead of file reference | Explicitly document the `<media>` pattern in system prompt |
| Style drift across generated images | Use JSON template with locked fields; use reference images for pose/composition |
| Agent hallucinates image quality issues (or misses real ones) | Use a separate vision analysis call — don't rely on agent "memory" of what it generated |
| Audio responses that recite URLs or use tables | Include explicit formatting rules for audio output in system prompt |
| Video generation creates incoherent sequences | Pass last frame of each clip as first-frame reference for the next segment |

---

## Practical Patterns — The Core Takeaways

### 1. The File Reference Problem

LLMs receive images as Base64 or URL content — but **cannot read the URL string itself**. So a tool can't reference the file unless you tell it explicitly.

**Pattern: pass a `<media>` tag alongside the file content:**

```
# User message structure
[text prompt]
[image content — base64/URL]
<media>file://workspace/input/photo.jpg</media>
```

```
# System prompt addition
When calling tools that need files, use the path from the <media> tag.
```

The code converts the path to base64/URL programmatically; the agent just passes the string.

---

### 2. Agent Instructions: Goals, Limits, Patterns — NOT Steps

Wrong approach (workflow-style):
> "Step 1: List files. Step 2: Read each. Step 3: Match to profile X, Y, or Z."

Right approach (agent-style — from `config.py`):
```
## GOAL
Classify items from images/ into categories based on profiles in knowledge/.

## REASONING
1. EVIDENCE — Only use what you can clearly observe.
2. MATCHING — Profiles are minimum requirements, not exhaustive descriptions.
3. AMBIGUITY — Multiple matches → copy to all. No match → unclassified.
```

The agent figures out *how* to navigate; the prompt defines *what* counts as success and *what rules* to apply.

---

### 3. Vision Tool for Self-Verification

Agents cannot "see" images they just generated. Equip them with an explicit tool:

```python
# native_tools definition (from tools.py)
{
    "type": "function",
    "name": "understand_image",
    "description": "Analyze an image and answer questions about it.",
    "parameters": {
        "image_path": "path/to/image.jpg",
        "question": "Does this match the style guide? Any blocking issues?"
    }
}
```

The agent then runs: generate → analyze → regenerate if needed. This loop is shown in `01_04_image_editing`.

---

### 4. JSON Prompts for Image Generation

Instead of rewriting full prompts for each variation, use a JSON template and **only modify the relevant fields**:

```json
{
  "subject": "a phoenix rising from flames",
  "style": "cinematic realism",
  "lighting": "dramatic backlighting",
  "composition": "rule of thirds",
  "camera": "wide angle, low angle",
  "mood": "epic, triumphant"
}
```

Agent workflow:
1. Clone `template.json` to new directory
2. Edit only the fields that change
3. Pass template filename (not content) to the generation tool

Benefit: style consistency across images, no token waste rewriting stable fields.

---

### 5. Reference Images for Composition Control

Pass a reference image to "lock in" a pose or composition, combined with a JSON prompt to control style:

```
references/pose_walking.png  →  controls body position
character-template.json      →  controls visual style
<media> tag                  →  lets agent pass it to the tool
```

Use case: e-commerce product shots, ad materials, character consistency across scenes.

---

### 6. Audio — Format Responses for Ears, Not Eyes

When building audio-output agents:
- No URLs, no markdown tables, no code blocks in responses
- Plain prose, short sentences, natural spoken rhythm
- Add this explicitly to the system prompt

Decision point for audio architecture:

| Scenario | Approach |
|---|---|
| Simple transcription / TTS | Separate STT + TTS models |
| Rich analysis (tone, diarization, environment) | Native multimodal (Gemini Flash) |
| Real-time voice chat | Live API — but still unstable/expensive |

---

### 7. Video Generation — Frame Chaining

Model limit (e.g., Kling): 10 seconds per clip. To generate longer videos:

```
Image A → [Kling] → Clip 1 (0-10s)
           last frame of Clip 1 = first frame for Clip 2
           → [Kling] → Clip 2 (10-20s)
           ...
```

Missing piece mentioned: a tool to concatenate clips — but straightforward to add (ffmpeg).

---

## Immediate Takeaways

**1. Always add a `<media>` tag for file-using agents**
Why: Without it, agents can't reference files in tool calls — they have the image but not the path.
Action: Add `<media>{file_path}</media>` + system prompt instruction to every agent that uses file-based tools.

**2. Equip image-generating agents with a vision verification tool**
Why: Agents are blind to their own output. Without this, the generation loop has no feedback.
Action: Add an `understand_image` tool and instruct the agent to verify before finalizing.

**3. Use JSON templates + clone-and-edit for image prompts**
Why: Consistent style, cheaper iteration, no rewriting stable parameters.
Action: Create a `template.json` for your next image generation use case; let the agent only modify the `subject` field first.
