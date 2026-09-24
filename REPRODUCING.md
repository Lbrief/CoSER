# Reproduction workflow

## 1. Install the build environment

The native build targets **Ubuntu 24.04 x86-64 and GCC 13**, including Ubuntu under WSL 2. Use the system Python 3.12 for build helpers. The model front end has a separate, pinned Python 3.11 environment created by `tools/setup_python.py models`; do not install its dependencies into the system Python. The corpus encoder uses a third environment, created by `tools/setup_python.py retrieval`, because its PyTorch and Transformers versions differ from the model front end. AES, AVX2 and PCLMUL CPU instructions are required. Run inside the Linux filesystem rather than a Windows-mounted directory. Use a path without spaces. No GPU is required.

```sh
sudo apt-get update
sudo apt-get install -y build-essential gcc-13 g++-13 cmake ninja-build \
  pkg-config libssl-dev libgmp-dev m4 patch python3 python3-venv unzip
unzip CoSER-source.zip
unzip CoSER-dependencies.zip
cd CoSER
python3 tools/verify.py
python3 tools/check_environment.py
```

The check stops with a list of missing requirements. The supplied CPU Features, HEXL and SEAL sources are built locally; this route does not download or reuse historical binaries. Allow 12 GiB RAM for the complete source-build workflow with two compiler jobs. External framework source builds also retain large Bazel caches; reserve 50 GiB of free space when building the complete collection, or build and clean one framework at a time. The complete trained-model workloads have larger memory and data requirements described below.

## 2. Build and run public correctness checks

Choose a fresh, absolute work path. This example keeps every generated object, library and restored source under one directory:

```sh
export COSER_REPRO="$HOME/coser-reproduction"
mkdir "$COSER_REPRO"
python3 tools/build_native.py --work "$COSER_REPRO/native" --jobs 2 --target all
python3 tools/test_native.py --work "$COSER_REPRO/native" --output "$COSER_REPRO/native/public"
python3 tools/summarize_results.py --output "$COSER_REPRO/measurements.json"
```

`build_native.py` builds CPU Features, HEXL, SEAL, EMP, the appropriate OT backends and SCI libraries, followed by the selected CoSER entries:

| Target | Executable |
| --- | --- |
| `gpt2_wan` | Final GPT-2 decoder used for the WAN implementation |
| `gpt2_shared_host` | GPT-2 shared-host decoder |
| `bert_base_wan` | BERT-base native model entry |
| `bert_large_entry` | BERT-large native model entry and its separate library closure |
| `retrieval_complete` | Complete FiQA Top-10 retrieval and record-delivery entry |
| `retrieval_a0b0`, `retrieval_a0b1`, `retrieval_a1b0`, `retrieval_a1b1` | Four separate correction-protocol ablation entries |

The default `all` builds all nine. `--target transformers` selects the first four; `--target retrieval` selects the last five. To build one entry, use its target name. Commands, compiler output and failed attempts stay in `logs/`. A successful build produces a target-specific `BUILD_RESULT_<target>.json` and `BUILD_RESULT.json` with executable paths and hashes. Existing successful steps are reusable within an unchanged build tree; use a new work directory after changing source code. Do not copy old experimental object files into the tree.

`test_native.py` compiles the production-consumer fixture against the newly built libraries. It runs two local parties for both protocol modes and both role mappings, using synthetic inputs and fresh randomness. The fixture exercises attention and reuses the same normalization, field, GELU and CRT-tail owners before and after attention. Each output is checked against an independent integer oracle. The final `public/RESULT.json` must say `PASS`; all party exit codes must be zero and all 144 test ports must be available again. These checks execute real native protocols. They do not load trained weights or produce a new model/WAN timing result.

The default test port blocks begin at 22000, 22700, 23400 and 24100. If occupied, choose another base with `--base-port`; the entire range must stay below 32768. Tests refuse an existing output directory. A failed test leaves its logs and terminates its owned processes; retain those logs before creating a new output directory.

To run the paper's public equation checks as well:

```sh
python3 -m venv "$COSER_REPRO/venv"
"$COSER_REPRO/venv/bin/python" -m pip install -r requirements-public.txt
"$COSER_REPRO/venv/bin/python" tools/test_arithmetic.py --work "$COSER_REPRO/native"
```

