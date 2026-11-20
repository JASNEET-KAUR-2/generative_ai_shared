# From model to application: choosing and designing generative AI systems

## Learning objectives

After this lecture, you should be able to:

1. Classify a proposed application by the kind of generation it needs, and give examples from several industries.
2. Judge whether a task suits generative AI using five criteria: the cost of an error, how easily the output can be checked, who reviews it, where the knowledge comes from, and volume.
3. Choose between prompting, retrieval-augmented generation (RAG), fine-tuning, and a system with no generative model at all.
4. Read a case study critically: what was deployed, what was measured, and what the evidence does and does not show.
5. Write a design for a generative AI application: users and tasks, scope, requirements, architecture, failure modes and guardrails, and an evaluation plan written before building.

**Prerequisites.** [RAG with speech](../../../module3/lectures/rag/rag_speech.ipynb) (Module 3). It is the running example, and this lecture reuses its measured results. Also [fine-tuning GPT-2](../../../module3/lectures/fine_tuning/gpt2_fine_tuning.ipynb) and [PEFT and LoRA](../../../module3/lectures/peft/peft_lora.ipynb) (what fine-tuning changes and what it costs), and [evaluating fine-tuned models](../../../module3/labs/qlora_dialogsum/qlora_dialogsum_lab.ipynb) (comparisons against a baseline, human rating).

Slides: [applications_design.pptx](../../slides/applications_design/applications_design.pptx).

## Contents

