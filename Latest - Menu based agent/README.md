# Norvex Multi-Agent Policy Assistant

A conversational AI assistant built in **Oracle AI Agent Studio** that helps employees at
**Norvex Technologies** get answers to company policy questions.

Instead of one generic agent trying to know everything, it routes each conversation through a
**menu of five specialized topic areas**, each backed by its own AI agent and its own set of
policy documents.

---

## Problem Statement

> Companies deal with large volumes of internal documents — HR policies, product manuals,
> onboarding guides, SOPs — and employees waste time hunting for answers buried in PDFs.
> Build an AI assistant that lets users ask questions in plain English and get accurate,
> cited answers from a document library.

Company policies live as dozens of PDFs on an HRMS portal. An employee wondering *"How many days
do I have to submit a travel claim?"* has to guess which document applies, open a 40-page PDF,
`Ctrl+F` for the right clause, and interpret dense policy language alone — or raise an HR ticket.

A naive chatbot makes this worse: a confidently hallucinated policy answer is more damaging than
no answer, because employees act on it. The assistant has to be **verifiable**, not just fluent.

---

## Solution

| | |
|---|---|
| **Platform** | Oracle AI Agent Studio (Fusion HCM) |
| **Workflow** | 36 nodes, 7 node types |
| **Agents** | 5 published specialist agents |
| **Document tools** | 5 document tools, 18 policy PDFs |
| **Status** | Published |

Five specialist agents, each bound to exactly one document tool, so an agent can only ever answer
from its own domain's policies:

| # | Agent | Document tool | Covers |
|---|---|---|---|
| 1 | Lifecycle & Career | DT_Lifecycle_Career | Confirmation, transfer, relocation, PIP, appraisal & promotion |
| 2 | Conduct & Compliance | DT_Conduct_Compliance | POSH, whistleblower, disciplinary, BGV |
| 3 | IT & Security | DT_IT_Security | IT policies, HR security, remote connectivity & data protection |
| 4 | Compensation, Rewards & Travel | DT_Comp_Rewards | Rewards & recognition, certification reimbursement, travel |
| 5 | Onboarding & Culture | DT_Onboarding_Culture | Employee handbook, onboarding, referrals, communication |

Every agent is instructed to:

- **Ground every answer strictly in its attached documents** — never general knowledge or guesses
- **Say plainly when the answer is not there**, and point the employee to HR, the Ethics helpline,
  or the IT helpdesk as appropriate for that domain
- **Cite the exact file name it drew from**, plus the section, heading, or clause title whenever
  the document explicitly provides one — and never cite the document tool itself as a source
- **Never invent a section** that the source material does not contain
- **Suggest two to four follow-up questions**, phrased as complete natural questions, staying
  strictly inside its own topic

---

## The Workflow

Every time the employee sends a message, the entire workflow runs once from start to end, using
the full conversation history to understand context.

```
                    ┌───────────┐
                    │   START   │
                    └─────┬─────┘
                          ▼
          ┌───────────────────────────────┐
          │          Code Node            │  deterministic — no AI
          │  • which topic is ACTIVE?     │
          │  • is this a FRESH topic pick?│
          │  • menu-redisplay reset       │
          └───────────────┬───────────────┘
                          ▼
          ┌───────────────────────────────┐
          │         Check Stage           │  AI classifier
          │  treats the Code Node result  │  decides what KIND of
          │  as fixed, external truth     │  turn this is — and
          │  never answers the question   │  extracts the question
          └───────────────┬───────────────┘
                          ▼
       ╔══════════════════════════════════════════╗
       ║     13 CONDITION NODES, IN CASCADE       ║
       ║   each tests one outcome; else → next    ║
       ╚══════════════════════════════════════════╝
             │              │                │
             ▼              ▼                ▼
      ┌────────────┐  ┌───────────┐  ┌──────────────────┐
      │ LLM nodes  │  │  Return   │  │   Agent  (×5)    │
      │            │  │  static   │  │         ↓        │
      │ greeting   │  │  text     │  │  Formatter (×5)  │
      │ out-of-    │  │           │  │  • format answer │
      │  scope     │  │ menu +    │  │  • Sources block │
      │ topic      │  │ 5 topic   │  │  • numbered f/ups│
      │  mismatch  │  │ descrip-  │  │  • change-topic  │
      │            │  │ tions     │  │                  │
      └─────┬──────┘  └─────┬─────┘  └────────┬─────────┘
            └───────────────┴─────────────────┘
                            ▼
                      ┌───────────┐
                      │    END    │
                      └───────────┘
```

