# Reproduction workflow

## 1. Verify the release and recover the build layout

Use Python 3.10 or newer for the release tools. Native experiments require Linux x86-64 (the recorded client used WSL Ubuntu), a C++17 compiler with the recorded AES/AVX2/PCLMUL flags, CMake, OpenMP, OpenSSL, GMP, and the pinned HE/MPC dependencies. GPU acceleration is not assumed.

```sh
python3 tools/verify.py
python3 tools/materialize.py --output /absolute/new/path/coser-build
python3 tools/recipes.py
python3 tools/recipes.py gpt2_wan --materialized /absolute/new/path/coser-build
```

`materialize.py` creates a fresh tree and refuses overwrites. It restores the original file names and include/import relationships **only inside that generated build tree**, rebasing `/mnt/c`, `/mnt/d`, `/home`, and `/data` into the chosen output directory. The published files retain concise names. `PATHS.json` identifies the materialized path for each published source.

This step does not compile, contact a server, create shares, start a benchmark, or bypass an admission check. Files recovered from an archive without a recorded original host location remain in the named source tree; the tool does not guess their original location.

## 2. Build the native dependencies and executable

The recorded compiler invocations, object dependencies, archive commands, link commands and source pins are in `provenance/build/` and `provenance/dependencies/`. Inspect the appropriate group with `tools/recipes.py`. Its output is a command listing, not a topologically ordered shell script; do not pipe it directly into a shell.

The build order is:

1. Build CPU Features and Intel HEXL from their supplied sources, then SEAL using the recorded configuration and generated headers. Keep the recorded polynomial degree, coefficient primes and security settings.
2. Build the EMP support and the selected OT backend. The final GPT-2 backend includes the batched Boolean-wire implementation. The obsolete unbatched backend must not replace it.
3. Compile the recorded SCI support objects and archive them into the FloatingPoint, BuildingBlocks, Math, GC, LinearOT and OT libraries.
4. Compile the selected model entry and link against those libraries and the selected backend. The BERT-large library closure and GPT-2 shared-host/WAN entries are distinct build groups.
5. Resolve system-library locations for the new machine. Paths to `/usr/lib` in a historical command are records of the original environment, not a promise that another distribution installs the library in the same place. Retain ABI-compatible dependencies and record new hashes.

For BumbleBee, the supplied upstream source is the recorded `47c5560d069543591f3b0176eb530ede28e33fc3` tree with its Bazel workspace files. Use the repository's installation instructions and the experiment adapter's runtime overlay. The original runtime used CPython 3.12, JAX 0.4.26 and Transformers 4.31 with its recorded native SPU/YACL build. Do not substitute an unrelated current SPU wheel. BOLT, Panther and Pisces also use the included experiment adapters; they are not interchangeable with a fresh checkout of an upstream default example.

The package preserves source and commands, not an already rebuilt container or binary toolchain. Third-party dependency downloads referenced by upstream build files may still be required. A full clean-machine rebuild of every native target has not been performed during this packaging audit.

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
