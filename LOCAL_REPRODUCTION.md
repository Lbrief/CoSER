# Local reproduction checks

The repaired package was checked on Ubuntu 24.04 under WSL 2 with GCC 13.3, CMake 3.28 and Ninja 1.11. Native helpers use system Python 3.12; the model front end uses pinned Python 3.11.13 and corpus encoding uses its separate pinned environment. These checks validate the build, data preparation and correctness routes. They do not add performance measurements to the paper.

## Completed checks

| Route | Executed check | Result |
| --- | --- | --- |
| CoSER Transformer code | Four native entries rebuilt in two empty directories with their source-built dependencies | PASS |
| CoSER production consumers | Both modes and role mappings; attention and shared normalization, field/GELU and CRT-tail owners | 11,645,888 integer outputs exact |
| Retrieval and ablation code | Five executable entries built from source | PASS |
| BumbleBee runtime | Source wheel, Python 3.11 installation and three two-party native MPC cases | 252 integer outputs exact |
| BOLT | Native model and boundary builds; public boundary checks | 98,304 values exact |
| Panther | Source-built client and server; complete FiQA Top-10 record delivery | 10 records exact |
| Pisces | Final WAN-adapter source build, including its seven overlays; complete FiQA Top-10 delivery | 10 records exact |
| CoSER retrieval | Complete FiQA Top-10 delivery and both parties' score checks | 10 records and 115,276 score words exact |
| Model entry environments | Six CoSER/BumbleBee GPT-2 and BERT module-entry checks against the installed native runtime | PASS |
| Data preparation | Pinned GPT-2 and BERT checkpoints, exact conversion/export, MRPC tokens, FiQA corpus encoding/index and BOLT author assets | PASS |
| Public equation companion | Python equations and compiled C++ arithmetic harness | PASS |

GPT-2 preparation checked 148 tensors containing 124,439,808 parameters. BERT-base and BERT-large exports checked all 198 and 390 native tensor files respectively. MRPC preparation checked all 408 token identities. FiQA preparation used 57,638 documents and 1,148 queries with the documented selection and index construction. The BOLT reference was checked separately from encrypted inference.

The public production-consumer suite checked every reconstructed word against independent integer references. The same owners and contexts were reused around attention, including the final CRT-tail integration. The compiled arithmetic companion checked 2,621,440 dense coordinates and the same number of mask coordinates; the Python companion checked 6,553,600 coordinates and 2,000 additional locator cases.

## Repairs included

- Complete support-library source closure, generated-code definitions and build recipes replace dependencies on historical object files.
- Historical paths are restored in one pass without consulting the original experiment directories.
- External builds include the final Pisces adapter and the narrow GCC 13 compatibility changes required by Panther and BumbleBee.
- BumbleBee binds both native Python build variables to the pinned interpreter and checks the generated configuration before installation. The installed extension and three native MPC cases were then checked directly.
- Large public downloads use bounded transfers followed by the pinned complete-file hash check. BERT export follows each model's actual tensor layout.
- Build limits include preparation and nested build jobs. Failures retain diagnostics; an unchanged incomplete build can resume its own cache.

## Evidence and scope

`provenance/reproduction/` contains the original clean-build and protocol records. `provenance/reproduction/extended/` records the external builds, data preparation, complete retrieval checks, frontend checks and final cleanup, with hashes for the retained detailed evidence.

The accepted verification scope is source builds and the correctness checks listed above. Model frontend checks import the actual modules and native runtime; they do not load trained weights. Full trained-model inference was not repeated during this packaging check, and no WAN benchmark was rerun. The paper's original model results remain in `provenance/evidence/`. Complete-model commands and their larger memory requirements are documented in `REPRODUCING.md`.

After preserving the build, output, failure and cleanup records, the two owned temporary Linux build/data trees were deleted. Original research sources, results and archives were retained.