### Step 1 — Code Node *(deterministic, no AI)*

Before any AI reasoning, this node scans the raw conversation history to determine two things with
total reliability: **which topic is currently active**, and **whether this message is a fresh topic
pick from the menu**.

It finds the last topic-selection confirmation and the last menu display in the transcript, and
compares their positions. If the menu was shown *after* the last selection, the active topic
resets. That single comparison is the entire "change topic" reset mechanism — no stored state,
just relative position in the conversation.

A fresh selection is then recognised by exact matching against a fixed list of accepted inputs per
topic — the number, the topic name, and a couple of common variants such as *"it"* or
*"it & security"*. This only runs when no topic is already active, which is what stops a bare `3`
from being read as a topic switch when the user meant *"follow-up question 3"*.

**Why this is deterministic and not AI:** purely AI-based classification struggled to consistently
track which topic was active across a long, growing conversation. As history grows, a model's
attention to *"which menu item did the user pick eleven turns ago"* degrades — and the failure is
silent, because the wrong specialist still produces a fluent-sounding answer. Exact matching is
100% consistent on turn 2 and on turn 40 alike.

The design bet: **use deterministic logic for state, use AI for language.**

### Step 2 — Check Stage *(AI classifier)*

This node takes the Code Node's result **as fixed truth** and classifies the turn into exactly one
of fourteen outcomes. It never answers the question — it only decides what kind of turn this is
and extracts the question text to hand off.

Its central rule is explicit:

> Treat the active topic as fixed, external truth. Do not derive it yourself from conversation
> history, do not question it, do not override it, and do not change it under any circumstance,
> **even if it seems to contradict what you would have concluded on your own.**

Precedence order, stopping at the first match:

| # | Check | Notes |
|---|---|---|
| 1 | **Greeting** | *hi*, *hello*, *thanks* — always wins |
| 2 | **Change topic** | Judged by intent: *switch topic*, *go back*, *main menu*, *reset* |
| 3 | **Follow-up number** | Only when a topic is active and the message is exactly `1`–`5` |
| 4 | **Normal question** | Out of scope → no topic match · matches active topic → agent · differs → mismatch |

Rewards and Recognition plausibly belongs to either the Career or the Compensation topic, so the
classifier settles it explicitly: anything about compensation, salary, monetary rewards, or travel
and expense reimbursement goes to Compensation; anything about career progression, promotion
criteria, or performance improvement goes to Career.

### Step 3 — Routing

Thirteen condition nodes, **chained** — each one's negative outcome points at the next test.
Because the classifier guarantees exactly one outcome is true, the cascade behaves as a plain
if-else chain; all the precedence logic lives in the classifier, and the graph just walks the list.

### Step 4 — Agent Call

The matching specialist receives the resolved question, searches its own documents, and either
answers with citations or reports that the documents do not cover it and directs the employee to
HR, the Ethics helpline, or the IT helpdesk as appropriate for that domain.

**Live incident escalation.** The Conduct and IT agents detect when a message describes a real,
ongoing incident rather than a general policy question — harassment or retaliation for Conduct,
phishing or a lost device for IT. They skip the normal answer, provide official reporting steps
immediately, and **suppress follow-up questions entirely**, since an active incident response
should not invite casual browsing. Both also refuse to collect case details in the chat, redirecting
the employee to the official reporting channel instead.

### Step 5 — Formatter

One per agent branch. Takes the raw answer and prepares the final reply in five steps: format the
answer to suit the content (numbered list for procedures, table for comparisons, bullets for
independent facts, paragraph for a single fact), add the sources block, list the follow-up
questions numbered from 1, add the *"change topic"* closing line, and carry the follow-up list
forward for the next turn.

The formatter also bans throat-clearing — no opening sentence restating the company's general
commitment or policy purpose, straight to the first substantive fact.

### The follow-up mechanism

Each reply ends with a marker that records the follow-up questions exactly as they were displayed.
The marker is invisible when the message is read normally.

If the next message is a bare number, the Check Stage reads that marker from **only the single most
recent reply**, resolves the number to the full question text, and sends that to the agent — never
the bare digit, which would retrieve nothing. If the marker is missing or no number matches, it
falls back to showing the menu rather than guessing from an older list.