This verifies the arithmetic companion and compiles its small public C++ harness. It is separate from encrypted model inference.

### How historical paths are restored

The archive uses descriptive names. The builder calls `materialize.py` to restore recorded include/import names inside `COSER_REPRO/native/materialized`. Absolute historical paths are rebased there in one pass, including nested paths. No original experiment directory is read or modified. `PATHS.json` maps published names to generated paths. The source map checks the original bytes before restoration. Recorded build recipes remain available through `tools/recipes.py`; its output is a listing, not a shell script.

### Build the model front end and external frameworks

The model front end needs the pinned Python 3.11 environment. Corpus encoding
uses a separate Python 3.12 environment. The setup helpers install these under
the selected directory without changing the system Python.

```sh
python3 tools/setup_python.py models --work "$COSER_REPRO/model-python"
python3 tools/setup_bazel.py --output "$COSER_REPRO/bazel"
. "$COSER_REPRO/model-python/env/bin/activate"
python tools/build_external.py bumblebee --work "$COSER_REPRO/bumblebee" \
  --bazel "$COSER_REPRO/bazel/bazel-6.5.0" --python "$VIRTUAL_ENV/bin/python" \
  --jobs 2 --timeout 14400
python tools/install_bumblebee.py --build "$COSER_REPRO/bumblebee" \
  --output "$COSER_REPRO/runtime-installation"
python tools/test_bumblebee.py --installation "$COSER_REPRO/runtime-installation" \
  --output "$COSER_REPRO/runtime-public"
python tools/check_model_frontends.py --installation "$COSER_REPRO/runtime-installation" \
  --qualification "$COSER_REPRO/runtime-public" --output "$COSER_REPRO/model-frontends"
```

BumbleBee's newly built wheel supplies the secure front end used by both
CoSER and BumbleBee. The installer checks its source binding and the public
test executes three batched integer products in two native MPC parties. Use
this wheel instead of an unrelated SPU package. The installer and test each
write a `RESULT.json`; proceed only when both say `PASS`.

Keep the model environment activated throughout these commands. The builder
binds both `PYTHON_BIN_PATH` and `PYTHON3_BIN_PATH` to that environment and checks
the generated native Python configuration. A wheel filename alone is not an
ABI check: its Python tag and the interpreter used for the C++ extension must
agree. The installation command verifies that the extension actually imports.

Panther uses Bazel 6.5.0; the final Pisces adapter uses 7.4.1:

```sh
python tools/build_external.py panther --work "$COSER_REPRO/panther" \
  --bazel "$COSER_REPRO/bazel/bazel-6.5.0" --jobs 2 --timeout 7200
python tools/build_external.py pisces-wan --work "$COSER_REPRO/pisces" \
  --bazel "$COSER_REPRO/bazel/bazel-7.4.1" --jobs 2 --timeout 7200
```

Each framework has its own restored source, dependency cache and build result.
The builder records the exact compatibility changes needed by GCC 13. It keeps
hash verification enabled for dependency downloads and bounds nested build
parallelism as well as the top-level build. On a timeout, preserve the failed
attempt and use the same command with `--resume` only for that incomplete,
unchanged source tree. A completed build is never rerun by `--resume`.

The first source build can take substantially longer than a model execution.
`--timeout` is a build limit, not an expected completion time. Failed attempts
and subsequent continuations have separate logs under `attempts/`.

## 3. Prepare the inputs and run a local workload

Choose the workload below. Run its commands in order and stop on an error.
The model commands use the environment activated above. Keep every output
directory fresh; do not substitute archived secret shares or completion files.
If you already have the pinned public checkpoint or dataset, skip its
`fetch_data.py` command and pass that existing directory to the preparation
helper. Preparation verifies the required input hashes and dimensions. Likewise,
reuse a successful preparation directory for subsequent framework checks; the
same corpus, tokenizer and weights do not need to be downloaded or converted
for each framework. Generated protocol outputs still require a fresh directory.
Complete Transformer executions use much more memory than their source builds:
the recorded local CoSER GPT-2 configuration allowed 48 GiB with swap disabled.
Check the available RAM before starting a full model.

