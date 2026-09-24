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
python3 tools/setup_python.py models --work "$HOME/coser-model-env"
. "$HOME/coser-model-env/env/bin/activate"
python tools/fetch_data.py list
python tools/fetch_data.py fiqa --output assets/fiqa
python tools/fetch_data.py encoder --output assets/encoder
python tools/fetch_data.py bert-base --output assets/bert-base
python tools/fetch_data.py bert-large --output assets/bert-large
python tools/fetch_data.py gpt2 --output assets/gpt2
python tools/fetch_data.py mrpc --output assets/mrpc
```

The download helper refuses an existing destination and writes `DOWNLOAD.json` with file sizes and hashes. It checks the FiQA and MRPC archive pins, the GPT-2 checkpoint files, and the BERT-base weight hash. It downloads the recorded Flax format for BERT-base and GPT-2, and safetensors for the sentence encoder and BERT-large. Upstream availability and download licenses still apply.

## FiQA

Use a separate encoder environment so its PyTorch/Transformers versions do not replace the baseline graph's Flax environment:

```sh
python3 tools/setup_python.py retrieval --work "$HOME/coser-encoder-env"
. "$HOME/coser-encoder-env/env/bin/activate"
```

The corpus contains 57,638 documents. Documents keep the order in the archive. The encoder input is `(title + " " + text).strip()`, limited to 256 tokens. The recorded encoder uses attention-mask mean pooling followed by L2 normalization, float32 vectors, dimension 384, and batches of 16. Query IDs are derived from the positive dev/test qrels, with the fixed selected IDs retained in `data/manifests/fiqa_queries.json` and the measurement records.

The integer representation is `round_to_nearest_even(96 * (embedding + 1))`, stored as unsigned bytes in `[0,192]`. The centered representation subtracts 96. The native task scores and each baseline's semantics must be preserved; do not replace them with an arbitrary approximate-nearest-neighbor index.

The record serializer retains the document ID, full UTF-8 title, and full UTF-8 text. Each variable record starts with little-endian `<QII>`: integer ID, title byte length, text byte length, followed by both byte strings. `record_offsets.u64le` indexes these records. The fixed transfer representation pads to a width of 17,008 bytes, preserving complete records rather than truncating long documents. The client receives the complete Top-10 records. The fixed WAN query is ID `3067`; the shared-host comparison uses the recorded selected query list.

The standalone preparation entry is:

```sh
python tools/prepare_fiqa.py --archive assets/fiqa/fiqa.zip \
  --encoder assets/encoder --output inputs/fiqa
```

It computes the complete public corpus and query embeddings, checks the recorded integer corpus and selected query hashes, and writes `server/corpus.u8`, complete records and offsets, per-query files under `client/`, and separate evaluation labels. The helper also verifies the record serializer's byte hashes. Encoding checkpoints can be resumed with `--resume` only when the input and runtime bindings match. A completed conversion cannot be resumed or overwritten. The preserved historical encoder and serializer remain available for source comparison; the standalone entry does not need their historical directories or completion receipts.

## MRPC and BERT

Reactivate the model environment created in the download section. It contains Python 3.11 and the pinned packages for checkpoint conversion and BumbleBee. Keep that existing environment:

```sh
. "$HOME/coser-model-env/env/bin/activate"
```

Keep the validation row order and selected IDs in the MRPC manifests. Use the tokenizer belonging to the pinned checkpoint, sentence-pair encoding, sequence length 128, and the original attention masks. The base model has 12 layers, hidden width 768 and 12 attention heads; the large model has 24 layers, width 1024 and 16 heads. The classifier and complete feed-forward dimensions remain part of the workload.

The preparation sources include the native asset contract, tensor-name conversion, token validation, and integer quantization. BERT-large conversion covers 393 tensors. Its pinned checkpoint and converted parameter hashes are retained in `data/manifests/bert_large_conversion.json`. CoSER's scale choices come from `bert_quantization_contract.json`; use per-tensor scales rather than applying a single guessed scale to all weights and biases. The BumbleBee adapter retains its own runtime arithmetic settings. Do not impose CoSER's rounding rules on the baseline.

```sh
python tools/prepare_bert.py base --checkpoint assets/bert-base --output inputs/bert-base
python tools/prepare_bert.py large --checkpoint assets/bert-large --output inputs/bert-large
python tools/prepare_mrpc.py --parquet assets/mrpc/mrpc/validation-00000-of-00001.parquet \
  --checkpoint assets/bert-base --output inputs/mrpc
```

The weight converter checks all 201 base or 393 large tensors, shapes, per-tensor scales, signed bounds and recorded integer identities. `flax_params.npz` preserves the float layout for the native baseline interface; `model.npz` contains CoSER's integers. The token converter checks all 408 logical token identities against the manifest before writing client inputs. These are public checkpoint/data conversions, not inference runs or new timing results.

## GPT-2

The workload uses the full 12-layer, 12-head GPT-2 model, hidden width 768, vocabulary 50,257, and 124,439,808 parameters. The public prompt is `I enjoy walking with my cute dog` (seven input tokens), followed by eight generated tokens. Prefix lengths are 7–14; the recorded graph does not use a KV cache.

The source chain is checkpoint acquisition → lossless trained-tensor mapping → Q12 tensor conversion → decoder assets and range checks. These steps are in `data/preparation/gpt2_checkpoint.py`, `gpt2_weight_mapping.py`, `gpt2_quantization.py`, and `gpt2_decoder_assets.py`. Quantization uses float64 multiplication by 4096 followed by ties-to-even rounding, with no clipping. The Flax checkpoint SHA-256 is `192e8257ae9e8f796f764630f4a488a6a16d1461762d62b49ef7405df951a283`. Tensor shape, ordering, and weight hashes are recorded in the included contracts.

```sh
python tools/prepare_gpt2.py --checkpoint assets/gpt2 --output inputs/gpt2
python tools/prepare_gpt2_prompt.py --checkpoint assets/gpt2 --output inputs/gpt2-prompt
```

This converts all 148 trained tensors, verifies their individual integer hashes, and emits native weight files, the tensor index and the matching range certificate. It refuses missing tensors, a different checkpoint, out-of-range weights or a mismatched certificate. It does not create either party's secret shares.

## BOLT prepared model

BOLT's word-elimination experiment uses the author's prepared model bundle,
not the CoSER checkpoint conversion. Obtain
[`bolt_word_elimination.zip`](https://drive.google.com/file/d/1cqh29kv9Cw-AabcVXoo97WQUF5z7AI5E/view)
from the [author's model folder](https://drive.google.com/drive/u/1/folders/13bBok39UevQ-6hDWHtBVtLJrYVo5VnsR).
The dense `bolt.zip` is a different workload. The expected bundle is
817,634,523 bytes. `data/manifests/bolt_public_assets.json` binds all 199
model/scale files and the public input, mask and evaluator labels by SHA-256.

```sh
python tools/prepare_bolt.py --archive "$HOME/Downloads/bolt_word_elimination.zip" \
  --output inputs/bolt
```

Preparation verifies and separates the files into `server`, `client` and
`evaluator` directories. The matched reference and local execution commands
are in `REPRODUCING.md`. They retain the author's twelve layers, word
elimination, scales and prepared post-embedding input. Labels are evaluator
data; they are not supplied as server inputs.

## Input ownership

The model owner prepares weights; the input owner prepares tokens or queries. Generate shares and keys on their intended hosts for a new run. Never combine the archived parties' shares or reuse a prior private execution's keys, masks, protocol counters, request ID, or completion receipts. Evaluation references are evaluator data, not inputs that should be supplied to both protocol parties.
