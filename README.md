# CoSER

Source and experiment records for **CoSER**, covering secure retrieval, the correction-protocol ablation, BERT-base, BERT-large, GPT-2, and the client–cloud WAN comparisons in the paper.

This repository preserves the implementations used for the reported results. The shared-host and WAN implementations are kept separately where they differ. It includes native C++ code, Python interfaces, baseline adapters, data preparation, recorded build commands, public arithmetic checks, and the measurements used to produce the tables.

## Get the code

Download **[CoSER-source.zip](CoSER-source.zip)** and **[CoSER-dependencies.zip](CoSER-dependencies.zip)** and extract both into the same directory. Together they contain the full named source tree, pinned upstream dependency sources and the tools described below. The accompanying `SHA256SUMS` identifies the archive. GitHub's automatic “Download ZIP” also contains both archives, so extract both inner archives before running the commands.

```sh
unzip CoSER-source.zip
unzip CoSER-dependencies.zip
cd CoSER
python3 tools/verify.py
python3 tools/summarize_results.py --output output/measurements.json
```

The verification command checks release files and their original source hashes. The summary command recomputes the archived retrieval and ablation tables.

For the executable reproduction route, follow the Ubuntu setup in **[REPRODUCING.md](REPRODUCING.md)**, then run:

```sh
python3 tools/check_environment.py
python3 tools/build_native.py --work "$HOME/coser-build" --jobs 2 --target all
python3 tools/test_native.py --work "$HOME/coser-build" --output "$HOME/coser-build/public"
```

This builds the four CoSER Transformer entries and five retrieval/ablation entries from source, then runs a two-party public correctness suite against the freshly built libraries. Build commands and diagnostics are saved automatically. BOLT, the Bazel-based baselines, and trained-model preparation have separate commands in the guide.

## Choose a reproduction route

| Workload | Guide |
| --- | --- |
| GPT-2, CoSER and BumbleBee | Full checkpoint, seven-token prompt and eight generated tokens |
| BERT-base and BERT-large | MRPC token preparation, exact tensor conversion and local model entries |
| BOLT | Author-provided word-elimination model and its matched reference |
| CoSER, Panther and Pisces retrieval | Complete FiQA corpus, public index preparation and Top-10 record validation |

All command sequences are in [REPRODUCING.md](REPRODUCING.md). Download links,
fixed revisions and tensor/record formats are in [DATA.md](DATA.md). The
[local verification record](LOCAL_REPRODUCTION.md) distinguishes source builds,
data preparation and executed correctness checks.

## What is included

| Directory | Contents |
| --- | --- |
| `source/` | CoSER retrieval, transformer, arithmetic, and OT implementations |
| `baselines/` | The BOLT, BumbleBee, Panther, and Pisces adapters used by the experiments |
| `experiments/` | Shared-host, WAN, and ablation drivers, interfaces, and verification code |
| `data/` | Checkpoint conversion, quantization, data serialization, and pinned asset manifests |
| `analysis/` | Table calculations, public equation checks, and figure scripts |
| `third_party/` | Collected upstream source, including SEAL, HEXL, EMP, and BumbleBee |
| `support/` | Materialized dependency sources retained from the recorded builds |
| `provenance/` | Source hashes, compiler dependency records, build commands, and paper-to-experiment bindings |
| `tools/` | Environment checks, native builds, two-party public checks, asset acquisition and archive verification |

The release uses descriptive file names. `provenance/source-map.json` maps them back to the exact experiment sources and SHA-256 hashes. Experimental identifiers in that manifest are provenance, not additional result claims. Source contents are preserved rather than silently rewriting mathematical code during packaging.

## Data and reproduction

Read **[DATA.md](DATA.md)** for datasets, checkpoint revisions, download commands, selection rules, quantization, and role ownership. Read **[REPRODUCING.md](REPRODUCING.md)** for environments, build order, driver selection, and end-to-end timing. **[EXPERIMENTS.md](EXPERIMENTS.md)** maps the paper's experiment families to the corresponding code and evidence.

The data are public upstream assets, but the recorded private executions used separately held inputs and shares. This release does not distribute private shares, secret keys, credentials, trained-weight binaries, or a copy of the dataset. Reproduction requires downloading the pinned public assets and preparing a fresh execution with fresh randomness.

## Interpretation of the results

The reported timings belong to the recorded implementations, workloads, endpoints, and deployments. Recomputing a table verifies its arithmetic; it does not reproduce a WAN measurement. The WAN experiments use a local client and a cloud server. “Model,” “full cold start,” and “complete online retrieval” identify different service endpoints and are explained in `EXPERIMENTS.md`.

The final GPT-2 WAN observation is **1857.466667696 seconds**, compared with the recorded BumbleBee reference of **2560.092641194 seconds**, giving **1.378271…×**. The exact records and other workloads are retained in `provenance/evidence/`. This is not a claim that every deployment has the same speedup.

## Verification status

The local reproduction record in **[LOCAL_REPRODUCTION.md](LOCAL_REPRODUCTION.md)** lists the native entries built from source and the public checks actually executed. Source hashes, compiler dependencies, archive contents and table calculations are checked separately. This packaging verification does not rerun private experiments or cloud benchmarks.

## Licenses

Third-party notices and licenses remain with their respective source directories. CoSER modifications are not relicensed by this packaging operation. No blanket license is asserted for the combined archive. Dataset and checkpoint terms are those of their upstream providers; download them from the linked sources.