### GPT-2

```sh
python tools/fetch_data.py gpt2 --output "$COSER_REPRO/assets/gpt2"
python tools/prepare_gpt2.py --checkpoint "$COSER_REPRO/assets/gpt2" \
  --output "$COSER_REPRO/inputs/gpt2"
python tools/prepare_gpt2_prompt.py --checkpoint "$COSER_REPRO/assets/gpt2" \
  --output "$COSER_REPRO/inputs/prompt"
python tools/run_gpt2_local.py coser \
  --installation "$COSER_REPRO/runtime-installation" --qualification "$COSER_REPRO/runtime-public" \
  --checkpoint "$COSER_REPRO/assets/gpt2" --prompt "$COSER_REPRO/inputs/prompt" \
  --weights "$COSER_REPRO/inputs/gpt2" --build "$COSER_REPRO/native" \
  --output "$COSER_REPRO/runs/gpt2-coser"
python tools/run_gpt2_local.py bumblebee \
  --installation "$COSER_REPRO/runtime-installation" --qualification "$COSER_REPRO/runtime-public" \
  --checkpoint "$COSER_REPRO/assets/gpt2" --prompt "$COSER_REPRO/inputs/prompt" \
  --output "$COSER_REPRO/runs/gpt2-bumblebee"
```

Both commands retain seven input and eight output tokens, all twelve layers and
the full vocabulary. The local correctness outputs are distinct from the
archived WAN measurements. Do not use local elapsed time as a WAN replacement.

### BERT preparation and CoSER entries

```sh
python tools/fetch_data.py bert-base --output "$COSER_REPRO/assets/bert-base"
python tools/fetch_data.py bert-large --output "$COSER_REPRO/assets/bert-large"
python tools/fetch_data.py mrpc --output "$COSER_REPRO/assets/mrpc"
python tools/prepare_mrpc.py \
  --parquet "$COSER_REPRO/assets/mrpc/mrpc/validation-00000-of-00001.parquet" \
  --checkpoint "$COSER_REPRO/assets/bert-base" --output "$COSER_REPRO/inputs/mrpc"
python tools/prepare_bert.py base --checkpoint "$COSER_REPRO/assets/bert-base" \
  --output "$COSER_REPRO/inputs/bert-base"
python tools/prepare_bert.py large --checkpoint "$COSER_REPRO/assets/bert-large" \
  --output "$COSER_REPRO/inputs/bert-large"
python tools/export_bert.py base --weights "$COSER_REPRO/inputs/bert-base" \
  --output "$COSER_REPRO/inputs/bert-base-native"
python tools/export_bert.py large --weights "$COSER_REPRO/inputs/bert-large" \
  --output "$COSER_REPRO/inputs/bert-large-native"
export COSER_BASE_RUNS="$COSER_REPRO/native/materialized/layout/data/coser_wan_remaining_20260918_v1/ours_classifier/runs"
python tools/run_bert_local.py base --build "$COSER_REPRO/native" \
  --weights "$COSER_REPRO/inputs/bert-base" --native-weights "$COSER_REPRO/inputs/bert-base-native" \
  --tokens "$COSER_REPRO/inputs/mrpc" --installation "$COSER_REPRO/runtime-installation" \
  --qualification "$COSER_REPRO/runtime-public" --output "$COSER_BASE_RUNS/local"
python tools/run_bert_local.py large --build "$COSER_REPRO/native" \
  --weights "$COSER_REPRO/inputs/bert-large" --native-weights "$COSER_REPRO/inputs/bert-large-native" \
  --tokens "$COSER_REPRO/inputs/mrpc" --installation "$COSER_REPRO/runtime-installation" \
  --qualification "$COSER_REPRO/runtime-public" --output "$COSER_REPRO/runs/bert-large"
```

The native BERT-base entry retains its recorded relative asset layout inside the
new build tree. Its fresh correctness route uses the supplied secure lookup
graph to produce owner-local prefix shares; the archived BERT-base WAN timing
started after embedding LayerNorm and used a separately recorded prefix
producer. This correctness check does not recreate the prefix's historical
timing or reuse archived shares. Each role loads only its own inputs and keeps
intermediate values secret.

