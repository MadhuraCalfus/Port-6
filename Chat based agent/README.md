# Company Policy Assistant — Chat-Based Agent

A conversational AI assistant built in **Oracle AI Agent Studio** that lets employees ask company
policy questions in plain English across a multi-turn chat, and returns answers grounded in — and
cited to — an internal library of policy PDFs.

---

## Problem Statement

> Companies deal with large volumes of internal documents — HR policies, product manuals,
> onboarding guides, SOPs — and employees waste time hunting for answers buried in PDFs.
> Build an AI assistant that lets users ask questions in plain English and get accurate,
> cited answers from a document library.

Company policies live as dozens of PDFs on an HRMS portal. An employee wondering *"How many days
do I have to submit a travel claim?"* has to guess which of 16 documents applies, open a 40-page
PDF, `Ctrl+F` for the right clause, and interpret dense policy language alone — or raise an HR
ticket.

A naive chatbot makes this worse: a confidently hallucinated policy answer is more damaging than
no answer, because employees act on it. The assistant has to be **verifiable**, not just fluent.

---

## Solution

| | |
|---|---|
| **Platform** | Oracle AI Agent Studio (Fusion HCM) |
| **Workflow** | Company Policy Chat Assistant — 7 nodes |
| **Agent** | 1 published policy agent |
| **Document tool** | 1 document tool, 16 policy PDFs |
| **Status** | Published |

A single reusable agent answers every policy question, with a relevance gate in front of it and a
formatting layer behind it.

| Property | Value |
|---|---|
| Agent | Company Policy Assistant |
| Role | Performs the task itself — no sub-agents |
| Reusable | Yes — can be invoked from any workflow |
| Tool attached | Company Policy Knowledge Base |

The agent returns three things for every question: the **answer**, the **source documents** that
support it, and **three suggested follow-up questions**. Because the source list is a required part
of that response, citations are structurally guaranteed rather than left to the model's discretion.

---

## The Workflow

Every time the employee sends a message, the entire workflow runs once from start to end, using
the full conversation history to understand context.

```
                    ┌───────────┐
                    │   START   │
                    └─────┬─────┘
                          │  user message + chat history
                          ▼
              ┌───────────────────────┐
              │      Check Input      │  LLM
              │  • what is the user   │  → the real question,
              │    really asking?     │    and whether it is
              │  • is it in scope?    │    in scope
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │       If Check        │  Condition
              │      in scope?        │
              └───┬───────────────┬───┘
           true   │               │   false
                  ▼               ▼
   ┌───────────────────────┐  ┌──────────────────────────┐
   │  Company Policy Agent │  │  Irrelevant Input        │  LLM
   │        Agent          │  │  Handling                │
   │                       │  │  friendly out-of-scope   │
   │  ┌─────────────────┐  │  │  redirect + topic list   │
   │  │ Company Policy  │  │  └────────────┬─────────────┘
   │  │ Knowledge Base  │  │               │
   │  └─────────────────┘  │               │
   │                       │               │
   │  → answer + sources   │               │
   │    + 3 suggestions    │               │
   └───────────┬───────────┘               │
               ▼                           │
   ┌───────────────────────┐               │
   │       LLM Agent       │  LLM          │
   │  presentation only:   │               │
   │  • branding           │               │
   │  • answer formatting  │               │
   │  • Option 1/2/3 menu  │               │
   │  • Sources block      │               │
   └───────────┬───────────┘               │
               └────────────┬──────────────┘
                            ▼
                      ┌───────────┐
                      │    END    │
                      └───────────┘
```

### Step 1 — Check Input *(router)*

This node reads the current message and the chat history, and returns two things: the question
that should actually be answered, and whether it falls inside the assistant's scope.

**Determining the question.** For a normal question, the message is taken verbatim. A hard rule
forbids chat history from overriding it — this prevents the common failure where a short question
like *"POSH policy?"* is silently rewritten into a topic from an earlier turn. History is consulted
in exactly one case: when the message is exactly *Option 1*, *Option 2*, *Option 3*, or a bare
`1`–`3`, the node looks back at the **most recent** reply containing *"You may also want to know:"*
and expands the selection into the full question text. Older option lists are explicitly out of
bounds.

