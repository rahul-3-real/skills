---
name: explain-me
description: Explain a topic, code snippet, file, or repository to a complete beginner using plain language, examples, FAQs, and an ongoing question-and-answer conversation. Use when the user invokes explain-me or requests a saved beginner walkthrough with follow-up teaching under _docs/explain.
---

Teach as if the learner has never encountered the subject. Help them understand why something exists, how it works, and how to reason about it. Save the lesson and subsequent conversation in Markdown so they can revisit it.

## Start with the Learning Target

If the user has not supplied a target, ask: **"What would you like to understand? You can share a topic, paste some code, point me to a file, or choose a repository."** Wait for their answer before creating an explanation.

If the target is already clear, use it immediately. Do not ask the user to repeat it or require them to assess their skill level; assume no prior knowledge. Ask a focused clarification only when the target is ambiguous or inaccessible. For a broad request such as "explain this repo," inspect the repository and start with the big picture, then organize the important concepts in learning order.

For code or repository targets, inspect relevant documentation, entrypoints, definitions, and callers before explaining. Trace a representative input through the actual code. Distinguish observed behavior from assumptions and proposed examples; do not infer how an entire repository works from filenames alone. Identify anything you could not inspect.

This is a teaching workflow. Read source code without changing its behavior, installing dependencies, or running unfamiliar application code merely to explain it. Use current authoritative documentation when external APIs need verification, and cite sources near the explanation they support.

## Explain for a Complete Beginner

- Start with the problem being solved and a familiar everyday example. Introduce the simplest useful picture before implementation details.
- Use short sentences and ordinary words. Define each technical term or acronym on first use, before relying on it elsewhere.
- Teach one concept at a time. Explain prerequisites just before they are needed and connect each new idea to the previous one.
- Use a friendly, respectful tone suitable for a child or a first-time learner. Avoid baby talk, condescension, "obviously," "simply," and claims that something is easy.
- Pair an analogy with the real mechanism. State where the analogy stops being accurate; simplification must not create a false explanation.
- Include small, concrete examples in the explanation itself. Show an input, walk through what happens, and show the output or result. Explain why each meaningful step occurs.
- For code, first describe the job in plain language, then show a small relevant excerpt with its file location. Explain unfamiliar syntax and values before using larger examples. Avoid dumping whole files or explaining every punctuation mark unless asked.
- For a repository, explain its purpose, the important parts and their jobs, and one end-to-end flow before diving into individual modules. Link to inspected source using paths relative to the saved document; add symbol names or line references when useful.
- Use a small table or Mermaid diagram only when it makes a relationship or flow easier to grasp. Explain the diagram in words too.
- End the explanation section with a short recap the learner could repeat in their own words.

## Save Location and Sequence

Resolve `_docs/explain/` relative to the target project's root, not the skill's installation directory. Use the repository root for a repository target; for a standalone topic, use the current workspace/project root. If multiple projects make the destination unclear, ask which project should contain the lesson.

Create directories as needed. Use a short lowercase kebab-case topic slug and a three-digit sequence beginning at `001`:

```text
_docs/explain/http-requests-001.md
```

Before creating a new lesson, inspect existing files and directories. For the same topic slug, select one greater than the largest existing suffix, considering both `<topic>-NNN.md` and `<topic>-NNN/`. Do not overwrite a lesson or fill an old gap. Continue numbering beyond `999` if needed.

A follow-up in the same learning conversation updates the existing lesson; it does not allocate a new sequence. When returning in a later session, read the relevant existing lesson before continuing. If several lessons could be intended, ask which one rather than appending to an arbitrary file. Create another sequence when the user requests a new or separate explanation.

## Keep One File Unless the Subject Needs Several

Keep the explanation, examples, FAQs, and all follow-up discussion for one focused topic in the same file. Do not create separate FAQ, transcript, or example files.

When the explanation would become unwieldy, or several distinct concepts each need their own examples and prerequisites, create a lesson directory instead:

```text
_docs/explain/my-app-001/
├── overview-001.md
├── http-requests-002.md
├── routing-003.md
└── database-queries-004.md
```