The native BumbleBee BERT entries use the same newly converted float checkpoint
and public token identity, while retaining their own graph and arithmetic:

```sh
python tools/run_bert_baseline.py base --checkpoint "$COSER_REPRO/assets/bert-base" \
  --weights "$COSER_REPRO/inputs/bert-base" --tokens "$COSER_REPRO/inputs/mrpc" \
  --installation "$COSER_REPRO/runtime-installation" --qualification "$COSER_REPRO/runtime-public" \
  --output "$COSER_REPRO/runs/bert-base-bumblebee"
python tools/run_bert_baseline.py large --checkpoint "$COSER_REPRO/assets/bert-large" \
  --weights "$COSER_REPRO/inputs/bert-large" --tokens "$COSER_REPRO/inputs/mrpc" \
  --installation "$COSER_REPRO/runtime-installation" --qualification "$COSER_REPRO/runtime-public" \
  --output "$COSER_REPRO/runs/bert-large-bumblebee"
```

### BOLT author model

The full BOLT model has a substantially larger memory footprint than its
compilation and public arithmetic checks. In the archived complete execution,
the server's peak memory counter was 47.33 GiB and the client's was 0.84 GiB.
These are separate endpoint peaks, not a measured simultaneous local peak.
For a colocated correctness run, reserve at least 56 GiB of available RAM for
the two parties, with additional room for the operating system. The source
build's memory allowance is not sufficient for this model. On WSL, check
Windows physical availability as well as the Linux limit: a large virtual
memory setting does not add physical RAM. Run full models serially.

Download `bolt_word_elimination.zip` from the author's prepared-model folder
linked in `baselines/bolt/native/README.md`. The fixed file is
`https://drive.google.com/file/d/1cqh29kv9Cw-AabcVXoo97WQUF5z7AI5E/view`.
It is distinct from the dense `bolt.zip` bundle and from CoSER's checkpoint
conversion. Keep the author-provided pruning weights and scale files.

```sh
python3 tools/build_bolt.py --work "$COSER_REPRO/native" --jobs 2
python3 tools/test_bolt.py --work "$COSER_REPRO/native" \
  --output "$COSER_REPRO/native/bolt-public"
. "$COSER_REPRO/model-python/env/bin/activate"
python tools/prepare_bolt.py --archive "$HOME/Downloads/bolt_word_elimination.zip" \
  --output "$COSER_REPRO/inputs/bolt"
python tools/bolt_reference.py --assets "$COSER_REPRO/inputs/bolt" \
  --output "$COSER_REPRO/inputs/bolt-reference"
python tools/run_bolt_local.py --build "$COSER_REPRO/native/bolt" \
  --assets "$COSER_REPRO/inputs/bolt" --reference "$COSER_REPRO/inputs/bolt-reference" \
  --output "$COSER_REPRO/runs/bolt"
```

Preparation verifies the 199 model/scale files, the public input and mask, and
the evaluator labels against their recorded hashes. The evaluator runs the
matched integer reference separately. The two native parties then consume
separate server/client directories and compare the final logits with that
reference. This entry begins at the author's prepared post-embedding input,
retains all twelve layers and word elimination, and does not claim a secure
token-embedding prefix or reproduce a physical WAN by using localhost.

### Complete-record retrieval

Use the separate encoder environment for corpus preparation.

```sh
python3 tools/setup_python.py retrieval --work "$COSER_REPRO/encoder-python"
. "$COSER_REPRO/encoder-python/env/bin/activate"
python tools/fetch_data.py fiqa --output "$COSER_REPRO/assets/fiqa"
python tools/fetch_data.py encoder --output "$COSER_REPRO/assets/encoder"
python tools/prepare_fiqa.py --archive "$COSER_REPRO/assets/fiqa/fiqa.zip" \
  --encoder "$COSER_REPRO/assets/encoder" --output "$COSER_REPRO/inputs/fiqa"
python tools/prepare_panther_index.py --inputs "$COSER_REPRO/inputs/fiqa" \
  --output "$COSER_REPRO/inputs/panther-index"
python tools/test_retrieval.py coser --build "$COSER_REPRO/native" \
  --inputs "$COSER_REPRO/inputs/fiqa" --output "$COSER_REPRO/runs/retrieval-coser"
python tools/test_retrieval.py panther --build "$COSER_REPRO/panther" \
  --inputs "$COSER_REPRO/inputs/fiqa" --index "$COSER_REPRO/inputs/panther-index" \
  --output "$COSER_REPRO/runs/retrieval-panther"
python tools/test_retrieval.py pisces --build "$COSER_REPRO/pisces" \
  --inputs "$COSER_REPRO/inputs/fiqa" --output "$COSER_REPRO/runs/retrieval-pisces"
```

