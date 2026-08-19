# Norvex Multi-Agent Policy Assistant

A conversational AI assistant built in **Oracle AI Agent Studio** that helps employees at
**Norvex Technologies** get answers to company policy questions.

Instead of one generic agent trying to know everything, it routes each conversation through a
**menu of five specialized topic areas**, each backed by its own AI agent and its own set of
policy documents.

| | |
|---|---|
| **Workflow** | `NOVANIX_MULTI_AGENT_POLICY_ASSISTANT` (version 2, `PUBLISHED`) — rename to `NORVEX_…` pending |
| **Architecture** | `data_pipeline` — 36 nodes, 7 node types |
| **Agents** | 5 published `WORKER` agents |
| **Document tools** | 5 `DOCUMENT` tools, 18 policy PDFs total |
| **Model** | `ORA_MODEL_CONFIG_PREMIUM_OPEN_AI_GPT_4_1_MINI` |
| **Family / Product** | `HCM` / `GLOBAL_HUMAN_RESOURCES` |

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Why Multi-Agent Instead of One Agent](#why-multi-agent-instead-of-one-agent)
- [The Five Specialist Agents](#the-five-specialist-agents)
- [Document Tools](#document-tools)
- [The Conversation Flow](#the-conversation-flow)
- [Node Types Used](#node-types-used)
- [Complete Node Inventory](#complete-node-inventory)
- [Step 1 — Code Node](#step-1--code-node-deterministic-logic)
- [Step 2 — Check Stage](#step-2--check-stage-ai-classifier)
- [Step 3 — Routing](#step-3--routing-the-condition-cascade)
- [Step 4 — Agent Call](#step-4--agent-call)
- [Step 5 — Formatter](#step-5--formatter)
- [The Follow-Up Question Mechanism](#the-follow-up-question-mechanism)
- [Topic Locking and Mismatch Handling](#topic-locking-and-mismatch-handling)
- [Live Incident Escalation](#live-incident-escalation)
- [The Topic Menu](#the-topic-menu)
- [Sample Conversation](#sample-conversation)
- [Design Principles](#design-principles)
- [Setup & Deployment](#setup--deployment)
- [Screenshots](#screenshots)
- [Known Issues & Limitations](#known-issues--limitations)

---

## Problem Statement

### Mission

> Companies deal with large volumes of internal documents — HR policies, product manuals,
> onboarding guides, SOPs — and employees waste time hunting for answers buried in PDFs.
> Build an AI assistant that lets users ask questions in plain English and get accurate,
> cited answers from a document library.

### The pain in practice

Company policies live as dozens of PDFs on an HRMS portal. An employee wondering *"How many days
do I have to submit a travel claim?"* has to guess which document applies, open a 20–40 page PDF,
`Ctrl+F` for the right clause, and interpret dense policy language alone — or give up and raise
an HR ticket.

A naive chatbot makes this **worse**, not better: a confidently hallucinated policy answer is more
damaging than no answer, because employees act on it. Any assistant here has to be *verifiable*,
not merely fluent.

### Requirements this project had to meet

| # | Requirement | Where it is solved |
|---|---|---|
| 1 | Accept plain-English questions in a natural chat | Menu-routed chat workflow |
| 2 | Answer only from the document library | Five Document Tools, one bound to each agent |
| 3 | Cite every answer | `sources[]` with exact `fileName` and `section` |
| 4 | Never invent a policy | `foundInDocuments` flag plus explicit no-guess instructions |
| 5 | Stay inside scope | `isOutOfScope` classification with a menu redirect |
| 6 | Help users who don't know what to ask | Topic menu plus numbered follow-up suggestions |
| 7 | Track conversation state reliably | Deterministic `CODE` node instead of AI judgment |
| 8 | Handle live incidents safely | Escalation branches in the Conduct and IT agents |

---

## Why Multi-Agent Instead of One Agent

A single agent pointed at every policy document degrades in three specific ways:

**Retrieval noise.** A question about VPN access competes against travel, POSH, and referral
documents in the same vector space. Narrowing the corpus per agent raises retrieval precision.

**Prompt dilution.** One agent covering everything needs one enormous prompt covering every
domain's edge cases. Five focused agents each carry only what their own domain needs — including
domain-specific behaviour like incident escalation, which would be noise in a travel-reimbursement
prompt.

**No sense of scope.** A single agent cannot tell the user *"that's a different topic"*, because
every topic is its topic. Splitting by domain makes topic boundaries real, which is what makes the
menu, the topic lock, and mismatch detection possible.

The trade-off is routing — something has to decide which specialist handles each turn. That
routing is what most of this workflow's 36 nodes are.

---

## The Five Specialist Agents

Each agent is a separate, independently configured `WORKER` agent with exactly **one Document
Tool** attached, so it only ever answers from the policy documents relevant to its own area.

| # | Agent code | Name | Tool | Max interactions |
|---|---|---|---|---|
| 1 | `LIFECYCLE_CAREER_AGENT` | Lifecycle_Career_Agent | `DT_LIFECYCLE_CAREER` | 3 |
| 2 | `CONDUCT_COMPLIANCE_AGENT` | Conduct_Compliance_Agent | `DT_CONDUCT_COMPLIANCE` | 3 |
| 3 | `IT_SECURITY_AGENT` | IT_Security_Agent | `DT_IT_SECURITY` | 3 |
| 4 | `COMPENSATION_REWARDS_AND_TRAVEL_AGENT` | Compensation Rewards and Travel Agent | `DT_COMP_REWARDS` | 10 |
| 5 | `ONBOARDING_AND_CULTURE_AGENT` | Onboarding and Culture Agent | `DT_ONBOARDING_CULTURE` | *(not set)* |

All five are `type: WORKER`, `reusableFlag: true`, `status: PUBLISHED`, and take a single string
input named `question`.

### What each agent covers

| Agent | Scope |
|---|---|
| **Lifecycle_Career_Agent** | Confirmation, Transfer, India Relocation, PIP, Performance Appraisal & Promotion |
| **Conduct_Compliance_Agent** | POSH, Whistleblower & Non-Retaliation, Disciplinary process, Background Verification (BGV) |
| **IT_Security_Agent** | IT Policies, HR Security Policy, Remote Connectivity and Data Protection |
| **Compensation_Rewards_Travel_Agent** | Rewards and Recognition, Certification Reimbursement, Travel policies |
| **Onboarding_and_Culture_Agent** | Employee Handbook, onboarding, Employee Referral, Communication Policy, workplace culture |

### Shared agent instructions

Every specialist agent is instructed to:

- **Ground every answer strictly in the attached document tool** — never general knowledge,
  assumptions, or guesses about what a policy likely says
- **Say so plainly when the answer is not in the documents**, and direct the employee to HR, the
  Ethics helpline, or the IT helpdesk as appropriate for that domain
- **Cite the exact file name** it drew from (e.g. `POSH Policy.pdf`), plus the section, heading, or
  clause title whenever the retrieved content explicitly provides one — and **never cite the
  document tool's own name or code** as a source
- **Never invent a section** if the retrieved content does not include one
- **Generate two to four follow-up questions**, phrased as complete natural questions, staying
  strictly inside its own topic and explicitly not suggesting another agent's territory

### Shared output schema

All five agents return structured output:

```jsonc
{
  "answer": "string",             // the response text to show the user
  "foundInDocuments": true,       // false when the documents don't cover it
  "isActiveIncident": false,      // live incident vs. general policy question
  "sources": [                    // empty when foundInDocuments is false
    {
      "fileName": "POSH Policy.pdf",          // required
      "section": "Complaint Procedure"        // only when explicitly present
    }
  ],
  "followUpQuestions": ["...", "...", "..."]  // 0–4 items; empty on a live incident
}
```

Two schema variations exist: `IT_SECURITY_AGENT` names its flag `isActiveSecurityIncident` rather
than `isActiveIncident`, and `LIFECYCLE_CAREER_AGENT` has no incident flag at all (its
`followUpQuestions` uses `minItems: 2` where the others use `0`, since it never needs to return an
empty list).

---

## Document Tools

Five `DOCUMENT`-type tools, all `PUBLISHED`. Each holds its domain's PDFs as separately named
collections.

### `DT_LIFECYCLE_CAREER`
> Handles employee lifecycle and career-related policies, including confirmation, transfers,
> relocation, performance improvement plans, appraisals, promotions, and career progression.

`Confirmation Policy.pdf` · `Transfer Policy.pdf` · `India Relocation Policy.pdf` ·
`PIP Process.pdf` · `Performance Appraisal and Promotion Policy.pdf`

### `DT_CONDUCT_COMPLIANCE`
> Handles employee conduct, workplace compliance, ethics, disciplinary procedures, POSH-related
> matters, whistleblower protection, non-retaliation, and background verification policies.

`POSH Policy.pdf` · `Whistleblower and Non Retaliation Policy.pdf` · `Disciplinary Policy.pdf` ·
`BGV Policy.pdf`

### `DT_IT_SECURITY`
> Handles IT, information security, HR security, remote-access, connectivity, data protection, and
> technology usage policies applicable to employees.

`All IT Policies.pdf` · `HR Security Policy.pdf` ·
`Remote Connectivity and Data Protection Policy.pdf`

### `DT_COMP_REWARDS`
> Handles employee rewards, recognition programs, certification reimbursement, business travel,
> travel eligibility, expenses, and related reimbursement policies.

`Rewards and Recognition Policy.pdf` · `Certification Reimbursement Policy.pdf` ·
`Travel Policy.pdf`

### `DT_ONBOARDING_CULTURE`
> Handles employee onboarding, general employee guidelines, workplace culture, referral programs,
> internal communication, and information contained in the employee handbook.

`Norvex India Employee Handbook.pdf` · `Norvex Crew  Employee Referral Policy.pdf` ·
`Communication Policy.pdf`

The one-agent-to-one-tool pairing is what makes *"answer only from your own documents"* an
enforceable constraint rather than an aspiration — an agent physically cannot retrieve another
domain's policy.

---

## The Conversation Flow

Every time the employee sends a message, **the entire workflow runs once from start to end**,
using the full conversation history to understand context — since there is no persistent memory
between turns other than the visible chat.

```
                       ┌───────────┐
                       │   START   │
                       └─────┬─────┘
                             │  user message + full chat history
                             ▼
             ┌───────────────────────────────┐
             │          CODE_NODE            │  type: CODE (JavaScript)
             │   deterministic state scan    │  NOT an AI step
             │                               │
             │  • which topic is ACTIVE?     │  → { activeTopic,
             │  • is this a FRESH topic pick?│      isFreshSelection,
             │  • menu-redisplay reset       │      selectedTopic }
             └───────────────┬───────────────┘
                             ▼
             ┌───────────────────────────────┐
             │         CHECK_STAGE           │  type: LLM (classifier)
             │  treats CODE_NODE output as   │
             │  FIXED, EXTERNAL TRUTH        │  → 14 boolean flags
             │                               │    + activeTopic
             │  never answers the question   │    + mismatchTopic
             │  exactly ONE flag may be true │    + resolvedQuestion
             └───────────────┬───────────────┘
                             ▼
        ╔════════════════════════════════════════════╗
        ║      13 CONDITION NODES, IN CASCADE        ║
        ║   each tests one flag; false → next test   ║
        ╚════════════════════════════════════════════╝
                             │
   ┌─────────────────────────┼──────────────────────────────┐
   │                         │                              │
   ▼                         ▼                              ▼
IF_GREETING            IF_*_SELECTED  (×5)          IF_*_QUESTION  (×4)
IF_MENU                                                + fall-through
IF_OUT_OF_SCOPE                                              │
IF_TOPIC_MISMATCH                                            │
   │                         │                              │
   ▼                         ▼                              ▼
┌──────────────┐   ┌────────────────────┐    ┌──────────────────────────┐
│  LLM nodes   │   │   RETURN nodes     │    │     AGENT nodes  (×5)    │
│              │   │   static text —    │    │  ┌────────────────────┐  │
│ GREETING_    │   │   no model call    │    │  │ CAREER_AGENT_CALL  │  │
│  HANDLER     │   │                    │    │  │ CONDUCT_AGENT_CALL │  │
│ OUT_OF_SCOPE │   │ SHOW_MENU          │    │  │ IT_AGENT_CALL      │  │
│ TOPIC_       │   │ CAREER_DESCRIPTION │    │  │ COMP_REWARDS_...   │  │
│  MISMATCH_   │   │ CONDUCT_DESCRIPTION│    │  │ ONBOARDING_...     │  │
│  HANDLER     │   │ IT_DESCRIPTION     │    │  └─────────┬──────────┘  │
│              │   │ COMP_..._DESC      │    │            │             │
│              │   │ ONBOARDING_..._DESC│    │            ▼             │
│              │   │                    │    │  ┌────────────────────┐  │
│              │   │                    │    │  │  FORMATTER  (×5)   │  │
│              │   │                    │    │  │  type: LLM         │  │
│              │   │                    │    │  │  • format answer   │  │
│              │   │                    │    │  │  • Sources block   │  │
│              │   │                    │    │  │  • numbered f/ups  │  │
│              │   │                    │    │  │  • change-topic    │  │
│              │   │                    │    │  │  • hidden marker   │  │
│              │   │                    │    │  └─────────┬──────────┘  │
└──────┬───────┘   └─────────┬──────────┘    └────────────┼─────────────┘
       │                     │                            │
       └─────────────────────┴──────────────┬─────────────┘
                                            ▼
                                      ┌───────────┐
                                      │    END    │
                                      └───────────┘
```

### The actual condition cascade

The 13 conditions are **chained**, not parallel — each one's `false` outcome points at the next
test:

```
CHECK_STAGE
  → IF_GREETING                       true → GREETING_HANDLER          → END
  → IF_MENU                           true → SHOW_MENU                 (RETURN)
  → IF_OUT_OF_SCOPE                   true → OUT_OF_SCOPE              → END
  → IF_CAREER_SELECTED                true → CAREER_DESCRIPTION        (RETURN)
  → IF_CONDUCT_SELECTED               true → CONDUCT_DESCRIPTION       (RETURN)
  → IF_IT_SELECTED                    true → IT_DESCRIPTION            (RETURN)
  → IF_COMP_REWARDS_TRAVEL_SELECTED   true → COMPENSATION_..._DESCRIPTION (RETURN)
  → IF_ONBOARDING_CULTURE_SELECTED    true → ONBOARDING_..._DESCRIPTION   (RETURN)
  → IF_TOPIC_MISMATCH                 true → TOPIC_MISMATCH_HANDLER    → END
  → IF_CAREER_QUESTION                true → CAREER_AGENT_CALL         → CAREER_FORMATTER      → END
  → IF_CONDUCT_QUESTION               true → CONDUCT_AGENT_CALL        → CONDUCT_FORMATTER     → END
  → IS_IT_QUESTION                    true → IT_AGENT_CALL             → IT_FORMATTER          → END
  → IS_COMP_REWARDS_TRAVEL_QUESTION   true → COMP_REWARDS_TRAVEL_AGENT_CALL → COMP_..._FORMATTER → END
                                     false → ONBOARDING_CULTURE_AGENT_CALL → ONBOARDING_..._FORMATTER → END
```

Note the tail: there is **no `IF_ONBOARDING_QUESTION` node.** Onboarding is the fall-through
default — the final `false` branch. This works because `CHECK_STAGE` guarantees exactly one flag is
true, so anything reaching that point must be an onboarding question. It is one node cheaper, at
the cost of making the Onboarding agent the catch-all if the classifier ever returns all-false.

Every condition node declares `convergenceTargetId: end`. The graph has **no dangling outcome
targets and no unreachable nodes.**

---

## Node Types Used

| Type | Count | What it does |
|---|---|---|
| `START` | 1 | Pipeline entry point, declared as `rootNode` |
| **`CODE`** | 1 | Runs **JavaScript** — deterministic, no language model involved |
| `LLM` | 9 | Direct model calls: 1 classifier, 3 handlers, 5 formatters |
| `CONDITION` | 13 | Branch on a single boolean from `CHECK_STAGE` |
| **`RETURN`** | 6 | Emits **static text** and terminates the turn — no model call |
| `AGENT` | 5 | Invokes a published specialist agent by `agentCode` |
| `END` | 1 | Exit point |

Two of these deserve attention.

**The `CODE` node is the architecturally distinctive choice.** Most Agent Studio workflows do
everything with LLM nodes. Putting deterministic JavaScript *before* any AI reasoning — and having
the classifier treat its output as fixed truth — is what makes topic tracking reliable across a
long conversation.

**The `RETURN` nodes are the cheap-and-exact choice.** The topic menu and the five topic
descriptions are fixed text that never varies, so they are emitted verbatim by `RETURN` nodes
rather than generated by a model. That is faster, free, and guarantees the wording is identical
every time — which matters, because the `CODE` node pattern-matches against exactly this wording.

---

## Complete Node Inventory

| Node code | Type | Role |
|---|---|---|
| `START` | `START` | Entry point |
| `CODE_NODE` | `CODE` | Deterministic topic-state extraction |
| `CHECK_STAGE` | `LLM` | Turn classifier — 14 flags, exactly one true |
| `IF_GREETING` | `CONDITION` | Route greetings |
| `GREETING_HANDLER` | `LLM` | Welcome + five-topic menu |
| `IF_MENU` | `CONDITION` | Route change-topic requests |
| `SHOW_MENU` | `RETURN` | The static topic menu |
| `IF_OUT_OF_SCOPE` | `CONDITION` | Route non-policy questions |
| `OUT_OF_SCOPE` | `LLM` | Polite redirect + menu |
| `IF_CAREER_SELECTED` | `CONDITION` | Topic 1 picked |
| `CAREER_DESCRIPTION` | `RETURN` | Topic 1 confirmation + scope list |
| `IF_CONDUCT_SELECTED` | `CONDITION` | Topic 2 picked |
| `CONDUCT_DESCRIPTION` | `RETURN` | Topic 2 confirmation + scope list |
| `IF_IT_SELECTED` | `CONDITION` | Topic 3 picked |
| `IT_DESCRIPTION` | `RETURN` | Topic 3 confirmation + scope list |
| `IF_COMP_REWARDS_TRAVEL_SELECTED` | `CONDITION` | Topic 4 picked |
| `COMPENSATION_REWARDS_AND_TRAVEL_DESCRIPTION` | `RETURN` | Topic 4 confirmation + scope list |
| `IF_ONBOARDING_CULTURE_SELECTED` | `CONDITION` | Topic 5 picked |
| `ONBOARDING_AND_CULTURE_DESCRIPTION` | `RETURN` | Topic 5 confirmation + scope list |
| `IF_TOPIC_MISMATCH` | `CONDITION` | Question belongs to a different topic |
| `TOPIC_MISMATCH_HANDLER` | `LLM` | Scope reminder + change-topic hint |
| `IF_CAREER_QUESTION` | `CONDITION` | On-topic career question |
| `CAREER_AGENT_CALL` | `AGENT` | → `LIFECYCLE_CAREER_AGENT` |
| `CAREER_FORMATTER` | `LLM` | Format career answer |
| `IF_CONDUCT_QUESTION` | `CONDITION` | On-topic conduct question |
| `CONDUCT_AGENT_CALL` | `AGENT` | → `CONDUCT_COMPLIANCE_AGENT` |
| `CONDUCT_FORMATTER` | `LLM` | Format conduct answer |
| `IS_IT_QUESTION` | `CONDITION` | On-topic IT question |
| `IT_AGENT_CALL` | `AGENT` | → `IT_SECURITY_AGENT` |
| `IT_FORMATTER` | `LLM` | Format IT answer |
| `IS_COMP_REWARDS_TRAVEL_QUESTION` | `CONDITION` | On-topic comp/travel question |
| `COMP_REWARDS_TRAVEL_AGENT_CALL` | `AGENT` | → `COMPENSATION_REWARDS_AND_TRAVEL_AGENT` |
| `COMP_REWARDS_TRAVEL_FORMATTER` | `LLM` | Format comp/travel answer |
| `ONBOARDING_CULTURE_AGENT_CALL` | `AGENT` | → `ONBOARDING_AND_CULTURE_AGENT` (fall-through) |
| `ONBOARDING_CULTURE_FORMATTER` | `LLM` | Format onboarding answer |
| `END` | `END` | Exit point |

---

## Step 1 — Code Node *(deterministic logic)*

**Type:** `CODE` · **Return type:** `object` · **AI involved:** none

Before any AI reasoning happens, a JavaScript node scans the raw conversation history to figure
out two things with total reliability:

1. **Which topic (if any) is currently active**
2. **Whether the user's current message is a fresh topic pick from the menu**

```js
return {
  activeTopic:      activeTopic,        // CAREER | CONDUCT | IT | COMP_TRAVEL | ONBOARDING | NONE
  isFreshSelection: !!selectedTopic,
  selectedTopic:    selectedTopic
};
```

### How it works

**Step A — find the active topic, with menu-redisplay reset.** Two regexes scan the history:

```js
/You selected (Career and Lifecycle|Conduct and Compliance|IT and Security|Compensation[^\n]*|Onboarding and Culture)/g
/Please choose a topic to get started|Please choose one of the following policy topics/g
```

It records the index of the **last** topic selection and the index of the **last** menu display,
then applies the rule:

```js
if (lastSelectedIndex > lastMenuIndex) { /* a topic is active */ }
```

If the menu was shown *after* the last selection, the topic resets to `NONE`. That single
comparison is the entire "change topic" reset mechanism — no state store, no flags, just relative
position in the transcript.

**Step B — detect a fresh selection.** Only when no topic is already active, the current input is
normalised (`toLowerCase`, strip everything but `a-z0-9 &`) and matched against exact allow-lists:

| Topic | Accepted inputs |
|---|---|
| `CAREER` | `1`, `career`, `career and lifecycle`, `career & lifecycle` |
| `CONDUCT` | `2`, `conduct`, `conduct and compliance`, `conduct & compliance` |
| `IT` | `3`, `it`, `it and security`, `it & security` |
| `COMP_TRAVEL` | `4`, `compensation`, `travel`, `compensation rewards and travel`, … |
| `ONBOARDING` | `5`, `onboarding`, `culture`, `onboarding and culture`, … |

Guarding this behind `activeTopic === "NONE"` is what stops a bare `3` from being read as a topic
switch when the user actually meant "follow-up question 3".

**Step C — a fresh pick takes effect immediately**, otherwise the history value carries forward.

### Why this is code and not AI

This step exists because **purely AI-based classification struggled to consistently track "which
topic is active" across a long, growing conversation.** As history grows, a language model's
attention to *"which menu item did the user pick eleven turns ago"* degrades — and the failure is
silent, because the wrong specialist still produces a fluent-sounding answer.

A Code node does this with **exact text matching** instead of language-model judgment, which is far
more reliable for this specific, narrow task. The cost is brittleness — it only matches the strings
it was written to match. The benefit is that within that scope it is 100% consistent, on turn 2 and
on turn 40 alike.

This is the workflow's central design bet: **use code for state, use AI for language.**

---

## Step 2 — Check Stage *(AI classifier)*

**Type:** `LLM` · **`chatHistoryEnabled`:** `true` · **Returns:** structured JSON

An LLM node takes the Code node's output **as fixed truth** and classifies the current turn. It
**never answers the question itself** — its only job is figuring out what kind of turn this is and,
when relevant, extracting the actual question text to hand off.

### Output: 17 fields

Fourteen boolean flags, plus three strings:

```
isGreeting  isMenu  isOutOfScope
isCareerSelected  isConductSelected  isITSelected
isCompTravelSelected  isOnboardingSelected
isTopicMismatch
isCareerQuestion  isConductQuestion  isITQuestion
isCompTravelQuestion  isOnboardingQuestion

activeTopic       enum: CAREER | CONDUCT | IT | COMP_TRAVEL | ONBOARDING | NONE
mismatchTopic     enum: (same) | ""
resolvedQuestion  string
```

All 17 are `required`, and `additionalProperties: false`.

### The three global rules

**Exactly one outcome.** Precisely one of the fourteen flags must be true and all the rest false.
`resolvedQuestion` stays empty unless one of the five question flags is the true one;
`mismatchTopic` stays empty unless `isTopicMismatch` is the true one. This is what lets the
downstream cascade be a simple chain of single-flag tests.

**Rule Zero — `activeTopic` comes from the Code node, never recompute it.** The prompt is explicit:

> Treat this activeTopic value as fixed, external truth. Do not derive it yourself from
> conversation history, do not question it, do not override it, and do not change it under any
> circumstance, **even if it seems to contradict what you would have concluded on your own by
> reading the history.**

That last clause is the whole point. Without it, the model helpfully "corrects" the Code node and
the determinism is lost.

**Fresh selection short-circuits everything.** If `isFreshSelection` is true, the classifier sets
the matching `isXSelected` flag and stops — Steps 1 through 4 never run.

### Precedence order

Checked in this exact order, stopping at the first match:

| # | Step | Notes |
|---|---|---|
| 1 | **Greeting** | `hi`, `hello`, `good morning`, `thanks`, `ok` |
| 2 | **Change topic request** | Judged by *intent*, not exact phrase — `change topic`, `switch topic`, `go back`, `main menu`, `reset`, `start over` |
| 3 | **Follow-up numeric selection** | Only when `activeTopic ≠ NONE` and input is exactly `1`–`5` |
| 4 | **Normal question classification** | Everything else |

A greeting or change-topic request **always wins**, even mid-topic and even right after a follow-up
list was shown.

### Step 4's decision table

| Condition | Result |
|---|---|
| Question matches none of the five topics | `isOutOfScope` |
| `activeTopic` is `NONE` | `isMenu` |
| Detected topic **matches** `activeTopic` | matching `isXQuestion` + `resolvedQuestion` = current input |
| Detected topic **differs** from `activeTopic` | `isTopicMismatch` + `mismatchTopic` = detected topic |

### The disambiguation note

Rewards and Recognition plausibly belongs to either CAREER or COMP_TRAVEL, so the prompt settles it
explicitly:

> Treat any question specifically about compensation, salary, monetary rewards, or travel and
> expense reimbursement as **COMP_TRAVEL**. Treat questions about career progression, promotion
> criteria, or performance improvement plans as **CAREER**.

The prompt closes with **eight worked examples** covering fresh selection, follow-up extraction,
unmatched number fallback, change-topic intent, greeting, on-topic question, topic mismatch, and
out-of-scope.

---

## Step 3 — Routing *(the condition cascade)*

**Type:** `CONDITION` × 13

Each condition tests exactly one flag:

```
{{$context.$nodes.CHECK_STAGE.$output.isGreeting}}
{{$context.$nodes.CHECK_STAGE.$output.isMenu}}
{{$context.$nodes.CHECK_STAGE.$output.isOutOfScope}}
...
```

Because `CHECK_STAGE` guarantees exactly one flag is true, the cascade is a plain if-else chain —
no compound expressions, no precedence logic in the graph itself. All the ordering intelligence
lives in the classifier prompt; the graph just walks the list.

| Classification | Response path |
|---|---|
| **Greeting** | `GREETING_HANDLER` — friendly welcome plus the five-topic menu |
| **Menu request** | `SHOW_MENU` — the static topic menu |
| **Out of scope** | `OUT_OF_SCOPE` — polite redirect back to the menu |
| **Topic selected** | The matching `*_DESCRIPTION` — confirms the topic and lists what it covers |
| **Topic mismatch** | `TOPIC_MISMATCH_HANDLER` — reminds the user what the current topic covers, and that they can say *"change topic"* |
| **On-topic question** | The matching specialist agent |

Routing **before** retrieval — rather than letting an agent field everything and decline afterwards
— costs no document lookup, and guarantees an off-topic message never reaches a knowledge base
where it could provoke an improvised answer.

---

## Step 4 — Agent Call

**Type:** `AGENT` × 5

Each agent node passes the classifier's resolved question straight through:

```
message  = {{$context.$nodes.CHECK_STAGE.$output.resolvedQuestion}}
question = {{$context.$nodes.CHECK_STAGE.$output.resolvedQuestion}}
```

The matching specialist searches **its own attached documents**, and either:

- **Answers** with `foundInDocuments: true`, populated `sources`, and follow-up suggestions, or
- **Says plainly it couldn't find the answer** — `foundInDocuments: false`, `sources: []` — and
  directs the employee to HR, the Ethics helpline, or the IT helpdesk as appropriate

That second path matters as much as the first. An honest *"not covered, here's who to ask"* is a
useful answer; a fabricated policy is not. The escalation target is chosen per domain, so an
unanswered IT question points at IT rather than generically at HR.

---

## Step 5 — Formatter

**Type:** `LLM` × 5 — one per agent branch

Each formatter takes its agent's raw output and prepares the final reply in five ordered steps.

**1 — Format the main answer.** Choose the presentation format from the *nature of the content*:
numbered list for sequential steps or processes, table for comparisons across shared attributes,
bullets for independent items, short paragraph for a single fact or simple explanation. Headings
when there are multiple distinct sections. Never force a format that doesn't improve readability,
and always honour an explicitly requested format.

The system prompt also bans throat-clearing:

> Do not begin the response with a sentence that restates the company's general commitment, policy
> purpose, or introductory framing… Skip straight to the first substantive fact, rule, or
> definition.

**2 — Display sources.**

```
Sources:

- File Name.pdf — Section: Section Name
- Another File.pdf
```

Every source listed, never invented or modified, section included only when provided, and the whole
block omitted when `sources` is empty.

**3 — Display follow-up questions.** Used exactly as provided — no rewording, no reordering, no
inventions — numbered from 1, introduced with *"You might also want to ask about:"* and followed by
*"You can reply with just the number to ask that question."* Omitted entirely when the array is
empty.

**4 — Closing message.** *"You can say "change topic" at any time to see the full topic menu
again."*

**5 — Hidden follow-up data marker.** See below.

The formatter is a presentation layer, not a reasoning layer: it decides how the answer looks, not
what it says. It is explicitly barred from mentioning workflows, nodes, tools, agents, variables,
context, flags, or system processes.

---

## The Follow-Up Question Mechanism

To let employees reply with just a number to ask a suggested follow-up question, the formatter
**embeds the follow-up list as a hidden JSON marker at the very end of its visible reply** —
invisible in normal reading, but present in the raw text.

```html
<!--FOLLOWUPJSON[{"n":1,"question":"What is the disciplinary process?"},{"n":2,"question":"What are my responsibilities?"}]FOLLOWUPJSON-->
```

The formatter must keep the marker's contents **identical in text and order** to the visible list,
and emits `<!--FOLLOWUPJSON[]FOLLOWUPJSON-->` when there are no follow-ups.

On the next turn, if the input is a bare `1`–`5` and a topic is active, `CHECK_STAGE`:

1. Identifies the **single last assistant turn** — the message immediately before the current input
2. Searches **only that turn** for a substring between `<!--FOLLOWUPJSON` and `FOLLOWUPJSON-->`
3. Parses the JSON array and finds the object whose `n` matches the input
4. Copies its `question` verbatim into `resolvedQuestion`
5. Sets the question flag matching `activeTopic` — since every follow-up belongs to the active topic

```
Turn 1
  User: "What is the reimbursement process for travel expenses?"
    → COMP_REWARDS_TRAVEL_AGENT answers, cites Travel Policy.pdf
    → formatter renders 1. … 2. … 3. …
       and appends the hidden marker carrying those exact questions

Turn 2
  User: "2"
    → CHECK_STAGE reads the marker from the LAST assistant message only
    → resolvedQuestion = "What is the deadline for submitting travel claims?"
    → the AGENT receives the FULL QUESTION, never the bare digit
```

### The anti-drift warning

The classifier prompt carries an unusually direct instruction about this specific failure:

> **CRITICAL WARNING ABOUT THIS SPECIFIC FAILURE MODE:** multiple recent assistant turns may
> contain very similar or even overlapping follow up questions… You may feel a pull toward
> whichever list's wording seems most familiar or most recently repeated in your own reasoning.
> **Resist this.** The correct list is determined ONLY by strict message position… If you find
> yourself drawn to a list because its wording feels closer to a recent theme, that is the wrong
> reason and you must instead re-check literal message position.

**Graceful fallback:** if the marker is missing from that last turn, or no `n` matches, the
classifier sets `isMenu` instead — it does **not** search earlier turns and does **not** fall
through to Step 4. A stale suggestion list can never be selected by accident.

---

## Topic Locking and Mismatch Handling

The conversation stays anchored to whichever topic was last selected. When a question drifts, the
system **detects it and gently redirects rather than answering under the wrong specialist.**

```
Active topic: CAREER

User: "What is the leave policy?"
  → CODE_NODE:    activeTopic = CAREER, isFreshSelection = false
  → CHECK_STAGE:  isTopicMismatch = true, mismatchTopic = ONBOARDING
  → TOPIC_MISMATCH_HANDLER: reminder of what Career and Lifecycle covers,
                            plus "change topic" hint
```

Answering that question through `LIFECYCLE_CAREER_AGENT` would fail one of two ways: either
`DT_LIFECYCLE_CAREER` contains nothing about leave (an unhelpful dead end), or a passing mention
gets retrieved and misrepresented as policy. The mismatch branch avoids both by never letting the
wrong specialist attempt the question.

`TOPIC_MISMATCH_HANDLER` receives both the active topic and the detected topic, and reproduces the
active topic's bullet list **word for word** from its system prompt — the same lists the
`*_DESCRIPTION` RETURN nodes use, so the user sees consistent scope wording wherever it appears.

The user is never trapped — `change topic` returns to the menu at any point, and the classifier
treats it at precedence level 2, above everything except greetings.

---

## Live Incident Escalation

Two agents carry full incident-handling instructions, because in their domains a policy question
and a crisis look superficially similar.

### `CONDUCT_COMPLIANCE_AGENT`

> If the employee appears to be describing a real, ongoing, or recent incident, such as harassment,
> retaliation, discrimination, or misconduct, rather than asking a general policy question, do not
> evaluate, judge, or attempt to resolve it. Immediately provide the official reporting steps from
> policy and clearly encourage the employee to report it. **Never discourage, minimize, or delay a
> report.** Always mention that policy provides confidentiality and protection from retaliation.

It further carries **no case details in chat** (redirect to the official channel rather than
drawing out incident detail), **no judgment calls** (never opine on whether something counts as
harassment, likely outcomes, or appropriate disciplinary action), and a required **calm,
respectful, non-judgmental tone.**

### `IT_SECURITY_AGENT`

> If the employee appears to be describing a real or ongoing security incident, such as a suspected
> phishing email, a lost or stolen device, malware, or unauthorized account access… do not
> investigate, diagnose, or try to resolve it yourself. Immediately provide the official reporting
> steps and clearly direct the employee to report it to the IT helpdesk or security team right
> away. **Never tell the employee to wait or handle it themselves first.**

Plus its own no-details rule: *"for security, please do not share that information here"* if the
user starts pasting passwords, account numbers, or confidential data.

### Suppressing follow-ups

When the incident flag is true, **`followUpQuestions` must be empty** —

> since an active incident response should not invite casual follow up browsing.

This is why the schema uses `minItems: 0`. Closing a harassment disclosure with *"You might also
want to ask about: 1. Who are the ICC members?"* would be tone-deaf and actively harmful.

| Question | Handling |
|---|---|
| *"What is the POSH complaint process?"* | Normal cited answer with follow-up suggestions |
| *"I am being harassed by my manager"* | Reporting steps, confidentiality note, **no** follow-ups |
| *"What is the password policy?"* | Normal cited answer with follow-up suggestions |
| *"I think my laptop has been compromised"* | Immediate IT helpdesk escalation, **no** follow-ups |

---

## The Topic Menu

Emitted verbatim by the `SHOW_MENU` RETURN node:

```
Please choose a topic to get started:

1 - Career and Lifecycle
Covers confirmation, transfers, relocation, performance management, promotions, PIP,
certification reimbursement, and other employee lifecycle policies.

2 - Conduct and Compliance
Covers Code of Conduct, Disciplinary Process, POSH, Whistleblower, Non-Retaliation,
and Background Verification (BGV) policies.

3 - IT and Security
Covers IT security, password and data protection, asset management, VPN, device usage,
remote access, and information security policies.

4 - Compensation, Rewards & Travel
Compensation, rewards and recognition, business travel, travel expenses, reimbursements,
and allowances.

5 - Onboarding and Culture
Onboarding, induction, leave and holiday policies, workplace culture, employee engagement,
and new-joiner policies.

Reply with the option name or number.
```

Selecting a topic returns that topic's `*_DESCRIPTION` node — for example:

```
You selected IT and Security.

This topic covers:

* IT Policies — General IT usage, technology resources, and employee responsibilities
* Password and Data Security — Password management, data protection, and information security
* Asset Management — Company devices, IT assets, and their appropriate usage
* VPN and Device Guidelines — VPN access, remote connectivity, and company device usage

Please type your question to continue.

You can type "change topic" at any time to choose a different policy topic.
```

The `You selected …` opening line is not cosmetic — it is exactly the string the `CODE_NODE` regex
scans for on every subsequent turn.

### Starter questions

The chat experience opens with two seeded prompts:

- *"Hey! I need help"*
- *"What topics can you help me with?"*

---

## Sample Conversation

**Employee:** `Hey! I need help`

→ `CODE_NODE`: `activeTopic = NONE` · `CHECK_STAGE`: `isGreeting` · `GREETING_HANDLER` renders the
welcome plus the five-topic menu.

**Employee:** `3`

→ `CODE_NODE`: no active topic, so Step B runs — `"3"` matches the IT allow-list →
`selectedTopic = IT`, `isFreshSelection = true` · `CHECK_STAGE` short-circuits to `isITSelected` ·
`IT_DESCRIPTION` returns the scope list.

**Employee:** `What are the password requirements?`

→ `CODE_NODE`: `activeTopic = IT` (from `You selected IT and Security` in history) ·
`CHECK_STAGE`: `isITQuestion`, `resolvedQuestion` = the input · `IT_AGENT_CALL` →
`IT_SECURITY_AGENT` searches `DT_IT_SECURITY` · `IT_FORMATTER` renders:

```
Passwords must:

• Be at least 8 characters long
• Include upper case, lower case, numeric and special characters
• Be changed every 90 days
• Not reuse any of the previous 5 passwords

Sources:

- All IT Policies.pdf — Section: Password Policy

You might also want to ask about:

1. What is the process if I forget my password?
2. What are the rules for multi-factor authentication?
3. How should I store credentials for shared systems?

You can reply with just the number to ask that question.

You can say "change topic" at any time to see the full topic menu again.
```

*(followed by the hidden `<!--FOLLOWUPJSON…FOLLOWUPJSON-->` marker)*

**Employee:** `2`

→ `CODE_NODE`: `activeTopic = IT`, so Step B is skipped — this is not a topic switch ·
`CHECK_STAGE` Step 3 reads the marker from the last assistant turn → `resolvedQuestion = "What are
the rules for multi-factor authentication?"` → `isITQuestion` → same agent, full question.

**Employee:** `How many casual leaves do I get?`

→ `CHECK_STAGE`: detected topic `ONBOARDING` ≠ active topic `IT` → `isTopicMismatch`,
`mismatchTopic = ONBOARDING` → `TOPIC_MISMATCH_HANDLER` reminds the user what IT and Security
covers and offers *"change topic"*.

**Employee:** `change topic`

→ `CHECK_STAGE` precedence step 2 → `isMenu` → `SHOW_MENU`. Its text contains *"Please choose a
topic to get started"*, so on the **next** turn the `CODE_NODE` sees `lastMenuIndex >
lastSelectedIndex` and resets `activeTopic` to `NONE` — making a fresh selection possible again.

> Answer content above is illustrative of the response *shape*. Replace with a real transcript
> excerpt from your screenshots.

---

## Design Principles

### Grounded, not hallucinated

Every agent answers **only from its own documents** and says clearly when something isn't covered,
rather than guessing. `foundInDocuments` makes that a structured fact rather than a phrasing
choice, and citations name the exact file and section so any answer can be verified against the
source PDF. Agents are explicitly barred from citing the document tool itself as a source — only
real files count.

### One topic at a time

The conversation stays anchored to whichever topic was last selected, and the system **actively
detects when a question drifts to a different topic**, gently redirecting rather than answering
under the wrong specialist. Each agent's follow-up questions are also constrained to its own
domain, so the suggestions never pull the user toward a topic the current agent cannot serve.

### Deterministic where possible

State tracking — which topic is active, whether this is a fresh selection — is handled by **exact
code logic rather than left entirely to AI judgment**, since that specific task benefits from being
100% consistent.

The general principle: give the language model the jobs that need language understanding
(classifying intent, formatting an answer, extracting a question) and give code or static text the
jobs that need exactness (tracking state, matching menu selections, emitting the menu). Mixing the
two is where reliability leaks.

That principle shows up three times in this workflow — the `CODE` node for state, the `RETURN`
nodes for fixed text, and Rule Zero forbidding the classifier from second-guessing either.

---

## Setup & Deployment

### Prerequisites

- Oracle Fusion Applications environment with **AI Agent Studio** enabled
- Privileges to create Document Tools, agents, and workflows in `HCM.GLOBAL_HUMAN_RESOURCES`
- The 18 policy PDFs, split across the five domain sets

### Import order

Order matters — each layer depends on the one before it.

1. **Document tools** — import all five from [src/tools/](src/tools/). Wait for ingestion to
   complete, then publish each.
2. **Specialist agents** — import all five from [src/agents/](src/agents/). Confirm each has
   exactly one tool attached per the [agent table](#the-five-specialist-agents). Publish each.
3. **Workflow** — import
   [novanix_multi_agent_policy_assistant.wf](src/workflows/novanix_multi_agent_policy_assistant.wf).
   Verify all 36 nodes and that every condition's `false` branch chains to the next test. Publish.

### Testing checklist

| Test | Expected result |
|---|---|
| Say `Hey! I need help` | Welcome plus the five-topic menu |
| Reply `3` | IT and Security description, scope bullets |
| Reply `it & security` instead of `3` | Same result — the allow-list accepts both |
| Ask an on-topic question | Cited answer, `Sources:` block, numbered follow-ups, change-topic line |
| Reply with a bare number | The *correct* suggested question is answered |
| Reply with a number after several similar follow-up lists | Resolves against the **most recent** list only |
| Reply with a number when the last turn had no marker | Falls back to the menu, not a wrong answer |
| Ask a question from a different topic | Mismatch reminder naming the current topic's scope |
| Say `change topic`, then `switch topic`, then `go back` | All three reach the menu (intent-matched) |
| Say `hi` mid-topic | Greeting wins over everything else |
| Ask something off-topic entirely | Polite redirect to the menu |
| Ask about a policy in no document set | `foundInDocuments: false` → honest not-found + referral |
| Describe a live incident (Conduct or IT) | Escalation steps, **no** follow-up suggestions |
| Continue past 15+ turns, then reply with a number | Active topic still correct |

The last row is worth running deliberately — it is the exact failure the `CODE` node exists to
prevent.

---

## Screenshots

> Place images in `docs/screenshots/` inside this folder.

### Workflow design

**Full workflow canvas — all 36 nodes**

![Workflow canvas](docs/screenshots/01-workflow-canvas.png)

**`CODE_NODE` — the JavaScript state-extraction logic**

![Code node](docs/screenshots/02-code-node.png)

**`CHECK_STAGE` — classifier prompt and its 17-field output schema**

![Check stage](docs/screenshots/03-check-stage.png)

**The condition cascade — 13 chained `IF_*` nodes**

![Routing](docs/screenshots/04-routing.png)

**A `RETURN` node — `SHOW_MENU` static text**

![Return node](docs/screenshots/05-return-node.png)

**An `AGENT` call node and its formatter**

![Agent node](docs/screenshots/06-agent-node.png)

### Agent and tool configuration

**The five specialist agents in Agent Studio**

![Agent list](docs/screenshots/07-agent-list.png)

**A specialist agent's instructions and output schema**

![Agent config](docs/screenshots/08-agent-config.png)

**The five document tools with their policy PDFs**

![Document tools](docs/screenshots/09-document-tools.png)

### Working demo

**Greeting and the topic menu**

![Menu](docs/screenshots/10-menu.png)

**Topic selection and its scope description**

![Topic selected](docs/screenshots/11-topic-selected.png)

**A cited answer with numbered follow-up suggestions**

![Cited answer](docs/screenshots/12-cited-answer.png)

**Replying with a number to select a follow-up**

![Follow-up selection](docs/screenshots/13-followup-selection.png)

**Topic mismatch redirect**

![Topic mismatch](docs/screenshots/14-topic-mismatch.png)

**Live incident escalation**

![Incident escalation](docs/screenshots/15-incident-escalation.png)

**Out-of-scope handling**

![Out of scope](docs/screenshots/16-out-of-scope.png)

---

## Known Issues & Limitations

### Worth checking in the running app

**Greeting mid-conversation may not reset the topic.** The `CODE_NODE` resets `activeTopic` only
when it finds *"Please choose a topic to get started"* or *"Please choose one of the following
policy topics"* in history. `SHOW_MENU` emits the first string exactly, and `OUT_OF_SCOPE`'s
prompt models it — but `GREETING_HANDLER` is an LLM instructed to close with *"Please reply with
the topic name or number to continue"*, which matches **neither** reset phrase. So saying `hi`
while a topic is active shows the menu without resetting the lock; a following `3` is then read as
a follow-up number rather than a topic switch. It self-corrects — the unmatched number falls back to
`isMenu`, which fires the real `SHOW_MENU` and resets state — but it costs the user a turn. Adding
the exact reset phrase to `GREETING_HANDLER`'s required output would close this.

**Certification Reimbursement is claimed by two topics.** Menu item 1 lists *"certification
reimbursement"* and the classifier's CAREER definition repeats it, but
`Certification Reimbursement Policy.pdf` actually lives in `DT_COMP_REWARDS` — the topic-4 tool.
A question routed to CAREER on that basis reaches an agent with no such document and returns
`foundInDocuments: false`. The same tension exists for Rewards and Recognition, though the
classifier's disambiguation paragraph resolves that one explicitly. Removing certification
reimbursement from the topic-1 menu text and the CAREER definition would align routing with the
document layout.

**`isActiveIncident` is only half-wired on two agents.**
`COMPENSATION_REWARDS_AND_TRAVEL_AGENT` and `ONBOARDING_AND_CULTURE_AGENT` both declare the field
in their output schema with a description, but neither prompt contains a *"Critical, active
incident handling"* section telling the agent what to do when it is true. Only
`CONDUCT_COMPLIANCE_AGENT` and `IT_SECURITY_AGENT` have real escalation behaviour.

**Inconsistent `maxInteractions`.** Three agents allow 3 tool interactions, the compensation agent
allows 10, and the onboarding agent has no value set at all. If that is not deliberate, worth
aligning.

**Three copies of the menu text.** `SHOW_MENU` (static), `GREETING_HANDLER` (prompt), and
`OUT_OF_SCOPE` (prompt) each define the five topics independently, and their wording already
differs slightly. Since the `CODE_NODE` pattern-matches against this text, drift between the three
has real consequences.

**Workflow name still reads Novanix.** The organisation is **Norvex Technologies** — that is the
name used throughout the policy documents, the agent prompts, and this README. The workflow itself
is still coded `NOVANIX_MULTI_AGENT_POLICY_ASSISTANT` (and exports as
`novanix_multi_agent_policy_assistant.wf`), which is a leftover to rename in Agent Studio before
this goes in front of users. Nothing functional depends on it — no prompt, regex, or node
reference reads the workflow code — so it is a display-name fix only.

**A vestigial `followupPrompt`.** The workflow carries a platform follow-up-generation prompt even
though `followupEnabledFlag` is `false` — follow-ups are entirely agent-generated. Harmless, but
dead configuration.

### Inherent limitations

- **No persistent memory between turns** beyond the visible chat — the entire workflow re-runs from
  start to end on every message, re-deriving state from history each time.
- **The `CODE` node matches only the strings it was written for.** Rewording a menu line or a
  `You selected …` prefix breaks topic tracking until the regex is updated in step.
- **Five fixed topics.** Adding a sixth means touching the menu text, the `CODE` node allow-lists,
  the classifier schema and prompt, and the condition cascade together.
- **Only the most recent suggestion list is selectable** — deliberate, but a user cannot pick an
  option from earlier in the conversation.
- **One topic at a time by design** — a question genuinely spanning two domains (relocation *and*
  travel reimbursement) has to be asked twice.
- **Onboarding is the fall-through branch**, so a classifier that ever returns all-false would send
  the question to the Onboarding agent rather than erroring.

### Possible enhancements

| Enhancement | Value |
|---|---|
| Deep links into the source PDF page | One-click verification of any citation |
| Suggest the right topic on mismatch, with a one-tap switch | Turns a redirect into a shortcut |
| Cross-topic questions via multi-agent fan-out | Handles genuinely spanning questions |
| Single source of truth for the menu text | Removes the three-copy drift risk |
| Region / entity / role metadata filters | Correct policy variant per employee |
| Logging of `foundInDocuments: false` questions | Reveals real gaps in the policy library |
| HRMS write-back actions | Move from *answering about* leave to *applying for* it |
| Slack or Teams surface | Meets employees where they already work |
| Feedback capture on each answer | Enables continuous evaluation and prompt tuning |

---

## Summary

| Aspect | Implementation |
|---|---|
| Platform | Oracle AI Agent Studio (Fusion HCM) |
| Organisation | Norvex Technologies |
| Pattern | Menu-routed multi-agent RAG |
| Workflow | `data_pipeline`, 36 nodes *(code still `NOVANIX_…` — rename pending)* |
| Node types | `START`, `CODE`, `LLM` ×9, `CONDITION` ×13, `RETURN` ×6, `AGENT` ×5, `END` |
| Agents | 5 `WORKER` agents, one Document Tool each |
| Documents | 18 policy PDFs across 5 tools |
| State tracking | Deterministic JavaScript `CODE` node, not AI judgment |
| Routing | LLM classifier (14 flags, exactly one true) → 13-condition cascade |
| Citations | Exact `fileName` + `section`, tool name never cited |
| Interaction model | Topic menu, then chat, with numbered follow-up selection |
| Follow-up mechanism | Hidden `<!--FOLLOWUPJSON…-->` marker, read from the last turn only |
| Scope control | Out-of-scope redirect plus active-topic mismatch detection |
| Safety | Live-incident escalation with follow-up suppression (Conduct, IT) |