The directory uses the parent topic's next sequence. Inside it, use sequential suffixes in the recommended reading order, starting with `overview-001.md`. Introduce only concepts needed to understand the user's target; do not generate an entire textbook because the subject is broad.

The overview explains the big picture and links to the concept files in reading order. Every file follows the same three-section structure below and keeps its own examples, FAQs, and conversation together. Put cross-topic follow-ups in the overview and topic-specific follow-ups in the relevant concept file, linking between them rather than duplicating the exchange.

If an existing single-file lesson later needs splitting, preserve its original path as the overview and link to a new sibling `<topic>-<sequence>/` directory of concept files. Retain its existing conversation and links; do not silently delete or relocate the original document.

## Required Document Order

Use the following structure in every explanation document. Replace the descriptive guidance with actual teaching content; do not leave scaffold text in the saved lesson.

```markdown
# Human-readable topic title

## 1. Explanation

Explain the problem, introduce the idea, and walk through examples here.
Include prerequisites, relevant source links, and a short recap.
For an overview, include links to the concept files in reading order.

## 2. FAQs

### A natural beginner question about this explanation?

Give a direct, beginner-friendly answer with an example where helpful.

## 3. Follow-up Questions and Explanation

### Questions to think about

Add a few short questions about the material just explained.

### Conversation

Append actual learner and teacher exchanges here as they happen.
```

FAQs anticipate the learner's likely confusion about the content above, rather than introducing a new advanced topic. Focus on questions such as "Why do we need this?", "What happens if this part is missing?", and "How is this different from that?" Answer them in the document before asking the learner comprehension questions.

## Continue as a Teaching Conversation

After writing the initial lesson, give the user a clickable link and ask the first comprehension question in chat. Save a few short questions at the end of the document, but ask them **one at a time** so the learner can respond. Prefer explaining in their own words, predicting a small example's output, or comparing two cases over trivia or trick questions.

Wait for a response before recording a learner answer. Never invent the user's questions, answers, agreement, or understanding. Initial FAQs are anticipated questions, not a conversation transcript.

For each subsequent learner message:

1. Read the active lesson and identify the concept they are asking about or trying to explain.
2. Answer their question first. If their answer shows a misconception, gently identify the specific gap and explain it again using a different analogy or smaller example. If it is correct, explain what their answer demonstrates and add nuance only when useful.
3. Append the actual exchange to section 3 in this format, preserving previous exchanges:

   ```markdown
   **I ask:** The learner's actual question or answer.

   **You answer:** The explanation given in response, including any example.

   **You ask:** The next comprehension question, when useful.
   ```

   The learner's message may be an answer rather than a question; retain its meaning under the requested chat labels. Preserve code and important wording. Do not save unrelated conversation or sensitive values just to make the transcript exhaustive.

4. Keep the saved answer consistent with the answer delivered in chat. Add a new FAQ only if the exchange reveals a reusable clarification; keep the original conversation as well.
5. Ask one useful next question and wait. If the learner asks about a different part of the lesson, follow their curiosity before returning to pending questions.

If an earlier explanation was wrong, correct its teaching text and explicitly explain the correction in the conversation. Do not preserve an inaccurate lesson just to maintain an append-only history.

Continue across turns until the user says they understand, chooses to stop, or changes the topic. Do not claim everything is understood merely because all prepared questions have been asked. Each turn should leave the document up to date; do not generate a fictional completed conversation or keep working without learner input.

## Completion Check for Each Turn

- The document is saved under the correct project's `_docs/explain/` path with a collision-free name.
- Each explanation file keeps explanation with examples, FAQs, and follow-up conversation in that order.
- Terms and examples are understandable without assuming knowledge the lesson has not introduced.
- Repository claims are grounded in inspected code; uncertainty and illustrative examples are identified.
- Existing explanations and actual learner exchanges remain available, and relative links resolve.
- The chat response links to the saved file or overview and asks at most one next question while the lesson is active.

If writing fails, say that the explanation was not saved and identify the intended path. Do not report a saved artifact until the write succeeds.
