# Evaluation and Monitoring for GenAI Solutions

A practical guide to testing and monitoring generative AI features so they can be deployed and operated reliably.

---

## 1. Why GenAI needs its own evaluation approach

Traditional software returns the same output for the same input, so unit tests with fixed expected values work. GenAI systems are different:

- **Outputs are non-deterministic.** The same prompt can produce different wording on each run.
- **Quality is multi-dimensional.** An answer can be fluent but wrong, correct but unsafe, or accurate but unhelpful.
- **Failures are silent.** A broken model rarely throws an exception. It produces plausible text.
- **Behaviour drifts.** Model versions, prompts, retrieved context, and user inputs all change over time.

Reliable deployments therefore combine three things: **automated test cases**, **quality metrics**, and **human feedback**, all feeding a **monitoring loop** that runs after release.

---

## 2. The evaluation lifecycle

```
 Design ──► Offline eval ──► Pre-release gate ──► Deploy ──► Online monitoring
   ▲                                                              │
   └────────── Human review, feedback, and new test cases ◄───────┘
```

| Stage | Purpose | Typical output |
|---|---|---|
| Design | Define what "good" means for the use case | Quality rubric, risk list |
| Offline eval | Run a fixed test set against candidate prompts/models | Scores per metric, failure list |
| Pre-release gate | Block release if thresholds are not met | Pass/fail decision |
| Online monitoring | Track live behaviour, errors, and feedback | Dashboards, alerts |
| Feedback loop | Turn real failures into new test cases | Growing regression suite |

---

## 3. Test cases

A test case is the unit of GenAI evaluation. Each one should capture:

- **Input**: the user message, plus any context (retrieved documents, conversation history)
- **Expected behaviour**: not an exact string, but a checkable property (e.g., "mentions the refund window", "does not invent a price", "responds in French")
- **Category**: happy path, edge case, adversarial, out of scope, safety-sensitive
- **Severity**: how bad a failure would be
- **Source**: where it came from (designed, reported by a user, found in production)

### Coverage guidance

| Category | Share of suite (suggested) | Examples |
|---|---|---|
| Core / happy path | 40% | Typical questions the product is built for |
| Edge cases | 20% | Very long input, typos, mixed languages, ambiguous requests |
| Out of scope | 10% | Requests the system should decline or redirect |
| Adversarial | 15% | Prompt injection, attempts to extract system prompts, jailbreaks |
| Safety and compliance | 10% | Personal data, regulated advice, harmful content |
| Regression | 5% + growing | Every past production bug gets a permanent test |

### Writing good test cases

- Prefer **checkable assertions** over exact-match strings.
- Keep each case focused on one behaviour so failures are diagnosable.
- Include **negative cases** that must not happen (e.g., "must not state a price").
- Version the test set alongside the prompts and code.
- Store cases in a plain format such as JSONL so they can be reviewed in pull requests.

---

## 4. Basic quality metrics

No single metric captures quality. Use a small set, and be clear about what each one can and cannot tell you.

### Rule-based metrics (fast, cheap, run on every change)

| Metric | What it checks | Limitation |
|---|---|---|
| Must-include terms | Required facts or keywords appear | Misses paraphrases |
| Must-not-include terms | Forbidden content, banned phrases, leaked secrets | Only catches known patterns |
| Format checks | Valid JSON, length limits, required sections | Says nothing about meaning |
| Refusal correctness | Out-of-scope requests are declined; in-scope ones are not | Needs a refusal detector that is itself tested |
| Latency and cost | Response time, tokens per request | Not a quality measure |
| Error rate | Failed calls, timeouts, empty outputs | Not a quality measure |

### Model-graded metrics (LLM-as-judge)

A second model scores an answer against a rubric. Useful for groundedness, helpfulness, and tone, but:

- Validate the judge against human labels before trusting it. Measure agreement on a sample.
- Use a clear rubric with a fixed scale and written anchors for each score.
- Watch for bias toward longer answers or the judge's own style.
- Pin the judge model version so scores stay comparable over time.

### Retrieval-augmented systems

If the solution uses retrieved context, measure the retrieval and the generation separately:

- **Retrieval**: did the right documents come back? (recall@k, precision@k)
- **Groundedness**: is each claim supported by the retrieved context?
- **Answer relevance**: does the answer address the question?

### Suggested starting thresholds

These are placeholders. Set real values from a baseline measured on your own data.

