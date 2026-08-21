# B2B thought-leadership content agent

**A human-in-the-loop content automation system built on Make.com and the Gemini API.**

## The problem

A B2B thought leader needs a steady stream of on-brand LinkedIn content — scheduled posts, timely reactions to industry news, thoughtful replies on other people's posts, and outreach when a genuine speaking opportunity appears. Doing this by hand is a research-and-writing tax on someone whose actual job is the expertise, not the writing. Doing it with a naive "LLM writes, bot posts" pipeline is worse: it risks off-brand content, fabricated claims, and reputational damage the moment nobody's watching.

The brief was to design something in between: an agent that does the research and the writing, but never has the authority to publish. Every output is a draft, saved to Drive, with a human notified — and only a human decides what goes live.

## Architecture

Four independent workflows, each triggered a different way, all converging on the same pattern:

**research → (optional) classify → generate (schema-enforced) → save draft → notify human**

| Workflow | Trigger | What it produces |
|---|---|---|
| Scheduled post | Cron (Tue/Thu 07:00) | A standalone post grounded in recent industry news |
| Breaking-news monitor | RSS poll | A timely reaction — but only if a cheap classifier judges the story genuinely relevant |
| Engagement comment | Webhook | A specific, non-generic reply to someone else's post |
| Speaker radar | Webhook | A warm, event-specific outreach draft — but only past a confidence threshold, so a stray use of the word "speaker" doesn't fire a pitch |

Two workflows carry a relevance gate before the LLM ever writes anything, because the cost of a wrong "yes" here isn't a bad draft — it's a founder getting pitched to apply for an event that was actually just a thank-you post. The gate is a separate, cheaper model call (Gemini Flash-Lite) making a binary judgment with a confidence score, and nothing downstream fires unless it clears a threshold.

Brand voice and system behavior live in two Google Docs, fetched fresh from Drive on every single run rather than baked into the workflow. That means a client can edit their own brand guidelines and see the change reflected on the next run — no redeploy, no code change, no engineer in the loop for a tone tweak.

## Key engineering decisions

**Structured output enforcement, not just prompting.** The first version of this system asked the model — via prompt instructions — to "return only this JSON shape." It worked most of the time. Then it didn't: Gemini returned a perfectly valid JSON object with a schema it invented on its own, and the pipeline silently produced blank drafts downstream, because nothing mapped to the fields the rest of the workflow expected. The fix was `responseSchema`, a Gemini API parameter that structurally constrains the output — not "please return this shape," but "the API will not return anything else." This is the single most important lesson the build surfaced: for anything downstream that parses an LLM's output programmatically, schema enforcement at the API level beats prompt-level instruction every time.

**Idempotency over hope.** The news-monitoring workflow polls the same feed repeatedly. Nothing stops it from seeing the same article twice. Rather than trusting the LLM or the workflow logic to notice a duplicate, a Make Data Store keyed on an MD5 hash of each article's URL enforces it structurally — the write step itself refuses a second insert of a seen key. Same principle as the schema fix: push correctness into a mechanism that can't drift, not into a judgment call that might.

**Confidence gates as noise filters, not just relevance filters.** Both classifier-gated workflows require the model to return a numeric confidence alongside its yes/no judgment, and the gate checks both. A `true` at 0.51 confidence and a `true` at 0.98 confidence are treated differently — only the latter reaches the LLM generation step. This was tuned by testing both a genuine call-for-speakers post and a deliberately similar-but-irrelevant post (a thank-you message that happened to contain the word "speaker") side by side, confirming the gate discriminates on meaning, not keyword presence.

**Provider-agnostic by design.** The drafting call is a plain HTTP request to a REST endpoint with a JSON body — not a vendor SDK. Swapping Gemini for Claude, or Google Docs for OneDrive/Word, is a one-module change: same system prompt, same schema, same downstream logic. This was a deliberate architecture decision recorded up front, not a retrofit.

**Human-in-the-loop as an architectural constraint, not a feature flag.** There is no code path in any of the four workflows that reaches a "post" or "send" action. The furthest any workflow goes is: save a Google Doc, email a link to it. Even the speaker-outreach pitch is explicitly framed in its own content as something to send by hand. This wasn't bolted on after the fact — every workflow was designed to structurally stop at that boundary.

## Tech stack

Make.com (orchestration) · Gemini API (`gemini-3.6-flash` generation, `gemini-3.5-flash-lite` classification) · Google Drive, Docs, Gmail APIs · Make Data Store (idempotency) · JSON Schema (structured output enforcement)