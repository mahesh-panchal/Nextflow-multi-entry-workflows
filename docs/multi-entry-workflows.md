# Multiple Entry-Point Workflows

## Overview

This pattern structures a pipeline as a set of **independently runnable workflow
files** (`clean_data.nf`, `quant_data.nf`, `main.nf`) rather than a single
monolithic pipeline controlled entirely by configuration flags. Each file is a
self-contained Nextflow workflow that can be launched directly with
`nextflow run`, yet they are composable: `main.nf` imports the same
subworkflows that the individual files use, so there is no duplicated logic.

```
clean_data.nf           ← trim & QC reads, write index samplesheet
quant_data.nf           ← quantify reads with Salmon
main.nf                 ← end-to-end (clean → quantify)

subworkflows/local/
  clean_data.nf         ← shared CLEAN_DATA subworkflow
  quant_data.nf         ← shared QUANT_DATA subworkflow
```

The critical glue between stages is an **index samplesheet**: `clean_data.nf`
writes a CSV containing the published paths of every trimmed read file.
That CSV is the only artifact needed to resume from any downstream entry point.

```bash
# Run stage 1 — outputs results/index/samplesheet.csv
nextflow run clean_data.nf -profile docker --input raw.csv --outdir results/

# Run stage 2 later, or on a different machine, pointing at stage 1 output
nextflow run quant_data.nf -profile docker \
    --input            results/index/samplesheet.csv \
    --salmon_index     /ref/salmon \
    --gtf              /ref/genome.gtf \
    --transcript_fasta /ref/transcriptome.fa \
    --outdir           results/
```

---

## How the pattern is implemented here

### Shared subworkflows

`subworkflows/local/clean_data.nf` and `subworkflows/local/quant_data.nf`
contain the actual process orchestration. Both the standalone entry-point
files and `main.nf` import these, so logic is never repeated.

### Named workflows for testability

Each entry-point file exposes a **named workflow** (`CLEAN_DATA_WF`,
`QUANT_DATA_WF`, `MAIN`) in addition to the anonymous `workflow {}` entry
point. nf-test requires a named target to `include`; the anonymous workflow
handles the production concerns (reading `params`, publishing outputs) while
the named workflow receives its inputs as channels, making it fully
injectable from a test `setup {}` block.

### The index samplesheet as a handoff contract

`clean_data.nf` contains a `WRITE_SAMPLESHEET` process that collects
trimmed-read paths into a CSV validated by the same `schema_input.json`
used by `quant_data.nf`. This means:

- The handoff is **typed** — nf-schema validates the CSV on both sides.
- The CSV lives in `outdir`, making it a normal published artifact that can
  be archived, transferred, or handed to a collaborator.
- Re-running a downstream stage is as simple as passing `--input` to the
  new entry point; no special resume flags or cache sharing is needed.

---

## Pros

### 1. Run only the section you need

A common real-world constraint is needing to re-run just part of a pipeline —
perhaps trimming parameters changed, or a new reference genome is available,
or only a subset of samples needs quantifying. With a monolithic pipeline you
typically work around this with `--skip_*` flags, which add combinatorial
complexity to every process condition. Here you simply call the right entry
point with the right `--input`.

### 2. Storage checkpointing for large cohorts

Nextflow's work directory grows proportionally to the number of intermediate
files. For large cohorts (hundreds to thousands of samples), the combined
scratch footprint of raw reads, trimmed reads, indices, and quantification
outputs can exceed available storage. The multi-entry pattern lets you:

1. Run `clean_data.nf` → archive or delete raw reads once trimmed copies are
   published.
2. Run `quant_data.nf` → archive or delete trimmed reads once quant outputs
   are published.
3. Clean the Nextflow work directory between stages without losing
   resumability for future stages (because the published CSVs are the
   checkpoints, not the work cache).

This is not possible with `-resume` alone, which only caches within a single
run's work directory.

### 3. Simpler `nextflow.config` and fewer flags

A monolithic pipeline that is selectively run via flags must expose every
`skip_*`, `run_*`, and conditional parameter in its schema. Here, the config
stays focused: each workflow file only declares the params it needs. The
cognitive surface for operators and users is smaller.

### 4. Composability is preserved

`main.nf` proves that composition is not sacrificed. Because the logic lives
in shared subworkflows, the end-to-end pipeline is only a thin wrapper — no
logic is duplicated. Teams that always run the full pipeline lose nothing; the
separate entry points are additive.

### 5. Cleaner nf-test isolation

Each workflow file has its own test file with its own `setup {}` block.
Tests for `quant_data.nf` build a Salmon index once and inject it; they have
no knowledge of the trimming stage. This leads to faster tests, smaller blast
radii when a stage changes, and clearer failure messages.

## Cons and trade-offs

### 1. No automatic cross-stage `-resume`

Nextflow's task-level cache (`work/`) is keyed on inputs + process code. When
you run `quant_data.nf` after `clean_data.nf`, Nextflow starts a new run with
a new session ID. Any tasks that were already cached in the `clean_data.nf`
run are **not reused** unless you explicitly pass `-resume` with the correct
`-work-dir` pointing at the prior run's cache.

For the common case — running stages sequentially without re-running the
previous stage — this is irrelevant. But if you restructure and want to
re-run the full pipeline as `main.nf` after already running the individual
stages, you will not benefit from the cached intermediate work.

### 2. The index samplesheet is a manual handoff

The CSV written by `WRITE_SAMPLESHEET` contains **absolute paths** to
published output files. If the `outdir` moves (e.g. you transfer results to
a different filesystem or object store), the paths in the CSV become stale and
`quant_data.nf` will fail its `checkIfExists` validation.

### 5. Parameter namespacing across entry points

Each entry-point file has its own `params` block in `nextflow.config` (or
inline defaults). If a parameter name is reused with different semantics
across files — or if a param needed by `quant_data.nf` is not present in its
section — users will get confusing validation errors. Discipline in parameter
naming and schema design is required.

### 6. Harder to enforce global invariants

A monolithic pipeline can check global conditions (e.g. "if genome X is
selected, ensure index Y exists") in a single validation block before any
process runs. With separate entry points, each file can only validate its own
inputs. Cross-cutting concerns must either be duplicated or documented as
user responsibility.

## Relationship to Nextflow's built-in `-resume`

This pattern is **complementary** to `-resume`, not a replacement. Within a
single stage run, `-resume` still works exactly as expected — re-running
`clean_data.nf` with the same inputs will skip already-completed tasks.
What the multi-entry pattern adds is the ability to **cross session
boundaries** by persisting published outputs as typed checkpoints, giving you
resume semantics that survive work-directory cleanup, machine boundaries, and
long gaps between stages.

