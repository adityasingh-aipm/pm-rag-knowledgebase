# Building a Production-Grade RAG System: From Tracing to Trust

*How I designed observability, evaluation, and decisioning for reliable GenAI products.*

---

## 1. Why This Artifact Exists

GenAI systems fail silently.

They often produce answers that *look* correct but are:
- not grounded in retrieved data,
- unsafe in edge cases,
- or impossible to debug after the fact.

As a Product Manager, I believe shipping GenAI is not about demos or prompts, it’s about **shipping trust**.

This artifact documents how I designed and implemented a **production-grade RAG system** with:
- full observability,
- explicit decisioning,
- online and offline evaluation,
- and clear ownership when things go wrong.

The goal was not to optimize a single metric, but to build a **reliable system** that can be debugged, evaluated, and improved over time.

Prod-online eval and Observability- https://colab.research.google.com/drive/1HykGd-FxqaFixBXPSeggZyAjof6C0XSG#scrollTo=8061f312

<img width="1436" height="810" alt="image" src="https://github.com/user-attachments/assets/b8edbb45-dffe-4f26-8c3c-d066789883d0" />



Offline-eval- https://colab.research.google.com/drive/19U49GORZ7b8RMiQm5_QU7Sv-DLDwO3Lp#scrollTo=YOKKBhyA_HX6

<img width="1432" height="813" alt="image" src="https://github.com/user-attachments/assets/a78def1c-d548-48b9-a7b4-b1a4998c3db1" />

---

## 2. The Core Problem with Naive RAG Systems

Most RAG implementations fail in predictable ways:

- The model answers from parametric memory when retrieval fails
- Hallucinations go undetected because answers “sound correct”
- Offline evals look good but don’t reflect production behavior
- There’s no explanation for *why* a bad answer was returned

The key insight I learned early:

> **RAG quality is a systems problem, not a model problem.**

Fixing prompts alone does not solve reliability.

---

## 3. System Architecture (Mental Model)

I designed the system as two clearly separated paths sharing the same RAG core.

### Online (Production)
- Prioritizes safety, latency, and interpretability
- Uses tracing, heuristics, and a gated LLM judge
- Produces explicit decisions: PASS / RETRY / REFUSE

### Offline (Evaluation)
- Prioritizes learning and improvement
- Uses a golden dataset and Ragas metrics
- Evaluates retrieval and generation quality separately

Both paths share:
- the same ingestion
- the same chunking
- the same embeddings
- the same retrieval logic

This ensures evals reflect real system behavior.

---

## 4. Observability: Making the System Explain Itself

### 4.1 Step-by-Step Tracing

Every request is traced end-to-end using LangSmith:
- input normalization
- retrieval with relevance scores
- context formatting
- LLM generation
- judge invocation (when triggered via flags)

This makes failures **inspectable**, not mysterious.

> If I can’t explain why the system answered something, I don’t trust it in production.

---

### 4.2 Metadata as Product Telemetry

Each run includes structured metadata:
- user_id
- document_id
- retrieval_k
- pipeline version
- retriever version
- prompt version

This enables:
- debugging regressions
- controlled experiments
- rollbacks and audits

This is product telemetry, not just logging.

---

## 5. Online Evaluation: Real-Time Quality Guardrails

### 5.1 Heuristic Flags (Cheap, Deterministic)

Before using an LLM judge, the system computes fast heuristic flags:

- `retrieval_empty`
- `low_relevance`
- `conflict_suspected`
- `answer_too_short`
- `sensitive_topic`

These flags are:
- cheap
- interpretable
- deterministic

They catch most failures without extra LLM calls.

---

### 5.2 LLM Judge (Selective and Gated)

An LLM judge is invoked **only if at least one undesirable flag is triggered**.

The judge evaluates:
- groundedness
- completeness
- hallucination risk

And returns a structured decision:
- PASS
- RETRY
- REFUSE
- ESCALATE

This balances:
- cost
- latency
- safety

Not every run deserves a judge.

---

### 5.3 Explicit Decisioning (Not Just Scoring)

Every request ends in a clear outcome:

- **PASS** → return the answer
- **RETRY** → expand retrieval and rerun
- **REFUSE** → return a safe fallback
- **ESCALATE** → future human review

Evaluation without action is noise.
This system always knows **what to do next**.

---

## 6. Offline Evaluation: Measuring True System Quality

### 6.1 Why Offline ≠ Online

Online evals protect users.  
Offline evals help the team learn.

I kept them separate on purpose.

Offline evaluation uses:
- a curated golden dataset
- batch execution
- Ragas metrics

---

### 6.2 Golden Dataset Design

I created a small but high-quality golden set (10–15 questions) that includes:
- in-scope questions
- out-of-scope questions
- edge cases
- sensitive topics

Ground truth answers are written as **rubrics**, not prose:
- key points
- acceptable variations
- required coverage

This avoids penalizing good answers for wording differences.


---

### 6.3 Ragas Metrics Used

Each metric diagnoses a different failure mode:

- **Faithfulness** – did the answer stay grounded in context?
- **Answer Correctness** – is the answer right vs ground truth?
- **Answer Relevancy** – did the answer address the question?
- **Context Precision** – was the retrieved context mostly useful?
- **Context Recall** – did retrieval include all needed info?

Before ground truth rubrics (direct answers) :
<img width="526" height="242" alt="image" src="https://github.com/user-attachments/assets/0f8dfdc8-a5d5-4495-af57-9a61ec96c062" />

After ground truth Rubrics :

<img width="502" height="236" alt="image" src="https://github.com/user-attachments/assets/56535033-63ae-48e5-ac52-6cd38698a772" />

After introducing rubric-based ground truths:
- Answer correctness mean increased from ~0.62 → ~0.83
- Correctness pass rate doubled from 40% → 80%
- Faithfulness improved from ~0.80 → 1.00, indicating fewer ungrounded responses
- Relevancy, precision, and recall remained consistently high

A key insight:
>Low correctness does not always indicate poor model performance, it can signal poorly designed ground truth.
>High correctness with low faithfulness means the model answered correctly **for the wrong reason** — a major production risk.


---

## 7. Aggregate Evaluation: From Scores to Decisions

Per-question scores are for debugging.  
Aggregates are for decisions.

For each metric, I compute:
- mean
- median
- pass rate vs threshold

This produces a compact RAG performance summary that answers:
- Is the system reliable overall?
- Where is it weak?
- Did this change improve or degrade quality?

---

## 8. Key Learnings

Some hard-won insights from this work:

- Correct answers can still be unsafe
- Retrieval quality dominates generation quality
- Faithfulness is more important than correctness in production
- Over-retrieval hurts precision more than it helps recall
- Evaluation without ownership paths does not improve systems

Most importantly:
> **Trust emerges from observability + evaluation + decisioning, not from better prompts.**

---

## 9. Points to keep in mind

This project thought me how I should think as an AI PM:

- Frame GenAI as a system, not a model
- Design metrics that map to decisions
- Balance cost, latency, and safety
- Care deeply about debuggability and trust

---

## 10. What I’d Build Next

If extending this system further, I’d focus on:
- automated regression gates in CI
- alerting on faithfulness drops
- human-in-the-loop escalation flows

---

### Closing Thought

GenAI products don’t fail loudly, they fail quietly.

My goal as an AI PM is to make failures:
- visible,
- explainable,
- and fixable.

This artifact documents one step in that direction.
