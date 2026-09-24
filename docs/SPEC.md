You are helping me build an open-source project from scratch. Read this spec in
full before writing any code. Then propose the file structure and the schemas,
and STOP for my approval. Do not write implementation code in your first reply.

═══════════════════════════════════════════════════════════════════════
PART 0 — WHO THIS IS FOR
═══════════════════════════════════════════════════════════════════════

Name: click-eval

This is NOT a new benchmark. ScreenSpot, ScreenSpot-v2, ScreenSpot-Pro,
OSWorld-G, UI-Vision and MMBench-GUI already exist and are actively used. Do not
propose adding a bundled benchmark dataset, a leaderboard, or public score
tracking at any point.

This is a measurement tool for ONE question:
  "How often does model X click the right thing in MY app?"

Primary user: an engineer on a small team shipping a computer-use or GUI agent
against a specific set of interfaces — an internal admin console, a supplier
portal, a legacy desktop client. They have 50-300 screens that matter. Public
benchmark scores don't predict their situation: current systems run roughly 80%
on web tasks and roughly 35% on desktop applications, so a leaderboard number
tells them nothing about their own software.

The recurring use case, which matters more than the first-time one:
  Six weeks later the model vendor ships a new version. Did clicking regress on
  MY app? Nobody can answer this today. That is the loop that creates adoption.

Design consequence, applied throughout this spec: `compare` and a CI mode are
first-class features, not day-two extras. Small datasets (n=50-300, slices of
n=5-30) are the normal case, not an edge case, so statistical honesty about
small samples is a core requirement rather than a nicety.

═══════════════════════════════════════════════════════════════════════
PART 1 — SCOPE
═══════════════════════════════════════════════════════════════════════

IN SCOPE for v0.1:
- A dataset format: screenshots plus annotated UI element targets.
- An annotation tool, because a dataset nobody can produce makes everything
  else worthless.
- A runner that evaluates models against a dataset, resumably.
- Scoring with per-slice breakdowns and explicit uncertainty.
- An HTML report whose gallery shows WHY the model missed.
- `compare` between two runs, and a CI mode with a meaningful exit code.
- Two model adapters plus a documented interface for adding more.

OUT OF SCOPE — do not build, do not "leave hooks for":
- Model training or fine-tuning.
- An agent loop (planning, multi-step actions, execution).
- A web app, dashboard, or hosted service.
- Automated UI crawling or screenshot capture.
- A database. Files on disk only.
- Any bundled public benchmark.

If you believe something out of scope is required, say so and stop. Do not add it.

═══════════════════════════════════════════════════════════════════════
PART 2 — STACK
═══════════════════════════════════════════════════════════════════════

- Python 3.11+, managed with uv, src/ layout (so tests run against the installed
  package, not the source tree).
- Typer for the CLI, Pydantic v2 for schemas, Pillow for images, pytest for tests.
- Base install must stay light. `pip install` to a scored report in under 30
  minutes is a hard promise.
- torch and transformers go in [project.optional-dependencies].local and NEVER in
  the base install. They are multiple GB and would break the promise above.
- No other heavyweight dependency without asking me first.

═══════════════════════════════════════════════════════════════════════
PART 3 — DATASET FORMAT
═══════════════════════════════════════════════════════════════════════

  mydataset/
    manifest.json
    images/
      login_screen.png

manifest.json: {version: "0.1", tasks: [...]}

Each task:
  id           stable unique string, pattern ^[A-Za-z0-9._-]+$
  image        path relative to dataset root; reject ".." traversal
  instruction  natural language, as a user would phrase it, min 3 chars
  targets      LIST of bounding boxes {x, y, width, height} in pixels, min 1
  tags         optional list of strings for slicing

MULTI-TARGET IS A LIST, DELIBERATELY. "Click Save" in a UI with a toolbar Save
and a modal Save has two correct answers; marking one a miss makes the SCORER
wrong, not the model. A prediction is a hit if it lands in ANY target.

Guardrail that must appear in the README and in `annotate`'s help text: if an
instruction has two valid answers, the usually-correct fix is to REWRITE THE
INSTRUCTION, not to box both. Multiple targets are for genuinely interchangeable
elements only. Without this rule, dataset quality rots quietly.

Pydantic models use ConfigDict(extra="forbid"). "instructions" instead of
"instruction" must fail loudly, not silently default.

VALIDATION RUNS IN TWO PASSES, because Pydantic cannot see the disk:
  Pass 1 (schema):  shape, types, ranges, unique ids, unknown keys.
  Pass 2 (loader):  each image file exists and opens, and every target fits
                    inside that image's ACTUAL dimensions. A box at x=1900 on a
                    1280px-wide screenshot is the single most common annotation
                    bug and pass 1 will never catch it.