The query is the recorded public FiQA query 3067. Validation includes ten full
records, their identifiers and UTF-8 payloads. The helper preserves each
framework's scoring and candidate-selection semantics. It does not silently
replace an approximate baseline with exact search.

Run these framework checks sequentially. For the supplied Panther corpus and
index, allow at least 16 GiB for the pair of native parties, in addition to the
operating system and other applications. Our local check reached 13.24 GiB;
an earlier 8 GiB limit caused an out-of-memory termination. The Pisces pair
reached 4.55 GiB under an 8 GiB limit. These are memory observations from the
documented correctness inputs, not universal bounds for other datasets.
If a container or service has a memory cap, check that cap as well as the
machine's installed RAM before starting the command.

### Recognize a completed run

For GPT-2, require `REPRODUCTION_RESULT.json` and `CLEANUP.json` to report
`PASS`; the CoSER receipt includes all eight generated tokens, 792 native
stages per party and 520 affine calls per field. BERT, BOLT and retrieval
entries retain their final output and role-exit checks in the output directory.
The expected token identities, tensor hashes and record bytes are checked by
the tools rather than inferred from a zero process exit alone.

Keep failed-run logs until the cause has been resolved. A successful public
matrix check verifies its matrix workload; the complete model and retrieval
commands verify their respective workloads separately. `LOCAL_REPRODUCTION.md`
lists the checks actually performed for this release.

## 4. Relate local execution to the recorded deployment

The portable entries use local processes and new randomness. They do not
connect to a cloud host. The preserved controllers under `experiments/` document
the original endpoint, channel layout and supervision. An independent WAN
deployment needs its own client/server addresses, host authentication, separate
owner data, new output directories, unused ports and resource limits. Historical
controllers contain the old deployment configuration and must not be launched
unchanged.

## 5. Measure the intended endpoint

Shared-host and physical-WAN experiments are separate. In the physical WAN, the local machine is the client and the cloud machine is the server. For the complete GPT-2 request, start the primary clock on the server using `CLOCK_MONOTONIC_RAW` before the first owned process and stop it after the final client acknowledgement. Keep cold startup, setup, loading/sharing, first constants, compilation, all eight steps, communication and waiting in the same interval. Do not splice intervals or subtract setup.

The BERT table's model endpoint starts after embedding LayerNorm and ends at the client classification acknowledgement. Retrieval measures complete online Top-10 record delivery; the reusable corpus-cache setup is reported separately. Each baseline comparison must retain the same service endpoint within its row, while preserving the recorded implementation differences.

For the recorded complete GPT-2 WAN configuration, client CPUs were 0–7, server CPUs 16–23, each with 64 GiB and no swap. Full local qualification used 48 GiB; public components used 32 GiB. These are workload configurations, not instructions to change an unrelated machine's settings. Record the actual available topology before choosing affinities.

Keep native progress, per-channel errors, memory, output counts, monotonic-clock evidence and final process/port cleanup. Process existence or SSH keepalive alone is not progress. Do not sum overlapping lanes, nested timers or the two hosts to construct a request wall time.

## 6. Audit the new run

Use the corresponding output/source/clock verifiers from `experiments/` after the new run has reached its terminal state. Compare integers and returned records against the independent references. Save the raw logs, counters, source/weight hashes and timing boundaries before drawing a performance conclusion. A passed archive check or an exact final token sequence is not a proof of full-logit equality, task-wide accuracy, or a new cryptographic security theorem.
