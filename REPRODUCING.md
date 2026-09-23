# Reproduction workflow

## 1. Install the build environment

The executable build route below targets **Ubuntu 24.04 x86-64, GCC 13 and Python 3.12**, including Ubuntu under WSL 2. AES, AVX2 and PCLMUL CPU instructions are required. Run inside the Linux filesystem rather than a Windows-mounted directory. Use a path without spaces. No GPU is required.

```sh
sudo apt-get update
sudo apt-get install -y build-essential gcc-13 g++-13 cmake ninja-build \
  pkg-config libssl-dev libgmp-dev python3 python3-venv unzip
unzip CoSER-source.zip
cd CoSER
python3 tools/verify.py
python3 tools/check_environment.py
```

The check stops with a list of missing requirements. The supplied CPU Features, HEXL and SEAL sources are built locally; this route does not download or reuse historical binaries. Allow 8 GiB RAM and at least 4 GiB free space for the source and temporary builds. The complete trained-model workloads have larger memory and data requirements described below.

## 2. Build and run public correctness checks

Choose a fresh, absolute work path. This example keeps every generated object, library and restored source under one directory:

```sh
export COSER_WORK="$HOME/coser-build"
python3 tools/build_native.py --work "$COSER_WORK" --jobs 2 --target all
python3 tools/test_native.py --work "$COSER_WORK" --output "$COSER_WORK/public"
python3 tools/summarize_results.py --output "$COSER_WORK/measurements.json"
```

`build_native.py` builds CPU Features, HEXL, SEAL, EMP, the appropriate OT backends and SCI libraries, followed by these four recorded CoSER entries:

| Target | Executable |
| --- | --- |
| `gpt2_wan` | Final GPT-2 decoder used for the WAN implementation |
| `gpt2_shared_host` | GPT-2 shared-host decoder |
| `bert_base_wan` | BERT-base native model entry |
| `bert_large_entry` | BERT-large native model entry and its separate library closure |

The default `all` builds all four. To build one entry, use its target name. Commands, compiler output and failed attempts stay in `logs/`. A successful build produces `BUILD_RESULT.json` with the executable paths and hashes. Existing successful steps are reusable within an unchanged build tree; use a new work directory after changing source code. Do not copy old experimental object files into the tree.

`test_native.py` compiles the production-consumer fixture against the newly built libraries. It runs two local parties for both protocol modes and both role mappings, using synthetic inputs and fresh randomness. The fixture exercises attention and reuses the same normalization, field, GELU and CRT-tail owners before and after attention. Each output is checked against an independent integer oracle. The final `public/RESULT.json` must say `PASS`; all party exit codes must be zero and all 144 test ports must be available again. These checks execute real native protocols. They do not load trained weights or produce a new model/WAN timing result.

The default test port blocks begin at 22000, 22700, 23400 and 24100. If occupied, choose another base with `--base-port`; the entire range must stay below 32768. Tests refuse an existing output directory. A failed test leaves its logs and terminates its owned processes; retain those logs before creating a new output directory.

To run the paper's public equation checks as well:

```sh
python3 -m venv "$COSER_WORK/venv"
"$COSER_WORK/venv/bin/python" -m pip install -r requirements-public.txt
"$COSER_WORK/venv/bin/python" tools/test_arithmetic.py --work "$COSER_WORK"
```

This verifies the arithmetic companion and compiles its small public C++ harness. It is separate from encrypted model inference.

### How historical paths are restored

The archive uses descriptive names. The builder calls `materialize.py` to restore recorded include/import names inside `COSER_WORK/materialized`. Absolute historical paths are rebased there in one pass, including nested paths. No original experiment directory is read or modified. `PATHS.json` maps published names to generated paths. The source map checks the original bytes before restoration. Recorded build recipes remain available through `tools/recipes.py`; its output is a listing, not a shell script.

### External frameworks

