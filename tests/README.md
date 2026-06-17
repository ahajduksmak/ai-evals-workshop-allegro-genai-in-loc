# tests/

Unit tests for the code evaluators in `docs/evaluators/`.

---

## Quick start

```bash
# First time only — create venv and install pytest
python3.12 -m venv .venv
.venv/bin/pip install pytest

# Run the full suite
.venv/bin/pytest tests/ -v
# Expected: 57 passed
```

CI runs the same command automatically on every push and pull request
(`.github/workflows/tests.yml`).

---

## Directory layout

```
tests/
├── conftest.py                        # Shared builders, loaders, helpers
├── fixtures/
│   ├── accuracy_subtype_match.json    # Editable business test cases
│   ├── critical_detection_match.json  # Editable business test cases
│   └── has_accuracy_error_match.json  # Editable business test cases
├── test-accuracy_subtype_match.py     # Tests for accuracy_subtype_match
├── test-critical_detection_match.py   # Tests for critical_detection_match
└── test-has_accuracy_error_match.py   # Tests for has_accuracy_error_match
```

---

## Adding a business-level test case (no Python required)

Open the relevant JSON fixture in `tests/fixtures/` and append an entry to the
`"cases"` array. Each fixture has an inline `_README` field that explains the
fields, but here is a condensed reference:

### `accuracy_subtype_match.json`

```json
{
  "name": "short description shown in the test report",
  "id": "WSLQA-XXX",          // optional — dataset row id for context
  "source": "English text",    // optional — for human readability
  "target": "Czech text",      // optional — for human readability
  "predicted_accuracy_subtype": "mistranslation",
  "gold_accuracy_subtype": "mistranslation",
  "expected_score": 1
}
```

Valid `accuracy_subtype` values (from `docs/datasets/Czech`):
`none` · `mistranslation` · `omission` · `addition` · `untranslated`

Use `null` to simulate a missing/blank value. `expected_score` must be `1`
(match) or `0` (mismatch).

### `critical_detection_match.json`

```json
{
  "name": "...",
  "predicted_severity": "critical",
  "gold_severity": "major",
  "expected_score": 0
}
```

This evaluator only cares whether each side is `"critical"` or not — `minor`
and `major` both count as non-critical. Valid values:
`none` · `minor` · `major` · `critical`

### `has_accuracy_error_match.json`

```json
{
  "name": "...",
  "predicted_accuracy_subtype": "none",
  "gold_accuracy_subtype": "omission",
  "expected_score": 0
}
```

This evaluator only cares whether each side is `"none"` (no error) or anything
else (has error). Uses the same `accuracy_subtype` vocabulary as above.

---

## Adding a new evaluator

1. Add `docs/evaluators/<new_evaluator>.py` with a `perform_eval(run, example=None)` function.
2. Create `tests/fixtures/<new_evaluator>.json` — copy any existing fixture as a template.
3. Create `tests/test-<new_evaluator>.py` — copy any existing test file as a template and change the two `load_evaluator(...)` and `load_cases(...)` calls.
4. Run `pytest tests/ -v` to confirm green.

---

## Technical edge cases (Python-level tests)

Each test file contains a `TestNestedOutput`, `TestJsonStringInput`,
`TestMissingKeys`, and `TestExampleNoneFallback` class that cover payload shapes
that cannot be expressed in the JSON fixtures:

| Class | What it tests |
|---|---|
| `TestNestedOutput` | Output wrapped under an `output` or `prediction` key |
| `TestJsonStringInput` | `outputs` delivered as a JSON string instead of a dict |
| `TestMissingKeys` | Neither side has the relevant key |
| `TestExampleNoneFallback` | Gold read from `run["reference_outputs"]` when `example=None` |

---

## `conftest.py` API reference

### Evaluator loading

```python
perform_eval = load_evaluator("accuracy_subtype_match")
# Imports docs/evaluators/accuracy_subtype_match.py and returns its perform_eval.
# Raises FileNotFoundError with an actionable message if the file is missing.
```

### Mockup-data builders

```python
make_prediction(subtype=None, severity=None, wrap=None, as_json=False)
# Returns an outputs dict (what the model predicted).
# wrap="output"     → {"output": {...}}
# wrap="prediction" → {"prediction": {...}}
# as_json=True      → serialises to a JSON string

make_gold(subtype=None, severity=None)
# Returns a reference outputs dict with "gold_" prefixed keys.

make_run(outputs, reference_outputs=None)
# Returns {"outputs": outputs, ...} as the evaluators expect.

make_example(outputs)
# Returns {"outputs": outputs} as the evaluators expect.
```

### Fixture loading

```python
cases = load_cases("accuracy_subtype_match")
# Reads tests/fixtures/accuracy_subtype_match.json and returns the list of cases.

run, example = case_to_run_example(case)
# Converts a JSON fixture case into (run, example) dicts for the evaluator.

label = case_id(case)
# Returns the human-readable label used as the pytest parametrize id.
```

---

## Repo structure note (for agents)

The evaluators live at `docs/evaluators/<name>.py` (not `docs/<name>.py`).
The gold datasets are at `docs/datasets/Czech/` and `docs/datasets/Dutch/`.
If the evaluators move, update `_EVALUATORS_DIR` in `tests/conftest.py`.
`load_evaluator()` will raise a clear error naming the missing path if the
directory is wrong.

`pytest.ini` at the repo root sets `python_files = test-*.py test_*.py` to
enable collection of the hyphenated `test-*.py` filenames.
