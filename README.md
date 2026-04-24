# TwinMind Copilot

A real-time AI meeting assistant. Records live audio, transcribes it in 
30-second chunks using Groq Whisper, and surfaces 3 contextual suggestions 
after each chunk. Click any suggestion for a detailed answer, or ask 
your own questions in the chat panel.

## Live Demo
[Netlify URL]

## GitHub
[GitHub URL]

## Setup
1. Open the deployed URL in Chrome
2. Open Settings (⚙) and paste your Groq API key (`gsk_...`)
3. Click **⏺ Record** and allow microphone access
4. Speak — transcript chunks appear every ~30 seconds
5. Suggestions auto-generate after each chunk, or click **↻ Refresh**
6. Click any suggestion card for a detailed answer in the Chat panel
7. Type follow-up questions directly in the chat input
8. Click **⬇ Export** to download the full session as JSON

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | Vanilla HTML/CSS/JS | No build step, single file deploy, every line is readable |
| Transcription | Groq Whisper Large V3 | Fast, accurate, specified in brief |
| LLM | Groq llama-3.3-70b-versatile | Best available OSS model on Groq |
| Hosting | Netlify static deploy | Drag-and-drop, instant, zero config |

No framework, no bundler, no dependencies. The entire app is one HTML file.

## Prompt Strategy

This is the core of the product. Three prompts, each with a distinct job.

### Suggestion prompt — reads the moment

Suggestions use only the **last 800 characters** of transcript, not the full 
history. This is intentional. Suggestions should react to what was JUST said. 
Longer context causes the model to summarize the entire meeting instead of the 
current moment — generating broad, generic cards instead of timely ones.

The prompt gives the model explicit type-selection rules rather than leaving 
it to decide freely:
- If a question was just asked → `answer` — respond directly
- If a specific number or claim was stated → `fact_check` — verify it
- If an obvious angle hasn't been covered → `question` or `talking_point`
- If a term was used without explanation → `context`

Previews are constrained to under 12 words and must be specific to what was 
actually said. The model is explicitly prohibited from writing generic previews 
like "Key considerations" or "Important factors."

The prompt returns structured JSON only — no markdown, no explanation. JSON 
is enforced via prompt instruction rather than structured outputs, which avoids 
an extra API round-trip and keeps latency low.

### Detail prompt — type-aware responses

When a card is clicked, `{type}` and `{suggestion}` placeholders are replaced 
with the card's actual values before the API call. The model receives a rigid 
per-type format to follow:

- `fact_check` → state the claim, give the accurate information, add one line of context
- `question` → give the direct answer, add one sentence on what else is worth exploring
- `talking_point` → state the point, give two supporting reasons, suggest how to introduce it
- `answer` → lead with the answer, add one sentence of supporting context
- `context` → define in one sentence, explain why it matters here in one sentence

Maximum 120 words enforced. One prompt template handles all five types. The 
strict format prevents the model from defaulting to its training pattern of 
long bullet-pointed reports, which are not useful in a live meeting context.

### Chat prompt — memory via conversation history

Free-text chat sends the **entire conversation history** on every API call. 
LLMs have no built-in memory between calls — passing history is how multi-turn 
coherence works. The system message also includes the last 3000 characters of 
transcript so every answer has full meeting context.

Suggestion clicks intentionally do NOT use chat history they are one-shot 
deep dives into a specific card, not continuations of a conversation. This 
keeps the detail response focused on the suggestion rather than being influenced 
by previous chat turns.

Chat responses are constrained to 2-3 sentences for simple questions and 
maximum 5 bullet points for complex ones. This keeps answers usable during 
a live meeting rather than reading like a report.

## Architecture
Browser mic
→ MediaRecorder (30s chunks)
→ Groq Whisper Large V3 (transcription)
→ state.transcript[]
→ generateSuggestions() — last 800 chars → Groq LLM → 3 cards
Card click / chat input
→ handleSuggestionClick() or sendChatMessage()
→ full transcript + chat history → Groq LLM
→ answer rendered in chat panel
Export button
→ state serialized to JSON → downloaded to disk

All state lives in one plain JS object. No framework, no reactivity library.
Every render function reads from state and rewrites the relevant DOM section.

## Tradeoffs

**800 char suggestion context vs full transcript**  
Short = timely, specific suggestions. Full transcript = generic summaries 
of the whole meeting. The right tradeoff for a live copilot is recency.

**Fixed 30s chunks vs voice activity detection**  
VAD would give smarter boundaries — chunks aligned to natural pauses. 
Fixed timer is simpler to implement and reliable. Good enough for this use case.

**Single HTML file vs component architecture**  
One file means instant deploy and zero build complexity. The tradeoff is 
that the file grows long. For a production codebase I'd split into modules. 
For a demo and interview, a single readable file is the right call.

**innerHTML rendering vs a virtual DOM**  
Each render function rewrites its section entirely. Simpler than diffing, 
fine at this scale. The tradeoff is re-attaching click listeners after every 
render — acceptable here, would use event delegation at scale.

## What I'd improve with more time

**Streaming responses** — Groq supports SSE streaming. First token would 
appear in ~200ms instead of waiting 2-3s for the full response. Biggest 
perceived latency improvement available.

**Suggestion deduplication** — Consecutive batches sometimes surface similar 
ideas because each independently reads the last 800 chars. I'd pass the 
previous batch's previews into the prompt and instruct the model to avoid 
repeating them.

**Voice activity detection** — Replace the fixed 30s timer with VAD-based 
chunking so audio boundaries align with natural pauses in speech.

**Speaker diarization** — Label transcript chunks by speaker. Makes 
suggestions much more useful in multi-person meetings.

**Markdown rendering** — Assistant responses contain markdown formatting 
that currently renders as raw text. Adding a lightweight renderer like 
marked.js would improve readability.