The classifier carries a pointed warning about this: several recent replies may contain similar
follow-up questions, and the correct list is determined **only by message position**, never by
which wording feels most familiar.

---

## Node Reference

**36 nodes across 7 types:** `START` ×1 · `CODE` ×1 · `LLM` ×9 · `CONDITION` ×13 · `RETURN` ×6 ·
`AGENT` ×5 · `END` ×1

| Node | Type | What it does |
|---|---|---|
| `START` | `START` | Pipeline entry point |
| `CODE_NODE` | `CODE` | Deterministic logic — works out the active topic and whether this is a fresh pick |
| `CHECK_STAGE` | `LLM` | Classifies the turn into one of 14 outcomes and extracts the question to answer |
| `IF_GREETING` | `CONDITION` | Is this a greeting? |
| `GREETING_HANDLER` | `LLM` | Friendly welcome plus the five-topic menu |
| `IF_MENU` | `CONDITION` | Is this a change-topic request? |
| `SHOW_MENU` | `RETURN` | Emits the static topic menu |
| `IF_OUT_OF_SCOPE` | `CONDITION` | Is this a non-policy question? |
| `OUT_OF_SCOPE` | `LLM` | Polite redirect back to the menu |
| `IF_CAREER_SELECTED` | `CONDITION` | Topic 1 picked? |
| `CAREER_DESCRIPTION` | `RETURN` | Confirms topic 1 and lists its scope |
| `IF_CONDUCT_SELECTED` | `CONDITION` | Topic 2 picked? |
| `CONDUCT_DESCRIPTION` | `RETURN` | Confirms topic 2 and lists its scope |
| `IF_IT_SELECTED` | `CONDITION` | Topic 3 picked? |
| `IT_DESCRIPTION` | `RETURN` | Confirms topic 3 and lists its scope |
| `IF_COMP_REWARDS_TRAVEL_SELECTED` | `CONDITION` | Topic 4 picked? |
| `COMPENSATION_REWARDS_AND_TRAVEL_DESCRIPTION` | `RETURN` | Confirms topic 4 and lists its scope |
| `IF_ONBOARDING_CULTURE_SELECTED` | `CONDITION` | Topic 5 picked? |
| `ONBOARDING_AND_CULTURE_DESCRIPTION` | `RETURN` | Confirms topic 5 and lists its scope |
| `IF_TOPIC_MISMATCH` | `CONDITION` | Does the question belong to a different topic? |
| `TOPIC_MISMATCH_HANDLER` | `LLM` | Reminds the user what the current topic covers, offers "change topic" |
| `IF_CAREER_QUESTION` | `CONDITION` | On-topic career question? |
| `CAREER_AGENT_CALL` | `AGENT` | Invokes `LIFECYCLE_CAREER_AGENT` |
| `CAREER_FORMATTER` | `LLM` | Formats the career answer for display |
| `IF_CONDUCT_QUESTION` | `CONDITION` | On-topic conduct question? |
| `CONDUCT_AGENT_CALL` | `AGENT` | Invokes `CONDUCT_COMPLIANCE_AGENT` |
| `CONDUCT_FORMATTER` | `LLM` | Formats the conduct answer for display |
| `IS_IT_QUESTION` | `CONDITION` | On-topic IT question? |
| `IT_AGENT_CALL` | `AGENT` | Invokes `IT_SECURITY_AGENT` |
| `IT_FORMATTER` | `LLM` | Formats the IT answer for display |
| `IS_COMP_REWARDS_TRAVEL_QUESTION` | `CONDITION` | On-topic compensation/travel question? |
| `COMP_REWARDS_TRAVEL_AGENT_CALL` | `AGENT` | Invokes `COMPENSATION_REWARDS_AND_TRAVEL_AGENT` |
| `COMP_REWARDS_TRAVEL_FORMATTER` | `LLM` | Formats the compensation/travel answer |
| `ONBOARDING_CULTURE_AGENT_CALL` | `AGENT` | Invokes `ONBOARDING_AND_CULTURE_AGENT` (fall-through branch) |
| `ONBOARDING_CULTURE_FORMATTER` | `LLM` | Formats the onboarding answer for display |
| `END` | `END` | Pipeline exit point — all branches converge here |

**Why `RETURN` nodes:** the menu and the five topic descriptions are fixed text that never varies,
so they are emitted verbatim rather than generated by a model. Faster, free, and the wording stays
identical every time — which matters, because `CODE_NODE` pattern-matches against exactly this
text.

