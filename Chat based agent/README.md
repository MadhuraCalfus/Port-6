# Company Policy Assistant — Chat-Based Agent

A conversational AI assistant built on **Oracle AI Agent Studio** that lets employees ask company
policy questions in plain English across a multi-turn chat, and returns answers grounded in — and
cited to — an internal library of policy PDFs.

---

## About This Export

The three artifact files in this folder are **byte-for-byte identical** to those in the
`Latest - Menu based agent` export. Both snapshots were taken from the same published Agent Studio
application after it had been iterated in place, so this README documents the configuration that
is actually present in `src/` here.

If you have an earlier export that captures the original chat-only configuration, drop it in and
this document can be rewritten against it.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [What This Project Delivers](#what-this-project-delivers)
- [Project Contents](#project-contents)
- [The Workflow — End to End](#the-workflow--end-to-end)
- [Node Types Used](#node-types-used)
- [Node-by-Node Explanation](#node-by-node-explanation)
- [Agents Used](#agents-used)
- [The Document Tool](#the-document-tool)
- [Conversational Behaviour](#conversational-behaviour)
- [Sample Conversation](#sample-conversation)
- [How Accuracy Is Enforced](#how-accuracy-is-enforced)
- [Setup & Deployment](#setup--deployment)
- [Screenshots](#screenshots)
- [Limitations & Future Enhancements](#limitations--future-enhancements)

---

## Problem Statement

### Mission

> Companies deal with large volumes of internal documents — HR policies, product manuals,
> onboarding guides, SOPs — and employees waste time hunting for answers buried in PDFs.
> Build an AI assistant that lets users ask questions in plain English and get accurate,
> cited answers from a document library.

### The pain in practice

Company policies typically live as dozens of PDFs on an HRMS portal. An employee wondering
*"How many days do I have to submit a travel claim?"* has to guess which of ~16 documents applies,
open a 20–40 page PDF, `Ctrl+F` for the right clause, and interpret dense policy language alone —
or give up and raise an HR ticket, adding to a queue that never shrinks.

A naive chatbot makes this **worse**, not better: a confidently hallucinated policy answer is more
damaging than no answer, because employees act on it. Any assistant in this space must be
verifiable, not merely fluent.

### Requirements this project had to meet

| # | Requirement | Where it is solved |
|---|---|---|
| 1 | Accept plain-English questions in a natural chat | REST-triggered workflow behind the Agent Studio chat experience |
| 2 | Answer only from the document library | `DOCUMENT`-type RAG tool as the sole authoritative source |
| 3 | Cite every answer | `sources[]` is a required field in the agent's output schema |
| 4 | Hold context across turns | Chat history enabled on the routing and formatting nodes |
| 5 | Never invent a policy or a citation | Layered grounding rules across agent, schema, and formatter |
| 6 | Admit when the answer isn't there | Explicit not-found path returning empty sources |
| 7 | Stay inside scope | Relevance gate before retrieval, with a friendly redirect branch |

---

## What This Project Delivers

A published Oracle AI Agent Studio application made of **one document tool, one reusable agent,
and one workflow of seven nodes**.

| Property | Value |
|---|---|
| Platform | Oracle AI Agent Studio (Fusion Applications) |
| Family / Product | `HCM` / `GLOBAL_HUMAN_RESOURCES` |
| Namespace | `HCM.GLOBAL_HUMAN_RESOURCES` |
| Default model | `ORA_MODEL_CONFIG_PREMIUM_OPEN_AI_GPT_4_1_MINI` |
| Workflow architecture | `data_pipeline` |
| Trigger | `REST` |
| Access modifier | `public` |
| Human approval | Not required |
| Status | `PUBLISHED` |
| Document source | 16 policy PDFs exported from the **greytHR** HRMS portal |

The assistant runs as a chat surface: the user types a question, the workflow classifies it,
retrieves from the policy library, and returns a formatted, cited answer. Conversation context
carries forward, so follow-up questions like *"Who approves it?"* resolve correctly against the
previous turn.

---

## Project Contents

```
Chat based agent/
├── README.md
└── src/
    ├── tools/
    │   └── company_policy_knowledge_base.tool     # DOCUMENT tool — 16 policy PDFs
    ├── agents/
    │   └── company_policy_assistant.agent         # WORKER agent — retrieval + reasoning
    └── workflows/
        └── company_policy_chat_assistant.wf       # data_pipeline — 7 nodes
```

Each file is a JSON export from Oracle AI Agent Studio, re-imported through the Studio's import
screen. They are configuration artifacts, not runnable code — there is nothing to install or compile.

| File | Code | Type |
|---|---|---|
| [company_policy_knowledge_base.tool](src/tools/company_policy_knowledge_base.tool) | `COMPANY_POLICY_KNOWLEDGE_BASE` | `DOCUMENT` |
| [company_policy_assistant.agent](src/agents/company_policy_assistant.agent) | `COMPANY_POLICY_ASSISTANT` | `WORKER` |
| [company_policy_chat_assistant.wf](src/workflows/company_policy_chat_assistant.wf) | `COMPANY_POLICY_CHAT_ASSISTANT` | `data_pipeline` |

---

## The Workflow — End to End

**Workflow code:** `COMPANY_POLICY_CHAT_ASSISTANT`
**Root node:** `start` · **Convergence target:** `end` · **Nodes:** 7 · **Branches:** 2

```
                          ┌───────────┐
                          │   START   │  type: START
                          └─────┬─────┘
                                │  user message + chat history
                                ▼
                    ┌───────────────────────┐
                    │      CHECK_INPUT      │  type: LLM
                    │  determines what the  │  → { isRelevant, resolvedQuestion }
                    │  user is really       │
                    │  asking, and whether  │
                    │  it is in scope       │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │       IF_CHECK        │  type: CONDITION
                    │   isRelevant == true? │
                    └───┬───────────────┬───┘
                 true   │               │   false
                        ▼               ▼
      ┌───────────────────────┐   ┌──────────────────────────┐
      │ COMPANY_POLICY_AGENT  │   │ IRRELEVANT_INPUT_HANDLING│  type: LLM
      │      type: AGENT      │   │  friendly out-of-scope   │
      │                       │   │  redirect + topic list   │
      │  invokes the agent    │   └────────────┬─────────────┘
      │  COMPANY_POLICY_      │                │
      │  ASSISTANT, which     │                │
      │  queries the RAG tool │                │
      │                       │                │
      │  → { answer,          │                │
      │      sources[],       │                │
      │      follow_up_       │                │
      │      options[3] }     │                │
      └───────────┬───────────┘                │
                  ▼                            │
      ┌───────────────────────┐                │
      │       LLM_AGENT       │  type: LLM     │
      │  presentation only:   │                │
      │  branding, answer     │                │
      │  layout, suggestions, │                │
      │  sources block        │                │
      └───────────┬───────────┘                │
                  │                            │
                  └────────────┬───────────────┘
                               ▼
                          ┌───────────┐
                          │    END    │  type: END
                          └───────────┘
```

### Execution trace of a single turn

1. The user's message enters at **START**, which hands control directly to `CHECK_INPUT`.
2. **CHECK_INPUT** reads the current message together with the chat history and emits two values:
   whether the question falls inside the assistant's scope, and the question that should actually
   be answered this turn.
3. **IF_CHECK** evaluates that boolean and selects one of two branches.
4. On the in-scope branch, **COMPANY_POLICY_AGENT** invokes the worker agent with the resolved
   question. The agent queries the document tool and returns structured JSON.
5. **LLM_AGENT** renders that JSON into the reply the user sees — branding, formatted answer,
   suggested next questions, and the sources block.
6. On the out-of-scope branch, **IRRELEVANT_INPUT_HANDLING** produces a polite redirect back to
   supported topics.
7. Both branches converge on **END**.

Only one branch executes per turn, and the two never interleave.

---

## Node Types Used

A `data_pipeline` workflow in Oracle AI Agent Studio is assembled from typed nodes. This project
uses **five node types** across its seven nodes.

### `START`

The pipeline entry point, declared as the workflow's `rootNode`. It holds no logic — only a
single `success` outcome pointing at the first working node. Exactly one per workflow.

**Used by:** `START`

### `LLM`

A direct model call, and the most-used node type here. It takes a `systemPrompt` and a `prompt`
as string inputs, both supporting `{{...}}` template expressions that pull in system context
(`$context.$system.$inputMessage`, `$context.$system.$chatHistory`) or another node's output
(`$context.$nodes.<CODE>.$output.<field>`).

Two optional behaviours matter in this workflow:

- **`outputSpecification`** — supply a JSON Schema and the node returns validated structured JSON
  that downstream nodes address field by field. Leave it empty and the node returns plain text
  meant for the user.
- **Metadata flags** — `chatHistoryEnabled` controls whether prior turns are visible to the node,
  and `answerInUserLanguage` controls multilingual replies.

**Used by:** `CHECK_INPUT`, `LLM_AGENT`, `IRRELEVANT_INPUT_HANDLING`

| LLM node | `outputSpecification` | `chatHistoryEnabled` | `answerInUserLanguage` |
|---|---|---|---|
| `CHECK_INPUT` | JSON Schema | `true` | `true` |
| `LLM_AGENT` | *(empty — plain text)* | `true` | `true` |
| `IRRELEVANT_INPUT_HANDLING` | *(empty — plain text)* | `false` | `true` |

The differing `chatHistoryEnabled` values are a deliberate part of the conversational design.
`CHECK_INPUT` needs history to resolve pronouns and references. `IRRELEVANT_INPUT_HANDLING`
runs **without** it, so an off-topic message is handled on its own terms with no risk of the
model dragging a previous policy topic into a redirect that has nothing to do with policy.

### `CONDITION`

A branching node. It evaluates a single `condition` string input and routes to different nodes
through `true` and `false` entries in its `outcomes` map. A `convergenceTargetId` declares where
the branches rejoin — here, `end`.

**Used by:** `IF_CHECK`

### `AGENT`

Invokes a separately published, reusable agent by its `agentCode`, instead of prompting a model
inline. The node supplies the agent's declared inputs and receives its structured output. This is
what separates a **reusable capability** (the agent, with its tools and output contract) from
**one workflow's orchestration** (this pipeline).

**Used by:** `COMPANY_POLICY_AGENT` → invokes `COMPANY_POLICY_ASSISTANT`

### `END`

The pipeline exit point, where both branches converge. Exactly one per workflow.

**Used by:** `END`

### Summary table

| Node code | Node type | Role in the flow |
|---|---|---|
| `START` | `START` | Entry point |
| `CHECK_INPUT` | `LLM` | Works out the real question and whether it is in scope |
| `IF_CHECK` | `CONDITION` | Branch on `isRelevant` |
| `COMPANY_POLICY_AGENT` | `AGENT` | Calls the worker agent, which queries the RAG tool |
| `LLM_AGENT` | `LLM` | Formats the structured result for display |
| `IRRELEVANT_INPUT_HANDLING` | `LLM` | Graceful out-of-scope redirect |
| `END` | `END` | Exit point |

---

## Node-by-Node Explanation

### 1. `CHECK_INPUT` — the router *(type: `LLM`)*

The first decision point of every turn. It reads `{{$context.$system.$inputMessage}}` and
`{{$context.$system.$chatHistory}}` and returns validated JSON:

```json
{
  "isRelevant": true,
  "resolvedQuestion": "What is the leave policy?"
}
```

Its schema marks both fields required and sets `additionalProperties: false`, so the node cannot
drift into returning prose that downstream nodes would fail to parse.

**Determining the question.** For a normal question, `resolvedQuestion` is the user's input
verbatim. A hard rule forbids the chat history from overriding it — this prevents the common
failure where a short question such as `"POSH policy?"` is silently rewritten into a topic from an
earlier turn. History is consulted only in one narrow case: when the message is exactly a
selection token (`Option 1`/`Option 2`/`Option 3`/`1`/`2`/`3`), in which case the node looks back
at the **most recent** assistant reply containing `"You may also want to know:"` and expands the
selection into the full question text. Older suggestion lists are explicitly out of bounds.

**Determining relevance.** `isRelevant` is `true` for company and HR policy, leave, attendance,
benefits, travel, expenses, workplace procedures, and company rules; `false` otherwise. Even when
`false`, `resolvedQuestion` still carries the user's input forward. The node never answers the
question — it only classifies and resolves.

### 2. `IF_CHECK` — the scope gate *(type: `CONDITION`)*

```
condition: {{$context.$nodes.CHECK_INPUT.$output.isRelevant}}
outcomes:  true  → COMPANY_POLICY_AGENT
           false → IRRELEVANT_INPUT_HANDLING
convergenceTargetId: end
```

Gating **before** retrieval, rather than letting the agent decline afterwards, is deliberate: it
costs no tool call, it is more reliable than burying a scope judgement inside a long retrieval
prompt, and it guarantees an off-topic message can never reach the knowledge base and provoke an
improvised answer.

### 3. `COMPANY_POLICY_AGENT` — retrieval and reasoning *(type: `AGENT`)*

Invokes the published agent `COMPANY_POLICY_ASSISTANT` with:

```
message = {{$context.$nodes.CHECK_INPUT.$output.resolvedQuestion}}
```

The agent runs its own tool-calling loop against the document tool and returns
`{answer, sources[], follow_up_options[3]}`, mirrored on the node as its `outputSpecification`.
Its single `success` outcome routes to the formatter.

### 4. `LLM_AGENT` — the presentation layer *(type: `LLM`)*

A deliberately **non-reasoning** node. It receives four template values — the resolved question,
the answer, the sources array, and the suggested follow-ups — and does nothing but format them.
It is explicitly forbidden from adding, removing, or altering any fact, source, section, or
question, and from mentioning JSON, nodes, tools, agents, or workflows to the user.

Layout it enforces:

```
**Calfus**
**[Policy Name — taken from documentName]**

<answer, rendered as a paragraph, bullets, numbered steps, or a Markdown table
 depending on the shape of the content and what the user asked for>

You may also want to know:

**Option 1:** <suggested question 1, verbatim>
**Option 2:** <suggested question 2, verbatim>
**Option 3:** <suggested question 3, verbatim>

**Sources**
* Document: <documentName>
* Section: <section, if available>
* Page / Heading / Chapter / Table / Clause / Appendix: <only when actually present>
```

A 20-point validation checklist runs before output, verifying that the answer is unchanged, every
source is displayed, every section stays bound to its own document, numbering is preserved
character-for-character, no section was invented or inferred, and suggested questions are copied
exactly rather than regenerated.

Answer shape is chosen to match the content: a short paragraph for a simple explanation, numbered
steps for a procedure, bullets for independent facts, headings for multiple categories, and a
Markdown table when the user asks for one or the content is naturally comparative. Content is
never forced into a table that doesn't suit it.

### 5. `IRRELEVANT_INPUT_HANDLING` — the graceful exit *(type: `LLM`)*

Handles the `false` branch. Instead of a blunt refusal it briefly acknowledges the message,
explains the assistant's scope, lists what it *can* help with — leave and holidays, attendance and
working hours, work from home, travel and expense reimbursement, benefits, code of conduct,
onboarding, workplace procedures — and invites a relevant question. It replies in the user's
language, uses short sections or bullets rather than a wall of text, acknowledges frustration if
the user seems stuck, and is barred from attempting the out-of-scope question or revealing any
internal machinery.

---

## Agents Used

The project uses **one reusable agent** plus **three inline LLM nodes** that act as
single-purpose agents inside the pipeline.

### Reusable agent — `COMPANY_POLICY_ASSISTANT`

**File:** [company_policy_assistant.agent](src/agents/company_policy_assistant.agent)

| Property | Value | Meaning |
|---|---|---|
| Agent code | `COMPANY_POLICY_ASSISTANT` | Referenced by the `AGENT` node's `agentCode` |
| Agent type | `WORKER` | Performs the task itself; it does not delegate to sub-agents |
| Reusable | `true` | Can be invoked from any workflow, not just this one |
| Max interactions | `5` | Tool-call budget per turn — caps retrieval loops |
| Declared input | `Question` (string) | The question to research |
| Tool attached | `COMPANY_POLICY_KNOWLEDGE_BASE` | Its only knowledge source |
| Summarization mode | `Default` | Platform summarizer over tool responses |
| Status | `PUBLISHED` | Available for workflows to invoke |

A `WORKER` agent is the executing unit of Agent Studio: it owns a role, a set of tools, and an
output contract, and it runs a bounded tool-calling loop until it can satisfy that contract.
Because `reusableFlag` is `true`, this same agent could be surfaced in Slack or dropped into a
different workflow without modification — the conversational orchestration lives in the workflow,
not in the agent.

#### Agent role

> You are a Company Policy Assistant for employees. Your role is to help employees find accurate
> and relevant information from the company's approved policy documents… You use the Company
> Policy Knowledge Base document tool as your primary and authoritative source of information.

#### Output contract

The agent is constrained by a JSON Schema, so the workflow receives predictable data instead of
free text:

```jsonc
{
  "answer": "string",                 // grounded answer, retrieved content only
  "sources": [                        // documentName is REQUIRED on every entry
    {
      "documentName": "string",       // exact policy document name
      "section": "string",            // exact section number/title, only when explicit
      "page": "string",
      "heading": "string",
      "chapter": "string",
      "table": "string",
      "clause": "string",
      "appendix": "string",
      "sourceIdentifier": "string"
    }
  ],
  "follow_up_options": ["string", "string", "string"]   // exactly 3 — minItems = maxItems = 3
}
```

Because `sources` is a first-class schema field with `documentName` required, **citations are
structurally guaranteed** rather than left to the model's discretion.

#### Instruction design

The agent prompt runs to 17 numbered sections. The load-bearing rules:

| § | Rule |
|---|---|
| 1 | The Knowledge Base is the primary and authoritative source; never answer policy from general knowledge |
| 2 | Answer only from retrieved content; state explicitly when the information is partial |
| 3 | `sources` is an array of objects with `documentName`, plus `section` only when explicitly present |
| 4 | Include page / heading / chapter / clause metadata **only** when the retrieved chunk provides it |
| 5 | Not found → fixed message, `sources: []`, `follow_up_options: []` |
| 6 | Conflicting documents → surface both, never silently pick one, recommend confirming with HR |
| 7 | May combine multiple sections or documents, but must cite each |
| 8 | **Follow-up handling** — resolve pronouns from conversation context ("Who approves *it*?") |
| 9 | **Clarification** — ask one short question when the topic is genuinely ambiguous |
| 10 | Out-of-scope topics → decline; never substitute outside knowledge |
| 11 | Never expose or infer salaries, credentials, or confidential employee data |
| 12 | Clear, concise, professional, employee-friendly; bullets or steps for complex answers |
| 12A | **Section-extraction protocol** — inspect the supporting passage for an explicit heading before emitting `section`; omit rather than infer |
| 13 | Accuracy over completeness — never hallucinate policies, sources, or section numbers |
| 14 | Generate exactly 3 suggested follow-up questions, grounded in retrieved content |
| 15 | Answer only the resolved question; never reuse the previous answer for a new question |
| 16 | A 10-step source-selection and section-extraction procedure |
| 17 | Final output shape — `section` lives inside each source object; no top-level `sections` field |

Sections 12A and 16 are by far the longest parts of the prompt. That is intentional: a wrong
document name is obvious to a reader, but a plausible-but-wrong section number is not, and the
assistant's entire value rests on citations being trustworthy.

### Inline agents — the three `LLM` nodes

These are not separately published agents; they are single-purpose prompts executed inline by the
pipeline. Each has one job and no tools.

| Node | Job | Returns |
|---|---|---|
| `CHECK_INPUT` | Determine the real question and classify relevance | Structured JSON |
| `LLM_AGENT` | Format the agent's structured output for display | Plain text / Markdown |
| `IRRELEVANT_INPUT_HANDLING` | Redirect off-topic messages | Plain text / Markdown |

**Why the answering agent and the formatting node are split:** mixing retrieval reasoning with
presentation formatting in one prompt degrades both — a model asked to style an answer starts
quietly "improving" it. Separating them means the agent owns truth and the formatter owns
appearance, and the formatter is explicitly forbidden from touching content.

---

## The Document Tool

**File:** [company_policy_knowledge_base.tool](src/tools/company_policy_knowledge_base.tool)

| Property | Value |
|---|---|
| Tool code | `COMPANY_POLICY_KNOWLEDGE_BASE` |
| Type | `DOCUMENT` (retrieval-augmented generation) |
| Collection name | `Company Policies` |
| Attachments | **16 PDFs**, ~7.7 MB total |
| User input required | `false` — the agent queries it autonomously |
| Status | `PUBLISHED` |

A `DOCUMENT` tool is Oracle AI Agent Studio's built-in RAG capability. The PDFs were downloaded
from greytHR and uploaded directly; the platform handles parsing, chunking, embedding, and
semantic retrieval. No external vector database, embedding pipeline, or custom code is involved.

### Document library

| # | Document | Covers |
|---|---|---|
| 1 | Calfus India Employee Handbook.pdf | Overall employment terms, general policies |
| 2 | All IT Policies.pdf | Asset use, security, acceptable use, incidents |
| 3 | HR Security Policy.pdf | People-side information security obligations |
| 4 | POSH Policy.pdf | Prevention of Sexual Harassment — complaint process, ICC |
| 5 | Whistleblower and Non-Retaliation Policy.pdf | Reporting channels, protection from retaliation |
| 6 | Disciplinary Policy.pdf | Misconduct categories, disciplinary process |
| 7 | PIP Process.pdf | Performance Improvement Plan lifecycle |
| 8 | Confirmation Policy.pdf | Probation and confirmation criteria |
| 9 | BGV Policy.pdf | Background verification checks and process |
| 10 | Travel Policy.pdf | Business travel, entitlements, claim rules |
| 11 | India Relocation Policy.pdf | Relocation eligibility and reimbursement |
| 12 | Transfer Policy.pdf | Inter-location and inter-project transfers |
| 13 | Certification Reimbursement Policy_Ver1.0.pdf | Certification funding and claim conditions |
| 14 | Calfus Crew — Employee Referral Policy_1.2.pdf | Referral eligibility, bonus, payout |
| 15 | Rewards and Recognition Policy.pdf | Award categories, nomination, cadence |
| 16 | Communication Policy.pdf | Internal and external communication standards |

---

## Conversational Behaviour

The assistant is designed as a chat, not a search box. Several parts of the configuration exist
purely to make multi-turn conversation work.

**Context carry-over.** `CHECK_INPUT` and `LLM_AGENT` both run with `chatHistoryEnabled: true`,
and agent instruction §8 requires pronouns in follow-ups to be resolved against the previous turn:

```
User: "What is the annual leave policy?"
Assistant: <cited answer>
User: "Who needs to approve it?"        →  "it" resolves to annual leave
```

**Protection against over-resolution.** The counterweight to context carry-over is an explicit rule
that history must **never** override a normal question. Without it, a terse new question gets
absorbed into the previous topic — a failure mode that is hard to notice because the answer still
reads plausibly.

**Clarification instead of guessing.** Agent instruction §9 requires a short clarifying question
when the topic is genuinely ambiguous:

```
User: "What is the policy?"
Assistant: "Which policy would you like to know about — for example, leave,
            attendance, work from home, or travel?"
```

The same rule forbids unnecessary clarification when intent is already clear, so the assistant
does not turn into an interrogation.

**Suggested next questions.** Every grounded answer closes with three suggested follow-ups,
generated from the retrieved content only. They must differ from the current question and from
one another, must never be invented to fill the quota, and are suppressed entirely when nothing
relevant was retrieved. They are suggestions, not a cage — a free-text question on a different
aspect of the same policy is answered normally.

**Starter questions.** The chat experience opens with two seeded prompts so the user is never
faced with a blank box:

- *"How do I file a POSH complaint?"*
- *"What is the leave policy?"*

**Multilingual replies.** All three LLM nodes carry `answerInUserLanguage`, so the assistant
responds in the language the user writes in, while retrieval still runs against the
English-language document library.

---

## Sample Conversation

**User:** `How do I file a POSH complaint?`

```
**Calfus**
**POSH Policy**

To file a complaint under the POSH Policy:

1. Submit a written complaint to the Internal Committee (IC) within the
   timeline specified in the policy.
2. Include the details of the incident, the parties involved, and any
   supporting evidence or witnesses.
3. The Internal Committee acknowledges the complaint and initiates inquiry
   as per the process defined in the policy.

You may also want to know:

**Option 1:** Who are the members of the Internal Committee?
**Option 2:** What is the timeline for the inquiry process?
**Option 3:** What protection is available against retaliation?

**Sources**
* Document: POSH Policy
* Section: Complaint Procedure
```

**User:** `What protection is available against retaliation?`

The question is answered on its own terms, potentially drawing on
`Whistleblower and Non-Retaliation Policy.pdf` as well as the POSH Policy — with each contributing
document cited separately.

**User:** `What's the weather today?`

```
I'm here to help with company policy questions, so I can't help with the weather —
but I'd be glad to help with things like:

* Leave and holidays
* Attendance and working hours
* Work from home and remote work
* Travel and expense reimbursement
* Benefits, onboarding, and code of conduct

What would you like to know?
```

**User:** `What is the company's stock option policy?` *(not in the library)*

```
I couldn't find information about this topic in the available company policy documents.
```

No sources are shown and no suggestions are offered, because nothing was retrieved.

---

## How Accuracy Is Enforced

Grounding is enforced at **four independent layers**, so a failure at one is caught by the next.

**Layer 1 — Retrieval.** A `DOCUMENT`-type tool is the only knowledge source, and the agent is
instructed never to answer policy questions from general knowledge.

**Layer 2 — Schema.** `sources` is a required output field, with `documentName` required on every
entry. The agent cannot emit a schema-valid answer without naming its source.

**Layer 3 — Agent instructions.** The section-extraction protocol (§12A and §16) explicitly
prohibits inferring a section from the answer text, from the document title, or from a numbered
list in the answer; forbids inventing section numbers or titles; requires omitting `section`
rather than guessing; and requires preserving original numbering and wording exactly.

**Layer 4 — Formatter.** The presentation node cannot introduce facts. It may only reformat what
the agent returned, must display every provided source and section, must keep each section bound
to its own document, and runs a 20-point validation checklist before output.

**Honest failure.** When the library has no answer, the assistant says so in a fixed message with
empty sources rather than improvising. When two documents conflict, it surfaces both, cites each,
and recommends confirming with HR rather than silently picking a winner.

---

## Setup & Deployment

### Prerequisites

- Oracle Fusion Applications environment with **AI Agent Studio** enabled
- Privileges to create tools, agents, and workflows in `HCM.GLOBAL_HUMAN_RESOURCES`
- The 16 policy PDFs, exported from greytHR or an equivalent HRMS

### Import order

Order matters — each artifact depends on the previous one.

1. **Document tool.** Import [company_policy_knowledge_base.tool](src/tools/company_policy_knowledge_base.tool),
   or create a `DOCUMENT` tool named *Company Policy Knowledge Base* and upload the 16 PDFs into a
   collection called *Company Policies*. Wait for ingestion to finish, then **Publish**.
2. **Agent.** Import [company_policy_assistant.agent](src/agents/company_policy_assistant.agent).
   Confirm `COMPANY_POLICY_KNOWLEDGE_BASE` is attached under Tools, and that the output schema
   carries `answer`, `sources`, and `follow_up_options`. **Publish**.
3. **Workflow.** Import [company_policy_chat_assistant.wf](src/workflows/company_policy_chat_assistant.wf).
   Verify all seven nodes are present and that `IF_CHECK` routes `true` to `COMPANY_POLICY_AGENT`
   and `false` to `IRRELEVANT_INPUT_HANDLING`. **Publish**.

### Testing checklist

| Test | Expected result |
|---|---|
| Ask a starter question | Cited answer naming the document and section |
| Ask a follow-up using a pronoun | Resolved against the previous turn, not misread |
| Ask a fresh unrelated policy question | Answered on its own terms, not absorbed into the old topic |
| Ask a vague question ("what is the policy?") | One short clarifying question |
| Ask something off-topic | Friendly redirect with a topic list, no policy content |
| Ask about a policy not in the library | Honest not-found message with no sources |
| Ask a question answered by two documents | Both cited separately |

### Adapting to a different document library

The architecture is domain-agnostic — product manuals, onboarding guides, and SOPs work the same
way. To repoint it:

1. Replace the PDFs in the document tool.
2. Update the tool description so the agent knows what it now covers.
3. Update the topic lists in agent prompt §1, in `CHECK_INPUT`'s relevance criteria, and in
   `IRRELEVANT_INPUT_HANDLING`.
4. Update the branding line in `LLM_AGENT` §1 and the two starter questions.

The citation and grounding logic need no changes.

---

## Screenshots

> Place images in `docs/screenshots/` inside this folder.

### Workflow design

**Full workflow canvas — all seven nodes and both branches**

![Workflow canvas](docs/screenshots/01-workflow-canvas.png)

**`CHECK_INPUT` — LLM node with structured output**

![Check input node](docs/screenshots/02-check-input-node.png)

**`IF_CHECK` — CONDITION node branching on `isRelevant`**

![If check node](docs/screenshots/03-if-check-node.png)

**`COMPANY_POLICY_AGENT` — AGENT node invoking the worker agent**

![Agent node](docs/screenshots/04-agent-node.png)

**`LLM_AGENT` — the formatting node**

![Formatter node](docs/screenshots/05-formatter-node.png)

**`IRRELEVANT_INPUT_HANDLING` — the out-of-scope branch**

![Irrelevant input node](docs/screenshots/06-irrelevant-input-node.png)

### Agent and tool configuration

**Agent definition and instructions**

![Agent configuration](docs/screenshots/07-agent-config.png)

**Structured output schema — `answer`, `sources`, `follow_up_options`**

![Output schema](docs/screenshots/08-output-schema.png)

**Document tool with the 16 uploaded policy PDFs**

![Document tool](docs/screenshots/09-document-tool.png)

### Working demo

**A cited answer in the chat**

![Cited answer](docs/screenshots/10-cited-answer.png)

**A multi-turn follow-up resolved from context**

![Follow-up](docs/screenshots/11-followup.png)

**Out-of-scope handling**

![Out of scope](docs/screenshots/12-out-of-scope.png)

**Information-not-found handling**

![Not found](docs/screenshots/13-not-found.png)

---

## Limitations & Future Enhancements

### Current limitations

- **Section granularity depends on the source PDFs.** Documents without explicit numbered
  headings yield document-level citations only — by design, since inventing a section is worse
  than omitting one.
- **English-centric library.** The assistant replies in the user's language, but retrieval runs
  against English-language documents.
- **A tool-call budget of 5 interactions** per turn may constrain questions that need to
  synthesise across many documents at once.
- **No deep links.** Citations name the document and section but do not link to a page anchor.
- **India-specific corpus.** A multi-geography rollout would need region filters so employees
  don't receive another region's policy.
- **Stale error-handler reference.** `LLM_AGENT`'s metadata carries an `errorNodeId` pointing at a
  node ID that no longer exists in the pipeline, and `errorHandlers` is empty — a leftover from an
  earlier iteration. Harmless today, but worth clearing or replacing with a real error branch.

### Possible enhancements

| Enhancement | Value |
|---|---|
| Deep links into the source PDF page | One-click verification of any citation |
| Region / entity / role metadata filters | Correct policy variant per employee |
| Confidence indicator on answers | Signals when a human check is warranted |
| Logging of unanswered questions | Reveals real gaps in the policy library |
| greytHR / HRMS write-back actions | Move from *answering about* leave to *applying for* it |
| Slack or Teams surface | Meets employees where they already work |
| Document version and freshness display | Prevents answers from superseded policy versions |
| Feedback capture on each answer | Enables continuous evaluation and prompt tuning |

---

## Summary

| Aspect | Implementation |
|---|---|
| Platform | Oracle AI Agent Studio (Fusion HCM) |
| Pattern | RAG over a curated document library |
| Documents | 16 company policy PDFs sourced from greytHR |
| Node types used | `START`, `LLM` ×3, `CONDITION`, `AGENT`, `END` |
| Agents | 1 reusable `WORKER` agent + 3 inline LLM nodes |
| Citations | Structurally required — document name + exact section |
| Interaction model | Multi-turn natural-language chat with context carry-over |
| Scope control | Pre-retrieval relevance gate with a friendly redirect |
| Failure mode | Honest "not found"; conflicts surfaced, never resolved silently |