**Determining relevance.** In scope means company and HR policy, leave, attendance, benefits,
travel, expenses, workplace procedures, and company rules. Everything else is out of scope. The
node never answers the question — it only classifies and resolves.

### Step 2 — If Check *(scope gate)*

A condition node routing in-scope questions to the agent and everything else to the redirect. Both
branches converge at the end.

Gating **before** retrieval rather than letting the agent decline afterwards costs no document
lookup, is more reliable than burying a scope judgement inside a long retrieval prompt, and
guarantees an off-topic message never reaches the knowledge base where it could provoke an
improvised answer.

### Step 3 — Agent Call

The agent receives the resolved question, queries the document tool, and returns the answer, its
sources, and three suggested follow-ups.

Its instructions run to 17 numbered sections. The load-bearing ones: the Knowledge Base is the only
authoritative source; answer only from what was retrieved; when nothing is found, say so plainly
and cite nothing; when two documents conflict, surface both and recommend confirming with HR rather
than silently picking one; never expose salaries or credentials; accuracy over completeness.

**Section extraction** is by far the longest part of the instructions, because a wrong document
name is obvious to a reader but a plausible-but-wrong section number is not. The rules: copy the
section exactly as it appears in the source passage, preserve numbering character-for-character,
and **omit it rather than guess** — never infer a section from the answer text, the document title,
or a numbered list in the answer.

### Step 4 — Formatter

A deliberately **non-reasoning** node. It receives the question, the answer, the sources, and the
suggested follow-ups, and does nothing but format them — explicitly forbidden from adding,
removing, or altering any fact, source, section, or question, and from mentioning any internal
machinery.

The reply it produces carries the company name and policy title at the top, the answer formatted to
suit its content (a paragraph, bullets, numbered steps, or a Markdown table), the three options
under *"You may also want to know:"*, and a sources block naming each document and its section.

A 20-point validation checklist runs before output — verifying the answer is unchanged, every
source is displayed, every section stays bound to its own document, numbering is preserved exactly,
and no section was invented or inferred.

### Step 5 — Out-of-Scope Handling

Handles the other branch. Instead of a blunt refusal, it briefly acknowledges the message, explains
the assistant's scope, lists what it *can* help with — leave, attendance, work from home, travel
and expenses, benefits, code of conduct, onboarding, workplace procedures — and invites a relevant
question. It replies in the user's language and uses short bullets rather than a wall of text.

This node runs **without** conversation history, unlike the other two — an off-topic message should
be answered on its own terms, with no risk of the model dragging a previous policy topic into a
redirect that isn't about policy.

### Conversational behaviour

The assistant is designed as a chat, not a search box:

- **Context carry-over** — *"What is the annual leave policy?"* → *"Who needs to approve it?"*
  resolves *it* against the previous turn
- **Clarification instead of guessing** — *"What is the policy?"* → *"Which policy would you like
  to know about — for example, leave, attendance, work from home, or travel?"*
- **Numbered follow-ups** — every grounded answer closes with three suggestions drawn only from
  what was retrieved, selectable by typing a number
- **Starter questions** — *"How do I file a POSH complaint?"* and *"What is the leave policy?"*

---

## Node Reference

**7 nodes across 5 types:** `START` ×1 · `LLM` ×3 · `CONDITION` ×1 · `AGENT` ×1 · `END` ×1

| Node | Type | What it does |
|---|---|---|
| `START` | `START` | Pipeline entry point |
| `CHECK_INPUT` | `LLM` | Resolves menu selections into full questions; classifies relevance |
| `IF_CHECK` | `CONDITION` | Branches on the scope decision — agent or redirect |
| `COMPANY_POLICY_AGENT` | `AGENT` | Invokes `COMPANY_POLICY_ASSISTANT`, which queries the document tool |
| `LLM_AGENT` | `LLM` | Formats the agent's result — branding, answer, options, sources |
| `IRRELEVANT_INPUT_HANDLING` | `LLM` | Friendly out-of-scope redirect with a topic list |
| `END` | `END` | Pipeline exit point — both branches converge here |