| Metric | Example release gate |
|---|---|
| Core test pass rate | ≥ 95% |
| Safety and adversarial pass rate | 100% for critical cases, ≥ 98% overall |
| Groundedness (judge score) | ≥ 4.0 / 5 on sampled cases |
| Format validity | ≥ 99% |
| p95 latency | Within the product's SLA |

---

## 5. Human-in-the-loop feedback

Automated metrics miss what users actually care about. Human input comes in three forms:

1. **Expert review (offline).** Subject-matter experts label a sample of outputs against the rubric. Use this to calibrate the automated judges and to build the initial gold set.
2. **Explicit user feedback (online).** Thumbs up/down, a reason dropdown, and an optional comment. Keep the interaction light so people actually use it.
3. **Implicit signals (online).** Regenerations, copy events, edits, escalations to a human agent, abandoned sessions. These are noisy but high volume.

### Making feedback useful

- Log the **full context** with each rating: input, prompt version, model version, retrieved context, and output. Without this, a thumbs-down cannot be reproduced.
- Route negative feedback into a **review queue** with a triage owner.
- Convert confirmed failures into **regression test cases**.
- Track **inter-rater agreement** among reviewers. Low agreement means the rubric is unclear.
- Respect privacy: redact personal data before reviewers see it, and document retention.

---

## 6. Monitoring in production

### What to watch

| Signal | Why it matters | Example alert |
|---|---|---|
| Error and timeout rate | Infrastructure or provider issues | > 2% over 15 minutes |
| Latency (p50, p95) | User experience and cost | p95 above SLA for 10 minutes |
| Token usage and cost per request | Budget and runaway prompts | Daily spend > 120% of forecast |
| Refusal rate | Over-blocking or a prompt regression | Sudden change of more than 2x |
| Format failure rate | Broken downstream parsing | > 1% invalid output |
| Thumbs-down rate | Perceived quality | Rise of more than 50% over baseline |
| Sampled judge scores | Quality drift | Rolling mean below threshold |
| Input distribution shift | New kinds of user requests | New topic cluster above a set share |

### Operational practices

- **Log everything needed to replay a request**, subject to privacy rules: prompt version, model version, parameters, and a request ID.
- **Sample and score** a fixed percentage of live traffic with rule-based checks and the judge.
- **Version everything**: prompts, models, retrieval indexes, and test sets. Tie every production change to a version.
- **Use staged rollouts** (canary or percentage-based) and compare the new version against the control on the same metrics.
- **Keep a rollback path.** Be able to revert a prompt or model version in minutes.
- **Review on a schedule.** A weekly quality review with the feedback queue and metric trends catches slow drift.

---

## 7. Release checklist

- [ ] Quality rubric written and agreed with stakeholders
- [ ] Test suite covers all categories in section 3, and includes past bugs
- [ ] Rule-based metrics pass release thresholds
- [ ] LLM judge validated against a human-labelled sample
- [ ] Adversarial and safety cases reviewed by someone outside the build team
- [ ] Latency and cost measured under realistic load
- [ ] Logging captures replayable context, with privacy controls in place
- [ ] Feedback channel live, with a named owner for the review queue
- [ ] Alerts configured for error rate, latency, cost, and quality signals
- [ ] Rollout plan with a canary stage and a tested rollback

---

## 8. Common pitfalls

- **Evaluating only on examples the team wrote.** Real users will find inputs the team did not imagine. Sample production traffic into the suite.
- **Trusting a single aggregate score.** Break results down by category. An average can hide a failure in safety cases.
- **Changing the judge without re-validating.** Scores then cannot be compared across time.
- **Ignoring cost and latency in quality discussions.** A slightly better answer that doubles latency may not be a better release.
- **Treating feedback as a vanity metric.** Feedback is only valuable when it leads to new tests or fixes.
- **Setting thresholds once and forgetting them.** Revisit them as the product and user base change.

---

## 9. Starter project

The companion file `genai_eval_starter.py` provides a minimal, dependency-free harness that:

- loads test cases from a JSONL file,
- runs them against any model function you supply,
- applies rule-based checks (must-include, must-not-include, format, refusal, length, latency),
- records human feedback to a CSV log,
- prints a summary and writes a JSON report with pass/fail against release thresholds.

Replace the stub model function with a call to your model or application, extend the checks as your rubric matures, and add model-graded scoring once the judge is validated.
