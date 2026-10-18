# GenAI Evaluation and Monitoring Starter

A guide and a small, dependency-free Python harness for testing GenAI features before release and tracking their quality afterwards.

## Contents

| File | Purpose |
|---|---|
| `genai_eval_guide.md` | Concepts: test case design, quality metrics, human-in-the-loop feedback, production monitoring, release checklist, common pitfalls |
| `genai_eval_starter.py` | Harness that runs test cases, applies rule-based checks, enforces a release gate, and logs human feedback |
| `test_cases.jsonl` | Sample test cases (created by the `demo` command) |
| `eval_report.json` | Example report from a demo run (created by `run`) |
| `feedback_log.csv` | Human feedback log (created by the `feedback` command) |

## Requirements

- Python 3.9 or newer (uses `list[str]` type hints and `from __future__ import annotations`)
- No third-party packages

## Quick start

Run the demo, which writes sample test cases and evaluates them:

```bash
python3 genai_eval_starter.py demo
```

This prints a summary, writes `eval_report.json`, and exits with status `0` if the release gate passes or `1` if it fails.

**Note:** the built-in `call_model` function is a stub written to answer the sample cases. A passing demo confirms the harness works. It does not show that a real model is good.

## Commands

### Run test cases

```bash
python3 genai_eval_starter.py run --cases test_cases.jsonl --report eval_report.json
```

Options:

- `--repeats N`: run each case N times to observe variability in non-deterministic outputs
- `--min-pass-rate 0.9`: override the overall pass-rate gate
- `--max-p95 3.0`: override the p95 latency gate in seconds

### Record human feedback

```bash
python3 genai_eval_starter.py feedback \
  --request-id 3fa9c2d41e07 \
  --rating down \
  --reason "invented a price" \
  --comment "Pricing was not in the source" \
  --reviewer "jdoe" \
  --prompt-version "v12" \
  --model-version "model-x-2026-10"
```

`--request-id` comes from the `request_id` field in a run report. Feedback is appended to `feedback_log.csv`.

### Summarise feedback

```bash
python3 genai_eval_starter.py feedback-summary --log feedback_log.csv
```

Shows counts by rating, the thumbs-down rate, and the most common reasons for thumbs-down.

## Test case format

Test cases are stored one per line in a JSONL file. Lines starting with `#` and blank lines are ignored.

```json
{"id": "core-001", "input": "What is your refund policy?", "category": "core", "severity": "high", "must_include": ["refund", "30 days"], "must_not_include": ["lifetime"], "notes": "Policy must match source"}
```

| Field | Type | Default | Meaning |
|---|---|---|---|
| `id` | string | required | Unique identifier |
| `input` | string | required | Text sent to the model |
| `category` | string | `core` | `core`, `edge`, `out_of_scope`, `adversarial`, `safety`, or `regression` |
| `severity` | string | `normal` | `low`, `normal`, `high`, or `critical` |
| `must_include` | list of strings | `[]` | Case-insensitive terms that must appear in the output |
| `must_not_include` | list of strings | `[]` | Case-insensitive terms that must not appear |
| `expect_json` | boolean | `false` | Output must parse as JSON |
| `expect_refusal` | boolean or null | `null` | `true` must refuse, `false` must not refuse, `null` is not checked |
| `max_chars` | integer or null | `null` | Maximum output length in characters |
| `notes` | string | `""` | Free-text context, such as the origin of a regression case |

Duplicate IDs cause the loader to exit with an error.

## Release gate

A run passes the gate only if all three conditions hold:

| Gate | Default | Setting |
|---|---|---|
| Overall pass rate | at least 95% | `DEFAULT_THRESHOLDS["pass_rate"]` or `--min-pass-rate` |
| Critical cases pass rate | 100% | `DEFAULT_THRESHOLDS["critical_pass_rate"]` |
| p95 latency | at most 5.0 s | `DEFAULT_THRESHOLDS["max_p95_latency_s"]` or `--max-p95` |

The defaults are placeholders. Set them from a baseline measured on your own data.

## Connecting your model

Replace the stub in `genai_eval_starter.py`:

```python
def call_model(user_input: str) -> str:
    return my_client.generate(prompt=SYSTEM_PROMPT + user_input).text
```

The function receives the test case input and returns the output as a string. Exceptions raised here are recorded as a failed `no_error` check, not as a silent skip.

## Report format

`eval_report.json` contains:

- `run_at`: UTC timestamp of the run
- `cases_file`: absolute path of the test case file
- `summary`: totals, pass rates, p95 latency, per-category counts, thresholds, gate results, and `release_ok`
- `results`: one entry per case run, with `request_id` (use it for feedback), output, latency, and each individual check with its detail

## Extending the harness

- **New rule checks:** add a block to `run_checks` that returns a `CheckResult`.
- **Model-graded scoring:** call a judge model from `run_checks` or a separate step, and validate it against human labels first (see the guide, section 4).
- **Refusal detection:** `REFUSAL_PATTERNS` is a short regex list. Test it against your own refusals and extend it as needed.
- **Storage:** the feedback log is a CSV file. For multiple reviewers or high volume, move it to a database and keep the same fields.

## Limitations

- Rule-based checks catch known patterns. They do not judge meaning, tone, or groundedness.
- Refusal detection is pattern-based and can miss or misread refusals.
- The harness runs sequentially. It has no retries, rate limiting, or concurrency.
- Feedback is stored in plain text. Redact personal data before storing it, as described in the guide.

## Related

See `genai_eval_guide.md` for the concepts behind the harness, including the evaluation lifecycle, coverage guidance, and the monitoring signals to track in production.
