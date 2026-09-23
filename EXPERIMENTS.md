# Paper experiment map

The exact paper-to-record index is `provenance/paper/USEFUL_EVIDENCE_MAP.json`. The table below groups scientific questions; it does not count each table or figure as an independent experiment. Each evidence entry retains its original hash and result scope.

| Paper evidence | Code family | Recorded evidence |
| --- | --- | --- |
| Complete FiQA retrieval against Panther/Pisces; accuracy and complete-record delivery; setup | `source/retrieval/complete`, `baselines/panther`, `baselines/pisces`, `experiments/retrieval` | E01–E03 |
| Correction mechanism and transport ablation, output equality, per-stage profiles | `experiments/ablation`, `analysis/ablation`, `analysis/paper` | E04–E07 |
| Identifier-only historical mechanism comparison | Retrieval producer/consumer and paper analysis sources | E08 |
| BERT-base shared-host comparison | `source/bert_base/shared_host_bound`, `experiments/bert_base`, BOLT/BumbleBee adapters | E09 |
| Author-side identifier-only aggregate comparison | `analysis/paper/author_aggregates.json` | E10; aggregate-level record, not invented per-query observations |
| BERT-large shared-host comparison | `source/bert_large`, `baselines/bumblebee/bert_large`, model controllers | E11 |
| GPT-2 shared-host comparison | `source/gpt2/shared_host`, `experiments/gpt2/shared_host`, BumbleBee GPT-2 graph/interface | E12 |
| BERT-base physical WAN | `source/bert_base/wan`, OT/attention support, BERT-base controllers and baseline WAN adapters | E13 |
| BERT-large physical WAN | `source/bert_large`, `experiments/bert_large/wan`, matching BumbleBee interface | E14 |
| GPT-2 physical WAN | `source/gpt2/wan`, `source/gpt2/frontend`, `source/ot`, `experiments/gpt2/wan` | E15 |
| FiQA complete-record physical WAN | Retrieval implementation/adapters plus `experiments/retrieval/wan_transport` | E16 |
| Public arithmetic, equations, quantization and range checks | `analysis/paper`, `tests/arithmetic`, `data/preparation` | E17–E19 |

## Versions and dependencies

The GPT-2 shared-host executable precedes the final WAN implementation. It is deliberately not replaced by the latest decoder when documenting the earlier result. BERT-base also has separate shared-host and WAN source bindings. BERT-large's materialized source closure is preserved along with the model entry and library recipes. For each executable, use its recorded dependency list, rather than selecting headers by their newest modification time.

The reference OT backend is retained for the BERT build that actually used it. The final GPT-2 build selects the batched-wire backend instead. Removing the historical backend would break the BERT source closure; substituting it into the GPT-2 build would discard a qualified optimization. Both are therefore included with separate bindings.

`source/gpt2/cohost` in the recovered source inventory is a historical support/cache layer used by build dependencies; the **shared-host result entry is `source/gpt2/shared_host`**. The final decoder and correct OT backend are selected by the `gpt2_wan` and `ot_backend` build records. Source-map bindings, not a directory's informal name, determine the executable lineage.

## WAN table

All physical-WAN rows use a local client and a cloud server. The endpoint labels describe what is timed, not a reversal of those roles.

| Workload | CoSER (s) | Baseline (s) | Baseline | Endpoint |
| --- | ---: | ---: | --- | --- |
| BERT-base, 128 tokens | 707.652 | 4299.855 | BOLT | Model |
| BERT-base, 128 tokens | 707.652 | 1032.479 | BumbleBee | Model |
| BERT-large, 128 tokens | 1587.724 | 3324.594 | BumbleBee | Model |
| GPT-2, 7 input / 8 output tokens | 1857.467 | 2560.093 | BumbleBee | Full cold start |
| FiQA, complete Top-10 records | 36.998 | 81.040 | Panther | Complete online |
| FiQA, complete Top-10 records | 36.998 | 92.480 | Pisces | Complete online |

Values here are rounded for reading. Use the original JSON for ratios. Model timing begins after embedding LayerNorm and ends at the client classification acknowledgement. Full cold start includes initialization and weight loading. Complete online retrieval excludes reusable corpus-cache construction. The baselines are measured adapter executions, not latency values copied from their papers. Preserve each implementation's arithmetic, graph and deployment conditions, including the historical BOLT reference's earlier implementation context.

## What this archive does not claim

The source collection and statistical checks do not establish a fresh WAN speedup, a clean-machine build of every framework, or a guarantee that new hardware reproduces the original wall time. Author-only aggregates retain their aggregate status. The release does not reconstruct missing per-query records, combine old private shares, or turn component measurements into complete-model results.