Error messages matter more than the happy path — a first run against a malformed
manifest is how most users meet this tool. Catch Pydantic's ValidationError and
re-emit human-readable output naming the task index, the task id, the field, and
what was wrong. Never show a raw Pydantic traceback. Report ALL problems in one
pass, not just the first.

═══════════════════════════════════════════════════════════════════════
PART 4 — RESULT SCHEMAS AND THE ADAPTER CONTRACT
═══════════════════════════════════════════════════════════════════════

Outcome enum:    HIT | MISS | ERROR
ErrorKind enum:  TIMEOUT | REFUSAL | UNPARSEABLE | TRANSPORT | ADAPTER_EXCEPTION

Prediction:
  exactly one of point (x,y) or box; validator enforces this
  raw            the model's unmodified output, ALWAYS populated, always kept
  latency_ms
  click_point    property; a box converts to its centre HERE, once, not in the
                 scorer

Adapter contract:

  class GroundingModel(ABC):
      name: ClassVar[str]
      version: ClassVar[str]
      @abstractmethod
      def locate(self, image: Image.Image, instruction: str) -> Prediction: ...

`locate` returns a Prediction or RAISES AdapterError(kind, raw, latency_ms). It
must never return a null or sentinel Prediction. This is what structurally
guarantees "the model didn't answer" is never scored as "the model was wrong" —
the guarantee comes from the type system, not from anyone remembering to be
careful. Errors are counted and reported separately from accuracy, always.

Adapters receive the PIL Image, so normalized-to-pixel conversion happens inside
the adapter, which has the dimensions. Never in the scorer.

TaskResult denormalizes image path, instruction, targets, image_size, tags, so
that `report` and `compare` never need the original dataset again.

═══════════════════════════════════════════════════════════════════════
PART 5 — RESUME AND IDENTITY (get this right, it is subtle)
═══════════════════════════════════════════════════════════════════════

Resume keyed on task_id alone is broken: edit a screenshot, rerun, and results
from two different images silently merge under one score.

Hash PER TASK, not per dataset:
  task_hash = sha256(image bytes + instruction + serialized targets)

Resume skips a task only if its own task_hash is unchanged. Adding a 21st task
therefore re-runs exactly one task, not all 21. Whole-dataset hashing would
invalidate every completed result every time the dataset grows, and datasets
growing is the normal case.

Also keep a run-level fingerprint (model_name, adapter_version, config_hash of
prompt template + temperature + endpoint, click_eval_version, and a
manifest_hash) — used for `compare` warnings, never for resume decisions.

Append each TaskResult to results.jsonl as it completes; write results.json at
the end. Appending to a JSON array means a crash mid-write corrupts the very
file you were crash-proofing.

`--force` re-runs everything. `--limit N` caps the run for a cheap smoke test.

═══════════════════════════════════════════════════════════════════════
PART 6 — SCORING AND STATISTICAL HONESTY
═══════════════════════════════════════════════════════════════════════

Primary metric: point-in-box. Hit if click_point falls inside any target.
Use a half-open interval (x <= px < x+w) so edges are unambiguous.

EVERY accuracy figure this tool prints — the headline included — carries:
  - n
  - a 95% Wilson confidence interval
  - an `underpowered` flag when n < 10

This is not optional and it is not slices-only. A user with 20 tasks and 4 tags
has slices of n=5; 3/5 has a 95% interval of roughly 23-88%. But the HEADLINE is
weak too: 60% at n=20 is roughly 39-78%. Printing a confident headline beside
hedged slices moves the misleading number rather than removing it. Wilson
interval via stdlib math, zero new dependencies.

Underpowered slices render greyed in the report.

Also compute:
  - Normalized distance for misses, normalized BY THE IMAGE DIAGONAL. State this
    in the report and the docs; an unstated normalizer makes the number
    uninterpretable.
  - Accuracy by tag.
  - Accuracy by target size bucket, using RELATIVE linear size
    sqrt(box_area / image_area): tiny <2%, small 2-5%, medium 5-12%, large >12%.
    Absolute pixel buckets break the moment someone mixes a Retina and a 1080p
    screenshot. Size slicing is where the real insight lives — "your model
    cannot hit 16px icons" is the finding users act on.
  - Median and p95 latency.
  - Error rate, by ErrorKind, separate from accuracy.

═══════════════════════════════════════════════════════════════════════
PART 7 — COMMANDS
═══════════════════════════════════════════════════════════════════════

