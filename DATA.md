# Data sources and preparation

## Public assets

| Asset | Source | Revision or integrity pin |
| --- | --- | --- |
| FiQA retrieval corpus and relevance labels | [BEIR](https://github.com/beir-cellar/beir), [FiQA archive](https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/fiqa.zip) | SHA-256 `32c7df99ed21252fdfb2cf3f5673502a8d245ee0c44c4a133570d92ce2b3ad02` |
| Sentence encoder | [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) | `1110a243fdf4706b3f48f1d95db1a4f5529b4d41` |
| BERT-base MRPC checkpoint | [textattack/bert-base-uncased-MRPC](https://huggingface.co/textattack/bert-base-uncased-MRPC) | `d421614df8fbeb22d6826a24d6397809fdc1e3ff` |
| BERT-large MRPC checkpoint | [yoshitomo-matsubara/bert-large-uncased-mrpc](https://huggingface.co/yoshitomo-matsubara/bert-large-uncased-mrpc) | `187657a5c43638d551bb61885b813348b9d91a51` |
| GPT-2 checkpoint | [openai-community/gpt2](https://huggingface.co/openai-community/gpt2) | `607a30d783dfa663caf39e06633721c8d4cfcd7e` |
| MRPC sentence pairs and labels | [GLUE/MRPC](https://huggingface.co/datasets/nyu-mll/glue/viewer/mrpc) | Validation split; exact selected rows and token bindings are in the included MRPC manifests |

To download an asset into a new directory:

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install huggingface_hub
python tools/fetch_data.py list
python tools/fetch_data.py fiqa --output assets/fiqa
python tools/fetch_data.py encoder --output assets/encoder
python tools/fetch_data.py bert-base --output assets/bert-base
python tools/fetch_data.py bert-large --output assets/bert-large
python tools/fetch_data.py gpt2 --output assets/gpt2
```

The download helper refuses an existing destination. It verifies the FiQA archive hash and pins model downloads to revisions. Upstream availability and download licenses still apply. Downloading a repository can retrieve several framework formats; the recorded preparation scripts select the actual format used. The helper was checked without downloading the large assets again during packaging.

## FiQA

The corpus contains 57,638 documents. Documents keep the order in the archive. The encoder input is `(title + " " + text).strip()`, limited to 256 tokens. The recorded encoder uses attention-mask mean pooling followed by L2 normalization, float32 vectors, dimension 384, and batches of 16. Query IDs are derived from the positive dev/test qrels, with the fixed selected IDs retained in `data/manifests/fiqa_queries.json` and the measurement records.

The integer representation is `round_to_nearest_even(96 * (embedding + 1))`, stored as unsigned bytes in `[0,192]`. The centered representation subtracts 96. The native task scores and each baseline's semantics must be preserved; do not replace them with an arbitrary approximate-nearest-neighbor index.

The record serializer retains the document ID, full UTF-8 title, and full UTF-8 text. Each variable record starts with little-endian `<QII>`: integer ID, title byte length, text byte length, followed by both byte strings. `record_offsets.u64le` indexes these records. The fixed transfer representation pads to a width of 17,008 bytes, preserving complete records rather than truncating long documents. The client receives the complete Top-10 records. The fixed WAN query is ID `3067`; the shared-host comparison uses the recorded selected query list.

`experiments/retrieval/fiqa_embeddings.py` and `data/preparation/fiqa_records.py` preserve the actual encoder and serializer. These historical scripts use fixed experiment paths and receipt checks; use the materialized copy and configure asset locations before execution. Do not fabricate an old receipt simply to bypass a check. The public corpus and query text are downloadable; no prior secret shares are needed.

## MRPC and BERT

Keep the validation row order and selected IDs in the MRPC manifests. Use the tokenizer belonging to the pinned checkpoint, sentence-pair encoding, sequence length 128, and the original attention masks. The base model has 12 layers, hidden width 768 and 12 attention heads; the large model has 24 layers, width 1024 and 16 heads. The classifier and complete feed-forward dimensions remain part of the workload.

The preparation sources include the native asset contract, tensor-name conversion, token validation, and integer quantization. BERT-large conversion covers 393 tensors. Its pinned checkpoint and converted parameter hashes are retained in `data/manifests/bert_large_conversion.json`. CoSER's scale choices come from `bert_quantization_contract.json`; use per-tensor scales rather than applying a single guessed scale to all weights and biases. The BumbleBee adapter retains its own runtime arithmetic settings. Do not impose CoSER's rounding rules on the baseline.

## GPT-2

The workload uses the full 12-layer, 12-head GPT-2 model, hidden width 768, vocabulary 50,257, and 124,439,808 parameters. The public prompt is `I enjoy walking with my cute dog` (seven input tokens), followed by eight generated tokens. Prefix lengths are 7–14; the recorded graph does not use a KV cache.

The source chain is checkpoint acquisition → lossless trained-tensor mapping → Q12 tensor conversion → decoder assets and range checks. These steps are in `data/preparation/gpt2_checkpoint.py`, `gpt2_weight_mapping.py`, `gpt2_quantization.py`, and `gpt2_decoder_assets.py`. Quantization uses float64 multiplication by 4096 followed by ties-to-even rounding, with no clipping. The Flax checkpoint SHA-256 is `192e8257ae9e8f796f764630f4a488a6a16d1461762d62b49ef7405df951a283`. Tensor shape, ordering, and weight hashes are recorded in the included contracts.

## Input ownership

The model owner prepares weights; the input owner prepares tokens or queries. Generate shares and keys on their intended hosts for a new run. Never combine the archived parties' shares or reuse a prior private execution's keys, masks, protocol counters, request ID, or completion receipts. Evaluation references are evaluator data, not inputs that should be supplied to both protocol parties.