---

## Document Tools

Five document tools, each attached to exactly one agent. Oracle AI Agent Studio handles
parsing, chunking, embedding, and semantic retrieval — no external vector database or custom code.

### DT_Lifecycle_Career
> Employee lifecycle and career policies — confirmation, transfers, relocation, performance
> improvement plans, appraisals, promotions, and career progression.

### DT_Conduct_Compliance
> Employee conduct, workplace compliance, ethics, disciplinary procedures, POSH-related matters,
> whistleblower protection, non-retaliation, and background verification.

### DT_IT_Security
> IT, information security, HR security, remote-access, connectivity, data protection, and
> technology usage policies.

### DT_Comp_Rewards
> Rewards, recognition programs, certification reimbursement, business travel, travel eligibility,
> expenses, and related reimbursement policies.

### DT_Onboarding_Culture
> Employee onboarding, general employee guidelines, workplace culture, referral programs, internal
> communication, and the employee handbook.

---

## Documents Uploaded

**18 policy PDFs**, downloaded from the HRMS portal and uploaded into the five document tools.

| Document Tool | Documents |
|---|---|
| DT_Lifecycle_Career | `Confirmation Policy.pdf`<br>`Transfer Policy.pdf`<br>`India Relocation Policy.pdf`<br>`PIP Process.pdf`<br>`Performance Appraisal and Promotion Policy.pdf` |
| DT_Conduct_Compliance | `POSH Policy.pdf`<br>`Whistleblower and Non Retaliation Policy.pdf`<br>`Disciplinary Policy.pdf`<br>`BGV Policy.pdf` |
| DT_IT_Security | `All IT Policies.pdf`<br>`HR Security Policy.pdf`<br>`Remote Connectivity and Data Protection Policy.pdf` |
| DT_Comp_Rewards | `Rewards and Recognition Policy.pdf`<br>`Certification Reimbursement Policy.pdf`<br>`Travel Policy.pdf` |
| DT_Onboarding_Culture | `Norvex India Employee Handbook.pdf`<br>`Norvex Crew  Employee Referral Policy.pdf`<br>`Communication Policy.pdf` |

The one-agent-to-one-tool pairing is what makes *"answer only from your own documents"* an
enforceable constraint rather than an aspiration — an agent physically cannot retrieve another
domain's policy.

---

## Screenshots

### Workflow design

**Full workflow canvas** — all 36 nodes as a single diagonal cascade, from the Code node through the condition chain to the five agent-and-formatter pairs.

![Full workflow canvas](docs/screenshots/01-workflow-canvas-full.png)

**Start of the pipeline** — `code node` → `Check Stage` → `If Greeting` → `If Menu` → `If Out Of Scope`, with the greeting and menu branches peeling off.

![Workflow start](docs/screenshots/02-workflow-start.png)

**Topic-selection branches** — the five `If … Selected` conditions, each with its `Return` description node, ending at `If Topic Mismatch`.

![Topic selection branches](docs/screenshots/03-workflow-topic-branches.png)

**Question routing** — the `Is … Question` conditions fanning out to the five `Agent Call` nodes, each feeding its own `Formatter`.

![Agent routing](docs/screenshots/04-workflow-agent-routing.png)

### Working demo

**Chat landing screen** — the assistant opens with its two starter questions.

![Chat landing](docs/screenshots/05-chat-landing.png)

**Greeting → topic menu** — *"Hey! I need help"* is classified as a greeting, and the five policy topics are presented.

![Greeting and menu](docs/screenshots/06-greeting-menu.png)

**Topic selected** — typing `2` locks the conversation to Conduct and Compliance and lists exactly what that topic covers.

![Topic selected](docs/screenshots/07-topic-selected.png)

**Cited answer with follow-ups** — *"What is POSH Policy?"* is answered from `POSH Policy.pdf`, cited down to the section, with four numbered follow-up questions.

![Cited answer](docs/screenshots/08-cited-answer.png)

**Follow-up by number** — replying `2` resolves to the retaliation-protections question and answers it, this time citing two separate sections.

![Follow-up selection](docs/screenshots/09-followup-selection.png)

**Out-of-scope handling** — *"Give me burger recipe"* is declined politely and the user is returned to the topic menu.

![Out of scope](docs/screenshots/10-out-of-scope.png)
