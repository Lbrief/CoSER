# Local build and correctness verification

The corrected source archive was extracted into an empty Linux directory and built on Ubuntu 24.04 under WSL 2. The compiler was GCC 13.3.0, with CMake 3.28.3, Ninja 1.11.1 and Python 3.12.3. The build used four physical CPU cores, an 8 GiB memory limit and no swap. This was a correctness and packaging check, not a new performance measurement.

## Results

| Check | Result |
| --- | --- |
| Source archive and original source hashes | PASS |
| CPU Features, HEXL and SEAL built from supplied source | PASS |
| EMP, OT backend and required SCI/BERT libraries rebuilt from source | PASS |
| GPT-2 WAN decoder compilation and linking | PASS |
| GPT-2 shared-host decoder compilation and linking | PASS |
| BERT-base WAN entry compilation and linking | PASS |
| BERT-large entry compilation and linking | PASS |
| Native public production-consumer suite, both modes and both role mappings | PASS |
| Independent Python equation checks and compiled public arithmetic harness | PASS |
| Archived retrieval and ablation table calculations | PASS |
| Owned process exit, all 144 public test ports and temporary-directory cleanup | PASS |

No historical object file, executable or native library was reused. After fixing the initial packaging defects, all four native targets were built again from the corrected archive in a second empty directory. The build records list the actual compiler invocations and output hashes.

The two-party public suite checked **11,645,888 reconstructed integer words** across four cases. Each case contained 103,728 attention words and, both before and after attention, 365,568 normalization words, 482,358 field/GELU words and 555,946 CRT-tail words. The same owners and contexts were reused across these operations. All outputs matched the independent integer references, and both parties exited successfully in every case.

The equation companion passed 100 valid cases covering 6,553,600 coordinates and 2,000 additional locator cases. Its separate compiled C++ harness passed 40 cases covering 2,621,440 dense coordinates and 2,621,440 mask coordinates. These checks validate public arithmetic; they are not encrypted full-model or security measurements.

## Packaging defects corrected

- Absolute-path restoration now processes each original path once, including nested original paths.
- Previously reused support objects now have explicit source-build recipes and their complete recorded dependencies.
- Backend and EMP object aliases resolve to newly compiled objects instead of missing historical copies.
- Missing external-framework build metadata and generated-code definitions have been included and bound to source hashes.
- The README now supplies executable environment, build and public-test commands instead of requiring manual reconstruction of the build order.

## Records and scope

`provenance/reproduction/` contains the environment, build result, public output checks, arithmetic output, cleanup receipt and a compact log archive. Temporary source copies, object files, executables, dependency installations and the test Python environment were deleted after collecting these records.

The BERT entries were compiled and linked; the public protocol suite exercised the final GPT-2 production consumers. This verification did not execute trained-model inference, rebuild the external baseline frameworks, or rerun a WAN benchmark. The original experiment records remain separate. New model executions require the assets and preparation described in `DATA.md` and the appropriate framework configuration in `REPRODUCING.md`.
