# CoSER

Source and experiment records for **CoSER**, covering secure retrieval, the correction-protocol ablation, BERT-base, BERT-large, GPT-2, and the client–cloud WAN comparisons in the paper.

This repository preserves the implementations used for the reported results. The shared-host and WAN implementations are kept separately where they differ. It includes native C++ code, Python interfaces, baseline adapters, data preparation, recorded build commands, public arithmetic checks, and the measurements used to produce the tables.

## Get the code

Download **[CoSER-source.zip](CoSER-source.zip)** and extract it. The archive contains the full named source tree and the tools described below. The accompanying `SHA256SUMS` identifies the archive. GitHub's automatic “Download ZIP” also contains this source archive, so extract the inner archive before running the commands.

```sh
unzip CoSER-source.zip
cd CoSER
python3 tools/verify.py
python3 tools/summarize_results.py --output output/measurements.json
```

The first command verifies release files and their original source hashes. The second recomputes the archived retrieval and ablation summaries; it does not start a cryptographic inference experiment.

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
| `tools/` | Release verification, public asset acquisition, source materialization, and recipe inspection |

The release uses descriptive file names. `provenance/source-map.json` maps them back to the exact experiment sources and SHA-256 hashes. Experimental identifiers in that manifest are provenance, not additional result claims. Source contents are preserved rather than silently rewriting mathematical code during packaging.

## Data and reproduction

Read **[DATA.md](DATA.md)** for datasets, checkpoint revisions, download commands, selection rules, quantization, and role ownership. Read **[REPRODUCING.md](REPRODUCING.md)** for environments, build order, driver selection, and end-to-end timing. **[EXPERIMENTS.md](EXPERIMENTS.md)** maps the paper's experiment families to the corresponding code and evidence.

The data are public upstream assets, but the recorded private executions used separately held inputs and shares. This release does not distribute private shares, secret keys, credentials, trained-weight binaries, or a copy of the dataset. Reproduction requires downloading the pinned public assets and preparing a fresh execution with fresh randomness.

## Interpretation of the results

The reported timings belong to the recorded implementations, workloads, endpoints, and deployments. Recomputing a table verifies its arithmetic; it does not reproduce a WAN measurement. The WAN experiments use a local client and a cloud server. “Model,” “full cold start,” and “complete online retrieval” identify different service endpoints and are explained in `EXPERIMENTS.md`.

The final GPT-2 WAN observation is **1857.466667696 seconds**, compared with the recorded BumbleBee reference of **2560.092641194 seconds**, giving **1.378271…×**. The exact records and other workloads are retained in `provenance/evidence/`. This is not a claim that every deployment has the same speedup.

## Verification status

The packaging audit checks collected source hashes, recorded non-system compiler dependencies, direct local Python imports, archive contents, and the archived statistical calculations. No private experiment or cloud benchmark was rerun during packaging. Build configuration and execution steps are described in `REPRODUCING.md`.

## Licenses

Third-party notices and licenses remain with their respective source directories. CoSER modifications are not relicensed by this packaging operation. No blanket license is asserted for the combined archive. Dataset and checkpoint terms are those of their upstream providers; download them from the linked sources.