click-eval validate ./mydataset
click-eval import   ./screenshots --out ./mydataset    # manifest skeleton
click-eval annotate ./mydataset
click-eval run      --dataset ./mydataset --model <name> --out ./results
click-eval report   ./results
click-eval compare  ./results/a.json ./results/b.json [--fail-under <delta>]

`compare` prints a side-by-side of both runs with intervals, and lists every
task where the two disagree — that per-task disagreement list is the thing a
user actually opens after a model upgrade. If the two runs have different
manifest_hashes, WARN LOUDLY at the top; do not quietly produce a table
comparing different datasets.

`--fail-under` makes compare exit non-zero when accuracy drops by more than the
given delta, so this drops into CI. Do not flag a drop as a regression when the
two confidence intervals overlap substantially; say so explicitly instead. A
regression checker that cries wolf on noise gets deleted in a week.

═══════════════════════════════════════════════════════════════════════
PART 8 — THE REPORT
═══════════════════════════════════════════════════════════════════════

report.html — single self-contained file, no external assets.

Naively base64-inlining 200 full-resolution screenshots produces a file well over
100MB that hangs the browser. Guaranteed, not theoretical. So, per task:
  - an 800px-max-width WebP q80 thumbnail for the grid, loading="lazy", AND
  - a FULL-RESOLUTION crop of roughly a 300px region centred on the target.

The crop is non-negotiable. A 16px icon inside an 800px-wide thumbnail is about
7 pixels; downscaling alone destroys the exact thing the gallery exists for. The
crop is what tells the user why the miss happened. Full-res originals stay on
disk beside the report.

Draw the true target box in one colour and the predicted point in another, at
full resolution, before downscaling.

Layout: headline numbers with n and intervals, then the tag and size-bucket
tables, then the gallery. A user should scroll the gallery and see the pattern
in the failures within about thirty seconds.

═══════════════════════════════════════════════════════════════════════
PART 9 — MODEL ADAPTERS
═══════════════════════════════════════════════════════════════════════

Ship three:
  fake.py    fixed predictions. Lives in the SHIPPED PACKAGE, not in tests —
             it is how a contributor sanity-checks their own adapter, and it is
             what lets scoring be built and tested before any real adapter exists.
  hosted.py  an HTTP VLM, API key from an env var.
  local.py   an open-weights VLM, behind the [local] optional extra.

Before implementing hosted.py or local.py: SEARCH for the current correct way
each model is prompted for coordinate output and how its response is formatted.
Do not reconstruct either from memory. Show me what you find and wait.

A registry maps name -> adapter. CONTRIBUTING.md explains adding one in under
50 lines.

═══════════════════════════════════════════════════════════════════════
PART 10 — REPO QUALITY
═══════════════════════════════════════════════════════════════════════

Adoption by strangers is the only success metric that matters here, so:

- README opens with the USER'S problem in their words, not a description of the
  software. Something in the shape of: "Your agent clicks the wrong thing 3 times
  in 10 on your own app. You don't know which 3. This tells you." Then a 60-second
  quickstart with real commands and a real screenshot of the report. No feature
  list before the quickstart.
- examples/mini/ — 4-5 self-made screenshots plus manifest, so `click-eval run
  --model fake` works immediately after clone, before anyone annotates anything.
- Type hints throughout. Docstrings on public functions only.
- Tests: schema validation, two-pass loader, scoring, Wilson intervals, resume
  identity, compare, report generation. Mock the adapter — no test hits a real
  model API.
- MIT license.

═══════════════════════════════════════════════════════════════════════
PART 11 — BUILD ORDER
═══════════════════════════════════════════════════════════════════════

Stop after each step and wait for me.

  1. Structure + both schema modules. No implementation.
  2. Dataset layer: schema validation, two-pass loader, human-readable errors,
     `import` skeleton command, per-task hashing. Tests.
  3. Scoring against fake.py: point-in-box, multi-target, Wilson intervals,
     size buckets, slices. Tests. Nothing here touches a real model.
  4. Runner: run loop, JSONL append, resume by task_hash, AdapterError handling.
     Tests.
  5. The annotate Tk app.
  6. hosted.py, then local.py behind the extra.
  7. Report, then compare and CI mode.
  8. README, written last, when you know what the thing actually does.

Scoring before adapters is deliberate: the scorer is the part whose correctness
you are actually claiming. Get it right against a fake model before spending
money on API calls.

At each stop, tell me what you would cut if the project had to ship in two days,
and be direct about anything in this spec you think is wrong.