The four targets above are CoSER entries. BOLT, BumbleBee, Panther and Pisces are separate frameworks with their own native builds, Python environments and experiment adapters. Their collected source, Bazel/CMake metadata, dependency pins and licenses are included. Use the selected framework's recorded source tree and adapter from `EXPERIMENTS.md`; a current upstream wheel is not a substitute for a recorded modified native backend. The local build-check results identify exactly which entries were executed. They do not mark external-framework rebuilds as passed.

## 3. Prepare data and configure a new execution

Follow `DATA.md` before preparing native input files. Asset conversion must check the pinned checkpoint, tensor dimensions, tokenization, quantization and output hashes. Historical preparation files have fixed output names and receipts; create new outputs and configure the new paths rather than reusing completed requests.

Select the implementation using `EXPERIMENTS.md` and `provenance/source-map.json`. The latter records the original file path and hash alongside its published name. The actual controller, native entry, prefix/lookup interface, baseline graph, output verifier and source/runtime manifests are part of the corresponding experiment families.

Before executing a historical driver, explicitly configure:

- the new source/build roots, Python environment, model assets and role-owned input paths;
- client and server addresses, an authenticated SSH identity supplied outside the repository, and a freshly verified server host key;
- a new request/output namespace and unused ports, with the original logical channel structure;
- CPU affinity, memory/swap limits, and supervision appropriate to the machine;
- the matching public qualification and output verification records for that new build.

Historical controller source is included for its exact timing, process supervision and protocol orchestration. It is not a preauthorized deployment configuration. Never run an old release/claim/completion controller against the original experiment directories. Do not edit security or correctness checks to make a new deployment appear qualified.

## 4. Validate before measuring

Validate public synthetic cases and the complete native topology first. Check both protocol roles, output oracles, shape boundaries, owner/session binding, counters, source/runtime hashes, memory limits and cleanup. For complete GPT-2, require all eight outputs, all 792 native stages per party and all 520 affine calls per field. Retain negative or slow cases and failed validation records.

The full GPT-2 workload remains 12 layers, 12 heads, 50,257 vocabulary entries, seven input and eight output tokens, prefixes 7–14, Q12 and no KV cache. Do not obtain an apparent speedup by shortening the output or dropping setup, prefix processing, vocabulary projection or acknowledgement.

The public equation/arithmetic companion is in `analysis/paper/`. Its historical `reproduce.py` also compiles a small public arithmetic harness. Run it from a fresh materialized copy with NumPy and SymPy installed; it does not qualify an encrypted model. The release's lightweight table-only entry is:

```sh
python3 tools/summarize_results.py --output output/measurements.json
```

## 5. Measure the intended endpoint

Shared-host and physical-WAN experiments are separate. In the physical WAN, the local machine is the client and the cloud machine is the server. For the complete GPT-2 request, start the primary clock on the server using `CLOCK_MONOTONIC_RAW` before the first owned process and stop it after the final client acknowledgement. Keep cold startup, setup, loading/sharing, first constants, compilation, all eight steps, communication and waiting in the same interval. Do not splice intervals or subtract setup.

The BERT table's model endpoint starts after embedding LayerNorm and ends at the client classification acknowledgement. Retrieval measures complete online Top-10 record delivery; the reusable corpus-cache setup is reported separately. Each baseline comparison must retain the same service endpoint within its row, while preserving the recorded implementation differences.

For the recorded complete GPT-2 WAN configuration, client CPUs were 0–7, server CPUs 16–23, each with 64 GiB and no swap. Full local qualification used 48 GiB; public components used 32 GiB. These are workload configurations, not instructions to change an unrelated machine's settings. Record the actual available topology before choosing affinities.

Keep native progress, per-channel errors, memory, output counts, monotonic-clock evidence and final process/port cleanup. Process existence or SSH keepalive alone is not progress. Do not sum overlapping lanes, nested timers or the two hosts to construct a request wall time.

## 6. Audit the new run

Use the corresponding output/source/clock verifiers from `experiments/` after the new run has reached its terminal state. Compare integers and returned records against the independent references. Save the raw logs, counters, source/weight hashes and timing boundaries before drawing a performance conclusion. A passed archive check or an exact final token sequence is not a proof of full-logit equality, task-wide accuracy, or a new cryptographic security theorem.
