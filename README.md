## RankGen - Improving Text Generation with Large Ranking Models

[![made-with-python](https://img.shields.io/badge/Made%20with-Python-red.svg)](#python)
[![arxiv](https://img.shields.io/badge/arXiv-2205.09726-b31b1b.svg)](https://arxiv.org/abs/2205.09726)
[![PyPI version rankgen](https://badge.fury.io/py/rankgen.svg)](https://pypi.python.org/pypi/rankgen/) [![License: Apache 2.0](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Downloads](https://pepy.tech/badge/rankgen)](https://pepy.tech/project/rankgen)

This is the official repository for our EMNLP 2022 paper, [RankGen - Improving Text Generation with Large Ranking Models](https://arxiv.org/abs/2205.09726). RankGen is a 1.2 billion encoder model which maps prefixes and generations from any pretrained English language model to a shared vector space. RankGen can be used to rerank multiple full-length samples from an LM, and it can also be incorporated as a scoring function into beam search to significantly improve generation quality (0.85 vs 0.77 [MAUVE](https://arxiv.org/abs/2102.01454), 75% preference according to humans annotators who are English writers). RankGen can also be used like a dense retriever, and achieves state-of-the-art performance on [literary retrieval](https://relic.cs.umass.edu/leaderboard.html).

This repository contains human evaluation data, links to HuggingFace-compatible model checkpoints, and code to integrate RankGen in beam search on HuggingFace models. RankGen is trained by fine-tuning the T5-XL encoder using the [T5X library](https://github.com/google-research/t5x).

### Updates

* (Nov 2022) We have updated our [arXiv version](https://arxiv.org/abs/2205.09726) to show that RankGen beats newer decoding strategies like contrastive search, contrastive decoding and eta sampling!
* (July 2022) RankGen is now a [PyPI package](https://pypi.org/project/rankgen), just run `pip install rankgen` to use it!
* (July 2022) RankGen checkpoints are now available on the HuggingFace Model Hub ([link](https://huggingface.co/kalpeshk2011))!

### Model checkpoints

All RankGen checkpoints are available on the HuggingFace Model Hub - [link](https://huggingface.co/kalpeshk2011)

We recommend using `RankGen-XL-all`.

| Checkpoint        | Size | Hub Model Name                    | HF Hub Link                                                      |
|-------------------|------|-----------------------------------|------------------------------------------------------------------|
| RankGen-base-all  | 0.1B | kalpeshk2011/rankgen-t5-base-all  | [link](https://huggingface.co/kalpeshk2011/rankgen-t5-base-all)  |
| RankGen-large-all | 0.3B | kalpeshk2011/rankgen-t5-large-all | [link](https://huggingface.co/kalpeshk2011/rankgen-t5-large-all) |
| RankGen-XL-all    | 1.2B | kalpeshk2011/rankgen-t5-xl-all    | [link](https://huggingface.co/kalpeshk2011/rankgen-t5-xl-all)    |
| RankGen-XL-PG19   | 1.2B | kalpeshk2011/rankgen-t5-xl-pg19   | [link](https://huggingface.co/kalpeshk2011/rankgen-t5-xl-pg19)   |

*Older versions of the checkpoints*:

RankGen XL checkpoints compatible with `T5XEmbeddingGeneratorLegacy` - [here](https://drive.google.com/drive/folders/1m8ujkAqkBBWYAJISZigz1Lw4tQGbZXaY?usp=sharing)

T5X JAX checkpoints (base, large, XL) - [here](https://github.com/google-research/google-research/tree/master/rankgen)

### Setup

**Requirements** (`pip` will install these dependencies for you)

Python 3.7+, `torch` (CUDA recommended), `transformers`

**Installation**

(from PyPI)

```
python3.7 -m virtualenv rankgen-venv
source rankgen-venv/bin/activate
pip install rankgen
```

(from source)

```
python3.7 -m virtualenv rankgen-venv
source rankgen-venv/bin/activate
git clone https://github.com/martiansideofthemoon/rankgen
cd rankgen
pip install --editable .
```

**Data Download / Test**

Get the data [here](https://drive.google.com/drive/folders/1DRG2ess7fK3apfB-6KoHb_azMuHbsIv4?usp=sharing) and place folder in root directory. Alternatively, use `gdown` as shown below,

```
gdown --folder https://drive.google.com/drive/folders/1DRG2ess7fK3apfB-6KoHb_azMuHbsIv4
```

Run the test script to make sure the RankGen checkpoint has loaded correctly,

```
python -m rankgen.test_rankgen_encoder --model_path kalpeshk2011/rankgen-t5-base-all

### Expected output
0.0009239262409127233
0.0011521980725477804
```

### Using RankGen

Loading RankGen is simple using the HuggingFace APIs, but we suggest using [`RankGenEncoder`](rankgen/rankgen_encoder.py), which is a small wrapper around the HuggingFace APIs for correctly preprocessing data and doing tokenization automatically. Please see [`rankgen/test_rankgen_encoder.py`](rankgen/test_rankgen_encoder.py) for an example of the usage or see below.

```
from rankgen import RankGenEncoder, RankGenGenerator

rankgen_encoder = RankGenEncoder("kalpeshk2011/rankgen-t5-xl-all")
```

**Encoding text to prefix/suffix vectors**

```
prefix_vectors = rankgen_encoder.encode(["This is a prefix sentence."], vectors_type="prefix")
suffix_vectors = rankgen_encoder.encode(["This is a suffix sentence."], vectors_type="suffix")
```

**Generating text**

```
# use a HuggingFace compatible language model
generator = RankGenGenerator(rankgen_encoder=rankgen_encoder, language_model="gpt2-medium")

inputs = ["Whatever might be the nature of the tragedy it would be over with long before this, and those moving black spots away yonder to the west, that he had discerned from the bluff, were undoubtedly the departing raiders. There was nothing left for Keith to do except determine the fate of the unfortunates, and give their bodies decent burial. That any had escaped, or yet lived, was altogether unlikely, unless, perchance, women had been in the party, in which case they would have been borne away prisoners."]

# Baseline nucleus sampling
print(generator.generate_single(inputs, top_p=0.9)[0][0])
# Over-generate and re-rank
print(generator.overgenerate_rerank(inputs, top_p=0.9, num_samples=10)[0][0])
# Beam search
print(generator.beam_search(inputs, top_p=0.9, num_samples=10, beam_size=2)[0][0])
```

### Reproducing experiments in the paper

**Running beam search with RankGen**

The main file is [`rankgen/rankgen_beam_search.py`](rankgen/rankgen_beam_search.py). To execute it,

```
python rankgen/rankgen_beam_search.py \
    --dataset rankgen_data/wiki.jsonl \
    --rankgen_encoder kalpeshk2011/rankgen-t5-xl-all \
    --num_tokens 20 --num_samples 10 --beam_size 2 \
    --output_file outputs_beam/wiki_t5_xl_beam_2_tokens_20_samples_10.jsonl
```

Evaluating using MAUVE (make sure JSONL file has several thousand generations for intuitive MAUVE scores, 7713 in our experiments),

```
python rankgen/score_multi_beam.py --dataset outputs_beam/wiki_t5_xl_beam_2_tokens_10_samples_10.jsonl
```

**Suffix Identification with GPT2**

The main file is [`rankgen/rankgen_beam_search.py`](rankgen/.py). To execute it,

```
mkdir gold-beats-neg-outputs
python rankgen/gpt2_score.py \
  --dataset rankgen_data/hellaswag_val.tsv \
  --model_size xl \
  --metric avg_conditional \
  --num_negatives 3
```

The corresponding data files can be found in the same Google Drive [folder](https://drive.google.com/drive/folders/1DRG2ess7fK3apfB-6KoHb_azMuHbsIv4?usp=sharing).

### Human evaluation data

We conducted our human evaluation on Upwork, hiring English teachers and writers. We performed blind A/B testing between RankGen and nucleus sampling. We also asked our annotators to provide a 1-3 sentence explanation. You can find all the 600 annotations across two files in [`human-eval-data`](human-eval-data). To compute the evaluation scores run,

```
python rankgen/score_ab_text.py
```

### Citation Information
If you use RankGen, please cite it as follows:
```
@inproceedings{rankgen22,
author={Kalpesh Krishna and Yapei Chang and John Wieting and Mohit Iyyer},
booktitle = {Empirical Methods in Natural Language Processing},
Year = "2022",
Title={RankGen: Improving Text Generation with Large Ranking Models},
}
```
