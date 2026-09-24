# Decisions

Format: decision — reason. Append, never edit history.

## 2026-09-24 — v0.1 design calls

1. **McNemar exact test for regression decisions.** Two runs cover the same
   tasks, so the data is paired. Unpaired Wilson throws away the pairing and a
   10-point drop at n=100 produces overlapping intervals — the CI gate would
   never fire. Marginal Wilson intervals still describe each run individually.

2. **DraftManifest / Manifest split, one file on disk.** `import` writes a
   manifest with no instructions and no targets. Under the strict `Manifest`
   schema that file is invalid the moment it is created. `annotate` and
   `import` parse as Draft; `validate`, `run`, `report` parse as strict.

3. **`run` emits thumbnail + full-res crop into results/assets/.** TaskResult
   carries only an image path, so `report` cannot draw overlays without the
   dataset. Emitting per task makes the results dir portable and the image work
   incremental rather than a 300-image batch after a run that might crash.

4. **sizing_target = smallest target box.** Slice keys must be a function of the
   task alone. If the bucket depended on which target the prediction landed
   near, the same task would land in different buckets across runs and
   `compare` would silently compare different slices.

5. **`--max-regression`, not `--fail-under`.** "fail-under" reads as an absolute
   accuracy floor. The flag is a delta against a baseline run.

6. **temperature=0 default, `--allow-nondeterministic` to opt out.** Wilson
   intervals describe variance over tasks, not run-to-run variance from a
   nondeterministic model. Temperature is in config_hash and on the report face.