---
name: designdoc-creator
description: >
  Creates a DESIGN_DOC.md for a feature, system, or change using a structured template.
  Use this skill whenever the user asks to write, create, draft, or generate a design document,
  design doc, tech spec, or technical specification — even if they just say "write up a design
  for X" or "document this feature". Also trigger when the user asks to "plan out" a new system
  component and wants something more formal than a PLAN.md. Always invoke this skill proactively
  when the request is about documenting a design decision or proposal, not just implementing one.
---

# Design Doc Creator

You are helping the user create a formal `DESIGN_DOC.md` for an AI/ML project.

## Step 1: Determine the input mode

Before writing anything, ask the user this question (only this one — don't ask anything else yet):

> "Should I generate the design doc by reading existing code, docs, or context files — or would you prefer I ask you a set of questions to build and refine the design together?"

Wait for their answer. The two modes:

- **From existing material** — user provides file paths, existing docs, or says "look at the codebase". Read the relevant files, extract what you can, then fill the template. Ask targeted follow-up questions only for gaps you cannot infer.
- **Q&A interview** — ask the user a structured set of questions, round by round, to gather all the information needed.

---

## Step 2A: From existing material

Read the files the user points you to (or explore the relevant parts of the codebase). Fill in every template section using what you find. For each section you cannot confidently fill in, mark it `[TODO: needs input]` and list all gaps in a single follow-up message.

Once the user has answered the gaps, finalize the document.

---

## Step 2B: Q&A Interview

Ask questions in rounds. Pause for answers between rounds — don't dump everything at once.

**Round 1 — Core framing**
1. What is the title of this design? (One sentence describing what it proposes.)
2. What problem does this solve, and why does it need to be solved now?
3. Who are the end users or systems affected?

**Round 2 — Scope**
4. What are the specific goals?
5. What is explicitly out of scope?

**Round 3 — Solution**
6. What is your proposed solution at a high level?
7. What's the tech stack — framework, key libraries, model/provider, infrastructure components?
8. How will users interact with it? (UI flows, API surface, or CLI)
9. What are the main system components and how do they interact? What external services does it call?
10. Are there any data model or schema changes?
11. Any new or modified APIs or interfaces?

**Round 4 — Data and metrics**
11. What data will be used for training, evaluation, or inference? Any licensing, PII, or compliance concerns?
12. How will you measure success? Primary metrics, target thresholds, and the baseline you're beating.

**Round 5 — Safety and operations**
13. What guardrails are needed? (Input validation, output filtering, confidence thresholds, fallbacks, rate limiting, responsible AI concerns.)
14. What gets logged in production — inputs, outputs, latency, errors? How will you trace a failed request? How will you detect model or data drift, and what's the response plan?

**Round 6 — Execution**
16. Is this serving a model in real time? If so: hardware, latency budget, throughput, estimated cost per call. (Skip if offline batch.)
17. Is training, fine-tuning, or prompt optimization involved? If so: experiment tracking tooling, what gets logged, how reproducibility is maintained.
18. Is this a POC? If so: go/no-go thresholds and timeline.
19. What's the MVP — the minimum that makes this useful in production? What would go in later phases?
20. How will this be tested and validated?

**Round 7 — Risk and context**
21. What alternatives did you consider, and why did you reject them?
22. What are the main risks and mitigations? Any external service or team dependencies worth flagging?
23. Are there any open questions?
24. Any additional background or references for the appendix?

After all answers are collected, confirm: "Ready to write the design doc — anything else before I generate it?"

---

## Step 3: Write DESIGN_DOC.md

Read the full template from `skills/designdoc-creator/DESIGN_DOC_TEMPLATE.md` and populate every section with real content. Never leave template placeholder text. For sections that genuinely don't apply (Inference Requirements, Experiment Tracking, POC Success Criteria), remove them entirely rather than writing "N/A".

For the **Revision History** table, use today's date and the user's email from the system context if available.

Save the output to `DESIGN_DOC.md` in the current working directory.

---

## Quality bar

- Design is concise, clear, and actionable. Avoid explanations and lengthy descriptions. Use bullet points and diagrams where helpful and keep paragraphs short.
- Title is one sentence that captures the proposal, not just a noun ("Add caching layer to user service" not "Caching")
- Background explains *why*, not just *what*
- Goals are specific and measurable where possible
- Non-Goals preempt scope creep — they should be things someone might reasonably assume are in scope
- Proposal section could be handed to a new engineer with no other context
- Evaluation Metrics include a concrete baseline and numeric thresholds, not just metric names. Non-standard metrics should be defined.
- Data section notes any PII, licensing, or compliance constraints — even if the answer is "none identified"
- Guardrails name specific failure modes, not just "input validation will be added"
- Observability & Monitoring specifies what is logged and at what level; drift detection thresholds are concrete
- Risks section is honest, not defensive
- Open Questions are actual unresolved issues, not rhetorical ones
- It's ok to leave >TBD and >TODO sections if there is not enough information at the moment.

If any section is vague or thin after gathering input, ask one targeted follow-up before writing rather than padding with generic text.