- [Part A: Where generative AI fits](#part-a-where-generative-ai-fits)
  1. [The problem: six proposals, one team](#1-the-problem-six-proposals-one-team)
  2. [What generative AI applications do](#2-what-generative-ai-applications-do)
  3. [When a task suits generative AI](#3-when-a-task-suits-generative-ai)
  4. [Prompting, RAG, fine-tuning, or no generative model](#4-prompting-rag-fine-tuning-or-no-generative-model)
  5. [Case study: an assistant for support agents](#5-case-study-an-assistant-for-support-agents)
  6. [Counterexample: a chatbot that answered on its own](#6-counterexample-a-chatbot-that-answered-on-its-own)
  7. [Back to the six proposals](#7-back-to-the-six-proposals)
- [Part B: Designing the application](#part-b-designing-the-application)
  8. [Users and their tasks](#8-users-and-their-tasks)
  9. [Scope and requirements](#9-scope-and-requirements)
  10. [Architecture](#10-architecture)
  11. [Failure modes and guardrails](#11-failure-modes-and-guardrails)
  12. [The evaluation plan, written before building](#12-the-evaluation-plan-written-before-building)
  13. [The design document](#13-the-design-document)
- [Summary](#summary) · [Check your understanding](#check-your-understanding) · [Assignment: a design document](#assignment-a-design-document) · [References](#references)

---

# Part A: Where generative AI fits

## 1. The problem: six proposals, one team

The food-delivery company from Module 3 now has a working prototype. It is the RAG assistant from [Unit 3.7](../../../module3/lectures/rag/rag_speech.ipynb), which answers policy questions from 13 help-centre articles, cites its sources, and has a voice front end. Management wants more of this, and different teams have sent in six proposals:

| # | Proposal | Proposed by |
| --- | --- | --- |
| P1 | Suggest replies to **human support agents** while they chat with customers | Support operations |
| P2 | Let **customers** ask policy questions to the RAG assistant directly, by text or voice | Product |
| P3 | Write menu descriptions for restaurants from a dish name and a list of ingredients | Restaurant partnerships |
| P4 | Summarize each week's complaint chats into a report for managers | Quality team |
| P5 | Let the assistant **issue refunds** itself when a customer reports a missing item | Finance (to cut costs) |
| P6 | Predict the delivery time shown to customers at checkout | Logistics |

There is one engineering team, and it can build one or two of these well. Which ones should it build first? Which should it not build at all?

Part A gives you a way to answer this question with reasons, not guesses. Part B then designs one of the systems properly.

## 2. What generative AI applications do

Industries look different, but most applications reuse a small number of **task types**. Each type has its own typical failure, and each is checked differently.

| Task type | Input → output | Examples across industries | Typical failure | How the output can be checked |
| --- | --- | --- | --- | --- |
| **Generate** | Brief → new text, image, or audio | Marketing copy, product images, game assets, synthetic training data | Bland, off-brand, or infringing content | A person judges quality; hard to automate |
| **Transform** | Text → the same content in another form | Translation, rewriting for a reading level, converting a description to code or a query | Meaning silently changed | Compare with the source; partly automatable |
| **Summarize** | Long text → short text | Meeting notes, patient-visit notes, legal discovery, complaint reports | Omitted or invented facts | Check each claim against the source; slow for long inputs |
| **Extract** | Unstructured text → structured fields | Invoice fields, insurance claims, résumé parsing | A wrong value in a valid-looking field | Validate formats and ranges; compare a sample with labels |
| **Converse** | A dialogue → the next turn | Tutoring, customer support, internal help desks | Drifts off topic; agrees with anything | Review transcripts; hard in real time |
| **Retrieve and answer** | Question + documents → a grounded answer | Policy and HR questions, technical manuals, research search | Wrong passage retrieved; claims not supported by the cited source | Recall@k for retrieval; citation and fact checks for answers (Unit 3.7) |
| **Write code** | Description or partial code → code | Programming assistants, spreadsheet formulas, data-analysis scripts | Code that runs but is wrong or insecure | Tests, type checks, review: the easiest output to verify automatically |

The industry examples show where these task types appear. They are not claims that a particular company uses them successfully.

Two observations from the table guide the rest of Part A:

- **The model is rarely the hard part.** A capable model can do all seven task types. What differs is how expensive it is when the output is wrong, and how expensive it is to find out.
- **Some outputs check themselves, others do not.** Code can be run against tests. A summary must be read against its source. Menu copy can only be judged by a person. The harder the output is to check, the more the application depends on people to check it.

## 3. When a task suits generative AI

A generative model produces fluent output cheaply and quickly, and is sometimes wrong in ways that are hard to see (the closed-book answers in Unit 3.7 were fluent, confident, and invented). So whether a task suits it depends less on what the model *can* produce than on what happens when it is wrong. Five questions capture this.

| Criterion | Question to ask | Favourable | Unfavourable |
| --- | --- | --- | --- |
| **1. Cost of an error** | What happens if one output is wrong? | A weak draft, quickly corrected | Money paid out, a health or safety risk, a legal commitment |
| **2. Verifiability** | Can someone or something check the output cheaply? | Tests, a known answer, an expert reads it in seconds | Checking takes as long as doing the task |
| **3. Human in the loop** | Who sees the output before it has an effect? | A trained employee who can edit or discard it | Nobody: it goes straight to a customer or a system |
| **4. Source of knowledge** | Where does the needed knowledge come from? | General knowledge, or documents the system can retrieve | Private facts the model cannot see; facts that must be exact and current |
| **5. Volume and value** | How often is the task done, and what is each instance worth? | Frequent, repetitive, and time-consuming for people | Rare, so the build and evaluation cost is never repaid |

Data and privacy constraints cut across all five. Can the inputs legally be sent to the model? Do they contain personal data? Is the model's licence compatible with commercial use?

### Cost of error × verifiability

The first two criteria interact. Draw them as two axes:

```text
                         output is EASY to check                      output is HARD to check
                 ┌──────────────────────────────────────┬──────────────────────────────────────┐
  error is       │  AUTOMATE, with spot checks          │  ASSIST: a person reviews drafts     │
  CHEAP          │  code with tests, field extraction   │  marketing copy, menu descriptions,  │
                 │  validated against a schema          │  first drafts of emails              │
                 ├──────────────────────────────────────┼──────────────────────────────────────┤
  error is       │  AUTOMATE THE CHECK, then decide     │  AVOID, or redesign until one of the │
  COSTLY         │  a rule or a person verifies every   │  other boxes applies                 │
                 │  output before it acts (refund       │  free-text medical, legal or         │
                 │  amounts recomputed by rules)        │  financial advice sent unreviewed    │
                 └──────────────────────────────────────┴──────────────────────────────────────┘
```

The bottom-right box is not "never". It means the design must **move** the task into another box. There are three ways to do it:

- **Add a reviewer.** An expert reads every output before it is used. The model now drafts and the person decides.
- **Narrow the output.** Generate a choice among approved answers instead of free text, or let fixed rules compute the critical number.
- **Make it checkable.** Ask for citations, then verify that the cited passage supports the claim.

### A back-of-the-envelope check

Small numbers often settle a debate faster than opinions. The figures below are **invented for illustration**; replace them with your own estimates.

Suppose proposal P2 answers $N = 10{,}000$ policy questions a day, and testing shows that a fraction $e = 0.05$ of answers contain a wrong policy fact. If a fraction $h = 0.1$ of the customers who receive a wrong answer act on it and the company has to compensate them at an average cost of $c = ₹400$, the expected daily cost of errors is

$$N \cdot e \cdot h \cdot c = 10{,}000 \times 0.05 \times 0.1 \times ₹400 = ₹20{,}000.$$

Compare this with what the assistant saves. If it handles questions that would otherwise cost ₹15 each in agent time, it saves $10{,}000 \times ₹15 = ₹1{,}50{,}000$ a day. In this example, errors cost far less than the assistant saves. But the calculation counts only money, and it leaves out what one dangerous answer can cost, for example about allergens (Section 11). Totals like these can hide a rare, severe failure. Those failures must be designed out, not averaged away.

## 4. Prompting, RAG, fine-tuning, or no generative model

Once a task looks suitable, the next decision is *how* to adapt a model to it. Unit 3.7 compared three methods: **prompting** changes the instructions, **RAG** changes the context the model reads, and **fine-tuning** changes the model's weights. A design review needs a fourth option: **no generative model at all**.

| Need | Best first choice | Why |
| --- | --- | --- |
| A format, tone, or simple instruction | **Prompting** | Cheapest to try and change; always the baseline |
| Facts that are private, large, or change often; answers that must cite a source | **RAG** | Update the documents, not the model; answers are traceable |
| A consistent style or output format, domain language, or a skill the prompt cannot teach | **Fine-tuning** (often LoRA/QLoRA, Units 3.5–3.6) | Changes behaviour permanently; needs data, compute, and a new evaluation for every change |
| A fixed set of categories | **A classifier**, not a generator | Easier to evaluate, cannot invent a new category |
| A number predicted from data (delivery time, demand) | **A regression model** | The output is a number, not language |
| An exact value from a database or a rule (refund amount, order status) | **A query or a rule** | Must be exactly right every time; a model adds only risk |

They combine. A fine-tuned model can supply the company's tone, RAG can supply today's policies, and a rule can compute the refund. **Always start with the simplest version that might work.** Usually that means prompting plus retrieval, measured on an evaluation set. Fine-tune only when the measurements show a gap that prompting and retrieval cannot close.

## 5. Case study: an assistant for support agents

**Source:** Brynjolfsson, Li and Raymond, *Generative AI at Work* (Quarterly Journal of Economics, 2025; figures below are from the November 2024 arXiv version).

### What was deployed

The study followed a Fortune 500 company that sells business-process software. Its customer-support agents, most of them working from offices in the Philippines, help customers by chat. The company rolled out an AI assistant to these agents in stages. The assistant was:

- **built on a GPT-family model**, fine-tuned on a large set of past support conversations labelled with their outcomes (was the issue resolved, how long did it take), with extra weight on conversations handled by the company's top-performing agents;
- **shown only to the agent, never to the customer.** It read the conversation and offered suggested replies and links to the company's technical documentation;
- **optional.** Agents remained responsible for the conversation and were free to ignore or edit every suggestion.

The data cover 5,172 agents. Because different groups of agents got access at different times, the researchers could compare agents with and without the tool over the same period, which is stronger evidence than a simple before-and-after comparison.

### What was found

| Measure | Result |
| --- | --- |
| Issues resolved per hour, all agents | **About 15% higher** with the assistant |
| Less experienced and lower-skilled agents | About **30%** more issues resolved per hour, with better quality. Agents with two months' experience and the tool performed as well as agents with over six months' experience and no tool |
| The most experienced and highest-skilled agents | Small gains in speed, and **small declines in quality** |
| How often agents used the suggestions | On average **38%** of suggestions; agents ignored most of them |
| Customers and agents | Customers were more polite and asked less often to speak to a manager; fewer agents quit, mostly among the newer ones |

### Reading it with the criteria of Section 3

| Criterion | This deployment |
| --- | --- |
| Cost of an error | Low per suggestion: a bad suggestion costs the agent a few seconds |
| Verifiability | High: a trained agent judges a suggested reply at a glance |
| Human in the loop | **Always.** Nothing reached a customer without an agent choosing to send it |
| Source of knowledge | Past conversations (through fine-tuning) and links to documentation (retrieval) |
| Volume and value | Very high: millions of chats, and agent time is the main cost of support |

It sits in the top-right box of the grid: errors are cheap and the outputs are hard to check automatically, so a person reviews every draft. That is a big part of why it worked.

### What the evidence does not show

- **One company, one task.** Technical support for business software, with agents trained in a particular way. The results may not transfer to other domains or to customers using the tool directly.
- **Productivity is not correctness.** "Issues resolved per hour" was measured carefully, but quality was only measured indirectly, through customer satisfaction and resolution rates. The small quality decline for top agents shows that help for some can be a cost for others.
- **The value came from people choosing.** Agents used only about 38% of the suggestions. An application that sent every suggestion automatically would be a different system with different results.

## 6. Counterexample: a chatbot that answered on its own

**Source:** *Moffatt v. Air Canada*, 2024 BCCRT 149, British Columbia Civil Resolution Tribunal (Canada), February 2024.

In 2022, a customer whose grandmother had died asked the chatbot on Air Canada's website about bereavement fares. The chatbot said he could buy a full-price ticket and apply for the reduced bereavement fare afterwards. The airline's own policy page, which the chatbot's answer linked to, said that bereavement fares could not be claimed after travel. When the airline refused the refund, the customer took the case to the tribunal.

The airline argued that it was not responsible for what the chatbot said. The tribunal member called this a "remarkable submission". The chatbot was "still just a part of Air Canada's website". The airline was responsible for all the information on its website, and it was not reasonable to expect a customer to check one part of the website against another. The tribunal found the airline liable for negligent misrepresentation and ordered it to pay CA$812.02, which comprised CA$650.88 in damages plus interest and tribunal fees.

The amount was small; the lesson is not. Compare it with the case study:

| | Support-agent assistant (Section 5) | Airline chatbot |
| --- | --- | --- |
| Who reads the output first | A trained agent | The customer |
| What the output is about | How to phrase a reply | A **policy commitment** about money |
| Can the reader check it | Yes, quickly | In principle, but the tribunal held they should not have to |
| Who bears the cost of an error | Agent time | The company, legally |

The airline case shows proposal P2 at its riskiest. A customer-facing assistant that states policy is making statements for the company. This is a decision for Canada under its law, and it is not a general rule for every jurisdiction. But the design consequence holds everywhere: **a deployed assistant's answer should be treated as the company's own statement, and the system must be designed accordingly.** Recall that the Unit 3.7 assistant, in its saved run, told a customer that a dish "does not contain nuts" when the source said nut-free dishes could not be guaranteed.

## 7. Back to the six proposals

Score each proposal on the five criteria from Section 3 (✓ favourable, ~ mixed, ✗ unfavourable). The scores are judgements meant to structure a discussion. They are not a formula, and you may reasonably disagree with some of them.

| # | Proposal | Error cost | Verifiable | Human in loop | Knowledge | Volume | Recommendation |
| --- | --- | --- | --- | --- | --- | --- | --- |
| P1 | Reply suggestions for agents | ✓ | ✓ | ✓ | ✓ RAG over policies | ✓ | **Build first.** Same pattern as the case study; the agent catches errors |
| P2 | Customer-facing policy assistant | ✗ | ~ citations help | ✗ | ✓ RAG | ✓ | **Build second, with guardrails** (Part B), after P1 has measured the RAG pipeline on real questions |
| P3 | Menu descriptions | ✓ | ✓ the restaurant approves | ✓ | ~ ingredients given; allergens must not be invented | ~ | Worth doing as a draft the restaurant edits. Never let it state allergen information |
| P4 | Weekly complaint summaries | ✓ | ~ | ✓ managers | ✓ the chats themselves | ~ weekly | Reasonable. Check summaries against counts computed from the data, and remove personal data first |
| P5 | Assistant issues refunds | ✗ money | ✓ by rules | ✗ | — | ✓ | **Do not let the model decide.** Rules compute eligibility and amount; the model may only explain the outcome. Unit 3.7 showed that retrieved text can inject instructions, and here an injection could move money |
| P6 | Delivery-time prediction | — | ✓ | — | — | ✓ | **Not a generative AI problem.** A regression model on order, distance, and traffic data |

The result is a typical pattern. Internal tools with a person in the loop go first. Customer-facing generation comes later and with more safeguards. Actions with consequences go through rules, not text generation. And some proposals do not need generative AI at all.

---

# Part B: Designing the application

We now design proposal P2: the customer-facing version of the Unit 3.7 voice assistant. A design answers six questions in order. Each answer constrains the next, and every choice should be traceable to a requirement or a measured result.

```text
  1 Users & tasks ──► 2 Scope & requirements ──► 3 Architecture ──► 4 Failure modes & guardrails ──► 5 Evaluation plan
         ▲                                                                                                  │
         └──────────────────────── revise when measurements disagree with assumptions ◄─────────────────────┘
```

## 8. Users and their tasks

Start from the people, not the model. For each user group, write what they are trying to do and what would count as success for them.

| User | What they are trying to do | Success for them | What they must never get |
| --- | --- | --- | --- |
| **Customer**, mid-order or just after delivery, often on a phone, sometimes speaking | Find out what happens next: refund timing, compensation, how to report a problem | A correct, short answer with the next step, in under a minute | A confident wrong answer about money, food safety or allergens |
| **Support agent** | Take over when the assistant hands off | The conversation so far and the assistant's sources, so the customer does not have to repeat themselves | A handoff with no context |
| **Policy team** | Keep the help-centre articles current | Edits take effect in the assistant the same day | An assistant that keeps giving the old policy |

Two questions often missed:

- **Who is excluded?** Voice input helps customers who find typing hard, but speech recognition is usually less accurate for some accents and for speakers of languages the model was trained on less. One audit of five commercial speech-recognition systems found substantially higher word error rates for Black speakers than for white speakers in the United States (Koenecke et al., 2020). An Indian customer base speaks many accents and switches between languages within a sentence. Measure accuracy for each group of users. An average can hide the groups that are served badly.
- **What does the user do with the answer?** Customers act on it, so a policy answer works like a promise (Section 6).

## 9. Scope and requirements

### Scope

Scope is defined as much by what the assistant must *not* do as by what it does.

| In scope | Out of scope (hand off to a person, or refuse) |
| --- | --- |
| Answer questions about the published policies, citing the article | Any question the articles do not answer |
| Explain how to report a problem and what happens next | Taking actions: issuing refunds, cancelling orders, changing addresses |
| Text and voice, in English (first release) | Medical advice; allergen guarantees for specific dishes |
| Hand off to an agent on request or when unsure | Anything about other customers or other accounts |

### Functional requirements

What the system does, stated so that each requirement can be tested:

- **F1.** Answers only from the indexed help-centre articles, and cites the article id for every fact.
- **F2.** For a question the articles do not answer, replies with the fixed refusal and offers a handoff.
- **F3.** Questions about allergens, food safety, and payments get a fixed, reviewed answer and an offer to talk to an agent, not a generated answer (justified in Section 11).
- **F4.** Accepts a spoken question and can read the answer aloud.
- **F5.** A handoff passes the transcript and the retrieved sources to the agent.

### Non-functional requirements

How well it must work, and under which constraints. The targets below are proposals for this design; a real team would set them with the business.

| Requirement | Target (proposed) | Where it comes from |
| --- | --- | --- |
| **Latency** | Voice: first audio within about 2 s; text: first words within about 1 s | Unit 3.7 measured 0.8–3.4 s for retrieval plus generation alone. Longer silences feel broken in a spoken conversation, so answers must be short and streamed |
| **Freshness** | A policy edit is live in the assistant the same day | Policies change every few weeks; this rules out fine-tuning as the source of policy facts |
| **Privacy** | No names, phone numbers, or addresses in the model's prompt or in logs | Customers type personal details into chat. Redact before the model sees the text and before anything is stored |
| **Licensing** | Every model allows commercial use | The MMS-TTS model used in Unit 3.7 is licensed CC-BY-NC 4.0, **non-commercial**. The product needs a different text-to-speech model |
| **Cost** | Cost per answered question below the cost of an agent answering it | Section 3; measure tokens per question in testing |
| **Availability** | If the model is down, fall back to search over the articles and a handoff | Customers must still get help |

The licensing row is a reminder that requirements can rule out a model that worked well in the prototype.

## 10. Architecture

The architecture follows from the requirements. Each component exists because a requirement or a measured failure calls for it.

```text
                                   ┌──────────────────────────── trusted index: help-centre articles only (F1)
                                   │                               re-indexed on every policy edit (freshness)
                                   ▼
 spoken ─► speech-to-text ─► redact ─► topic ─┬─► retrieve top-k ─► LLM: answer only ─► output ─► text-to-speech ─► customer
 or typed   (domain terms     personal  router │    chunks            from sources, cite   check       (commercial
 question   prompted, F4)     data             │                      ids, or refuse (F2)  (§11)       licence)
                                               │                                             │
                                               ├─► allergen / food-safety / payment ─► fixed reviewed answer + handoff (F3)
                                               │                                             │
                                               └─────────────────────────────────────────────┴─► hand off to agent with
                                                                                                  transcript + sources (F5)
```

Decisions, and the requirement or result behind each:

| Decision | Why |
| --- | --- |
| RAG, not fine-tuning, for policy facts | Freshness, and citations (F1). The weights never hold a policy |
| A small instruction-tuned model, possibly LoRA-tuned later for tone and brevity | Latency and cost. Fine-tune only if evaluation shows answers are too long or off-tone (Section 4) |
| Only trusted articles in the index | Unit 3.7's injected restaurant profile changed a refund answer; text outsiders can write never enters this index |
| A topic router before generation | F3: the highest-stakes topics never get a generated answer. It can be a small classifier: a category output, not free text (Section 4) |
| An output check after generation | Unit 3.7: the guarded system prompt did not stop the injection, and the output check flagged it. A check for payment requests, UPI IDs, links, and phone numbers stays even with a trusted index |
| Redaction before the model and before logging | Privacy. It also keeps personal data out of any future fine-tuning set built from the logs (Unit 3.3) |
| The assistant cannot take actions | Scope. An assistant that can only write text can be tricked into saying something wrong; one that can act can be tricked into doing something wrong |

## 11. Failure modes and guardrails

List what can go wrong *before* building, using every result you already have. Unit 3.7's saved run gives concrete evidence for this design:

| Failure (observed in Unit 3.7 unless marked) | Guardrail in this design | What still gets through |
| --- | --- | --- |
| **Retrieval miss:** "My biryani never came" retrieved food-safety chunks, and the answer told the customer to file a food-safety complaint | Hybrid retrieval and re-ranking; a minimum similarity score below which the assistant refuses; real customer phrasings in the evaluation set | Misses with confident scores |
| **Answer contradicts its source:** "the paneer tikka does not contain nuts", where the source said no dish is guaranteed nut-free | Allergens are routed to a fixed answer (F3) | Allergen questions the router misclassifies, so measure the router's recall on them |
| **Hallucination on unanswerable questions:** "Yes, we deliver groceries"; Bitcoin answered with a false citation | Refusal instruction; a check that each cited article supports the claim (for example, a second model call that compares claim and source) | Plausible claims that the checker accepts |
| **Prompt injection** through text an outsider wrote | Trusted-only index; output check; no tools | New attack forms the output check does not anticipate |
| **Speech recognition mishears domain terms:** "UPI account take" became "upiacomptake" | Give Whisper the product's vocabulary as a prompt; show the transcript to the user so they can correct it | Accents and noisy phones worse than the clean synthetic test audio |
| **Stale policy** *(anticipated)* | Re-index on every article edit; show the article's last-updated date | Cached answers, if caching is added later |
| **Answers too long for voice** (the model ignored "two or three sentences") | Token limit; stream sentence by sentence to text-to-speech | Long answers to complex questions |

Notice the pattern in the third column. **No guardrail is complete**, which is why they are layered, and why the remaining risk has to be measured and monitored after launch (Unit 4.4).

## 12. The evaluation plan, written before building

Write the evaluation plan before you build. Otherwise the test questions tend to become the ones the system already answers well. Unit 3.7 wrote its evaluation set before building the index for exactly this reason.

### The test set

| Slice | What it contains | Why |
| --- | --- | --- |
| Answerable | Several questions per article, in customers' own words, written by people who did not write the articles | Measures the main job; paraphrases test retrieval honestly |
| Unanswerable | Questions near the topic that no article answers | Measures hallucination; Unit 3.7 failed two of three |
| High-stakes | Allergen, food-safety, and payment questions, phrased many ways | Measures the router (F3) |
| Adversarial | Injection attempts, requests for actions, questions about other accounts | Measures guardrails |
| Voice | Real recordings across accents, phones, and noise levels | Synthetic speech is a best case (Unit 3.7) |
| Regression | Every failure found later, added as a test | Keeps fixed bugs fixed (Unit 4.4) |

### Metrics per stage, and acceptance thresholds

Evaluate each stage separately, as well as end to end. A failure must be traced to the stage that caused it.

| Stage | Metric | Example launch threshold (illustrative) |
| --- | --- | --- |
| Speech-to-text | Word error rate on real recordings, **per accent group** | No group much worse than the average |
| Topic router | Recall on high-stakes questions | Close to 1.0: a missed allergen question is the costliest error |
| Retrieval | Recall@3, mean reciprocal rank (MRR) | Set from a baseline, e.g. recall@3 ≥ 0.9 |
| Answers | Correct fact, correct citation, refusal rate on unanswerable questions, wrong-refusal rate on answerable ones | Refusal on unanswerable ≥ 0.95 |
| Safety | Injection and action requests that produce a harmful answer | Zero in the adversarial set |
| End to end | Latency (median and 95th percentile), cost per question, human ratings of a sample | Within the non-functional targets |

The thresholds above only illustrate the form; a team sets them with the business and from a baseline (Units 3.7–3.8). Three rules from Module 3 apply here too:

- **Compare against a baseline.** Here the baseline is today's help-centre search and the agents' answers. A RAG assistant that is worse than search is not worth deploying.
- **Automatic checks are a filter, not a verdict.** People read a sample of answers, using written criteria: is it correct, supported by the cited source, complete, and safe?
- **Do not tune on the final test set.** Keep a held-out part of the test set for the launch decision.

## 13. The design document

A design document collects these decisions on one or two pages so that others can review them before anyone writes code. A useful template:

1. **Problem and users.** Who, what task, what success means for them, and what they must never get.
2. **Why generative AI, and which kind.** The five criteria; prompting, RAG, fine-tuning, or none; and the baseline you will compare against.
3. **Scope.** In scope and out of scope, and what happens at the boundary.
4. **Requirements.** Functional (each testable) and non-functional (latency, freshness, privacy, licensing, cost), with targets.
5. **Architecture.** A diagram, and the requirement behind each component.
6. **Failure modes and guardrails.** What can go wrong, what limits it, and what still gets through.
7. **Evaluation plan.** Test-set slices, per-stage metrics, launch thresholds, human review.
8. **Open questions and risks.** What you do not know yet, and how you will find out.

The design is a hypothesis. Evaluation and monitoring (Unit 4.4) will show where it is wrong, and the document is revised when they do.

---

## Summary

| Question | How to answer it |
| --- | --- |
| What kind of generation is this? | Name the task type (Section 2); it predicts the typical failure and how to check the output |
| Should we use generative AI here? | Cost of an error, verifiability, human in the loop, source of knowledge, volume (Section 3) |
| How should the model get the knowledge or behaviour? | Prompting first; RAG for changing facts; fine-tuning for behaviour; rules, classifiers or regression when the output is not open-ended text (Section 4) |
| What does a case study prove? | Only what was measured, in that setting. Ask who reviewed the outputs and what was not measured (Sections 5–6) |
| How do we design it? | Users → scope and requirements → architecture → failure modes → evaluation plan, each traceable to the one before (Part B) |

The two cases from Part A show the central lesson. **How a generative model is placed in a system matters as much as the model itself.** It matters who sees the output first, what the output is allowed to commit to, and how errors are caught.

## Check your understanding

1. For each task type in Section 2, give one example from an industry not listed in the table, and say how its output could be checked.
2. Place these in the cost-of-error × verifiability grid and justify each: (a) generating unit tests for existing code; (b) translating a drug's dosage instructions; (c) generating images for a children's story app; (d) extracting the invoice total from scanned bills for automatic payment. For any you place bottom-right, propose a redesign that moves it to another box.
3. Redo the calculation in Section 3 for allergen questions only: 200 a day, a wrong-answer rate of 2%, and one serious incident per 1,000 wrong answers. Why is the expected daily cost a poor way to decide whether the assistant may answer allergen questions?
4. The case-study assistant raised productivity by about 15% on average but slightly reduced the quality of the best agents' conversations. Suggest one explanation, and one design change that might avoid it.
5. In the case study, agents used about 38% of suggestions. Is a low adherence rate a sign that the tool is failing? What would you want to know before deciding?
6. The airline's chatbot linked to the correct policy page while giving a wrong answer. Why did the link not protect the airline, and what does this imply for citations in the Unit 3.7 assistant?
7. Proposal P5 (the assistant issues refunds) was rejected in its proposed form. Design a version that uses a language model safely. What does the model do, what do rules do, and what does a person do?
8. A colleague says: "Our fine-tuned model scored ROUGE-L 0.45 on the test set, so we should replace RAG with it for the policy assistant." Give three reasons from this lecture why that does not follow.
9. Write two functional requirements and two non-functional requirements for proposal P4 (weekly complaint summaries), each phrased so that it can be tested.
10. Why must the evaluation plan be written before the system is built? Describe one way a test set written afterwards could make the system look better than it is.

## Assignment: a design document

Choose an application for an organization you know: a college, a hospital, a shop, a government service, or a company. It must not be the food-delivery assistant from this lecture. Write a design document of at most two pages, following the template in Section 13.

A strong design document:

- names specific users and what success means for them, including who might be served badly;
- justifies the use of generative AI with the five criteria, and names a baseline that does not use generative AI;
- has testable functional requirements and non-functional requirements with targets;
- traces each architecture component to a requirement;
- lists at least four failure modes, each with a guardrail and what still gets through;
- gives an evaluation plan with test-set slices, per-stage metrics and a human-review step;
- uses only public, appropriately licensed or clearly synthetic examples, with no real personal data.

Your instructor will give the deadline and marks.

## References

- Brynjolfsson, E., Li, D., & Raymond, L. R. (2025). Generative AI at work. *The Quarterly Journal of Economics*, 140(2), 889–942. <https://doi.org/10.1093/qje/qjae044> · Preprint: [arXiv:2304.11771](https://arxiv.org/abs/2304.11771) (version 2, November 2024, used for the figures above) · [NBER Working Paper 31161](https://www.nber.org/papers/w31161)
- *Moffatt v. Air Canada*, 2024 BCCRT 149. British Columbia Civil Resolution Tribunal. [Decision text](https://decisions.civilresolutionbc.ca/crt/crtd/en/525448/1/document.do). This is a decision of a Canadian small-claims tribunal under British Columbia law, not a general rule in other jurisdictions.
- Koenecke, A., Nam, A., Lake, E., et al. (2020). Racial disparities in automated speech recognition. *Proceedings of the National Academy of Sciences*, 117(14), 7684–7689. <https://doi.org/10.1073/pnas.1915768117>
- OWASP Foundation. *OWASP Top 10 for LLM Applications* (2025): LLM01 Prompt Injection, LLM06 Excessive Agency. <https://genai.owasp.org/llm-top-10/>
- Unit 3.7 of this course, [RAG with speech](../../../module3/lectures/rag/rag_speech.ipynb): the source of every measured result about the running example (one saved Kaggle run with Qwen2.5-1.5B-Instruct and greedy decoding).

**Further reading**

- Lewis, P., Perez, E., Piktus, A., et al. (2020). Retrieval-augmented generation for knowledge-intensive NLP tasks. *NeurIPS 2020*. [arXiv:2005.11401](https://arxiv.org/abs/2005.11401)
- Amershi, S., Weld, D., Vorvoreanu, M., et al. (2019). Guidelines for human-AI interaction. *CHI 2019*. <https://doi.org/10.1145/3290605.3300233>