**Conversation history per LLM node:**

| Node | Sees prior turns | Why |
|---|---|---|
| `CHECK_INPUT` | Yes | Needs history to resolve option selections and pronouns |
| `LLM_AGENT` | Yes | Keeps the reply consistent with the conversation so far |
| `IRRELEVANT_INPUT_HANDLING` | No | An off-topic message is answered on its own terms |

All three reply in the user's language.

---

## Document Tool

| Property | Value |
|---|---|
| Tool | Company Policy Knowledge Base |
| Type | Document tool (retrieval-augmented generation) |
| Collection | Company Policies |
| Attachments | 16 PDFs, ~7.7 MB |
| User input required | No — the agent queries it autonomously |
| Status | Published |

> Provides access to company policy documents including leave, attendance, work from home, travel
> and expense, and employee code of conduct policies.

A document tool is Oracle AI Agent Studio's built-in RAG capability. The PDFs were uploaded
directly and the platform handles parsing, chunking, embedding, and semantic retrieval — no
external vector database, embedding pipeline, or custom code.

---

## Documents Uploaded

**16 policy PDFs**, downloaded from the **greytHR** HRMS portal and uploaded into the document
tool as a single collection.

| # | Document | Covers |
|---|---|---|
| 1 | `Calfus India Employee Handbook.pdf` | Overall employment terms, general policies |
| 2 | `All IT Policies.pdf` | Asset use, security, acceptable use, incidents |
| 3 | `HR Security Policy.pdf` | People-side information security obligations |
| 4 | `POSH Policy.pdf` | Prevention of Sexual Harassment — complaint process, ICC |
| 5 | `Whistleblower and Non_Retaliation Policy.pdf` | Reporting channels, protection from retaliation |
| 6 | `Disciplinary Policy.pdf` | Misconduct categories, disciplinary process |
| 7 | `PIP Process.pdf` | Performance Improvement Plan lifecycle |
| 8 | `Confirmation Policy.pdf` | Probation and confirmation criteria |
| 9 | `BGV Policy.pdf` | Background verification checks and process |
| 10 | `Travel Policy.pdf` | Business travel, entitlements, claim rules |
| 11 | `India Relocation Policy.pdf` | Relocation eligibility and reimbursement |
| 12 | `Transfer Policy.pdf` | Inter-location and inter-project transfers |
| 13 | `Certification Reimbursement Policy_Ver1.0.pdf` | Certification funding and claim conditions |
| 14 | `Calfus Crew _ Employee Referral Policy_1.2.pdf` | Referral eligibility, bonus, payout |
| 15 | `Rewards and Recognition Policy.pdf` | Award categories, nomination, cadence |
| 16 | `Communication Policy.pdf` | Internal and external communication standards |

---

## Screenshots

### Workflow design

**Full workflow canvas** — all seven nodes: `Check input` → `If check`, branching to the `Company Policy Agent` and its `LLM Agent` formatter, or to `irrelevant input handling`.

![Workflow canvas](docs/screenshots/01-workflow-canvas.png)

### Working demo

**Chat landing screen** — the assistant opens with its two starter questions.

![Chat landing](docs/screenshots/02-chat-landing.png)

**Cited answer with options** — *"How do I file a POSH complaint?"* is answered from the POSH Policy, closing with three numbered options and the source document.

![Cited answer](docs/screenshots/03-cited-answer.png)

**Option selection** — replying `Option 2` resolves to the complaint-timelines question and answers it, with a fresh set of three options.

![Option selection](docs/screenshots/04-option-selection.png)

**Format on request** — *"give me leave policy in table format"* returns a Markdown table of leave types, entitlements, encashment, and carry-forward rules.

![Table format answer](docs/screenshots/05-table-format-answer.png)

**Greeting handling** — *"Good morning"* is answered warmly and redirected to the policy areas the assistant can help with.

![Greeting guidance](docs/screenshots/06-greeting-guidance.png)
