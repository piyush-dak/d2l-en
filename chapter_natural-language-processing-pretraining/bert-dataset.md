# The Dataset for Pretraining BERT

To pretrain the BERT model as implemented in :numref:`sec_bert`,
we need to generate the dataset in the ideal format to facilitate
the two pretraining tasks:
masked language modeling and next sentence prediction.
The original BERT model is pretrained on two huge corpora (see :numref:`subsec_bert_pretraining_tasks`),
making it hard to run for most readers of this book.
To facilitate the demonstration of BERT pretraining,
we use a smaller corpus WikiText-2 :cite:`Merity.Xiong.Bradbury.ea.2016`.

Comparing with the PTB dataset used for pretraining word2vec in :numref:`sec_word2vec_data`,
WikiText-2 i) retains the original punctuation, making it suitable for next sentence prediction; ii) retrains the original case and numbers; iii) is over twice larger.

```{.python .input  n=1}
import collections
import d2l
import mxnet as mx
from mxnet import autograd, gluon, init, np, npx
from mxnet.contrib import text
import os
import random
import time
import zipfile

npx.set_np()
```

In the WikiText-2 dataset,
each line represents a paragraph where
space is inserted between any punctuation and its preceding token.
Paragraphs with at least two sentences are retained.
To split sentences, we only use the period as the delimiter for simplicity.
We leave discussions of more complex sentence splitting techniques in the exercises
at the end of this section.

```{.python .input  n=2}
# Saved in the d2l package for later use
d2l.DATA_HUB['wikitext-2'] = (
    'https://s3.amazonaws.com/research.metamind.io/wikitext/'
    'wikitext-2-v1.zip', '3c914d17d80b1459be871a5039ac23e752a53cbe')

# Saved in the d2l package for later use
def _read_wiki(data_dir):
    file_name = os.path.join(data_dir, 'wiki.train.tokens')
    with open(file_name, 'r') as f:
        lines = f.readlines()
    paragraphs = [line.strip().lower().split(' . ')
                  for line in lines if len(line.split(' . ')) >= 2]
    random.shuffle(paragraphs)
    return paragraphs
```

## Defining Helper Functions for Pretraining Tasks

In the following,
we begin by implementing helper functions for the two BERT pretraining tasks:
next sentence prediction and masked language modeling.
These helper functions will be invoked later
when transforming the raw text corpus
into the the dataset of the ideal format to pretrain BERT.

### Generating the Next Sentence Prediction Task

According to descriptions of :label:`subsec_nsp`,
the `_get_next_sentence` function generates a training example
for the binary classification task.

```{.python .input  n=3}
# Saved in the d2l package for later use
def _get_next_sentence(sentence, next_sentence, paragraphs):
    if random.random() < 0.5:
        is_next = True
    else:
        # paragraphs is a list of lists of lists
        next_sentence = random.choice(random.choice(paragraphs))
        is_next = False
    return sentence, next_sentence, is_next
```

The following function generates training examples for next sentence prediction
from the input `paragraph` by invoking the `_get_next_sentence` function.
Here `paragraph` is a list of sentences, where each sentence is a list of tokens.
The argument `max_len` specifies the maximum length of a BERT input sequence during pretraining.

```{.python .input  n=4}
# Saved in the d2l package for later use
def _get_nsp_data_from_paragraph(paragraph, paragraphs, vocab, max_len):
    nsp_data_from_paragraph = []
    for i in range(len(paragraph) - 1):
        tokens_a, tokens_b, is_next = _get_next_sentence(
            paragraph[i], paragraph[i + 1], paragraphs)
        # Consider 1 '<cls>' token and 2 '<sep>' tokens
        if len(tokens_a) + len(tokens_b) + 3 > max_len:
            continue
        tokens, segments = d2l.get_tokens_and_segments(tokens_a, tokens_b)
        nsp_data_from_paragraph.append((tokens, segments, is_next))
    return nsp_data_from_paragraph
```

### Generating the Masked Language Modeling Task
:label:`subsec_prepare_mlm_data`

It follows the procedure described in :numref:`subsec_mlm`.

```{.python .input  n=5}
# Saved in the d2l package for later use
def _replace_mlm_tokens(tokens, candidate_pred_positions, num_mlm_preds,
                        vocab):
    # Make a new copy of tokens for the input of a masked language model,
    # where the input may contain replaced '<mask>' or random tokens
    mlm_input_tokens = [token for token in tokens]
    pred_positions_and_labels = []
    # Shuffle for getting 15% random tokens for prediction in the masked
    # language model task
    random.shuffle(candidate_pred_positions)
    for mlm_pred_position in candidate_pred_positions:
        if len(pred_positions_and_labels) >= num_mlm_preds:
            break
        masked_token = None
        # 80% of the time: replace the word with the '<mask>' token
        if random.random() < 0.8:
            masked_token = '<mask>'
        else:
            # 10% of the time: keep the word unchanged
            if random.random() < 0.5:
                masked_token = tokens[mlm_pred_position]
            # 10% of the time: replace the word with a random word
            else:
                masked_token = random.randint(0, len(vocab) - 1)
        mlm_input_tokens[mlm_pred_position] = masked_token
        pred_positions_and_labels.append(
            (mlm_pred_position, tokens[mlm_pred_position]))
    return mlm_input_tokens, pred_positions_and_labels
```

...

```{.python .input  n=6}
# Saved in the d2l package for later use
def _get_mlm_data_from_tokens(tokens, vocab):
    candidate_pred_positions = []
    # tokens is a list of strings
    for i, token in enumerate(tokens):
        # Special tokens are not predicted in the masked language model task
        if token in ['<cls>', '<sep>']:
            continue
        candidate_pred_positions.append(i)
    # 15% of random tokens will be predicted in the masked language model task
    num_mlm_preds = max(1, round(len(tokens) * 0.15))
    mlm_input_tokens, pred_positions_and_labels = _replace_mlm_tokens(
        tokens, candidate_pred_positions, num_mlm_preds, vocab)
    pred_positions_and_labels = sorted(pred_positions_and_labels,
                                       key=lambda x: x[0])
    pred_positions = [v[0] for v in pred_positions_and_labels]
    mlm_pred_labels = [v[1] for v in pred_positions_and_labels]
    return vocab[mlm_input_tokens], pred_positions, vocab[mlm_pred_labels]
```

## Transforming the Raw Corpus into the Pretraining Dataset

...

```{.python .input  n=7}
# Saved in the d2l package for later use
def _pad_bert_inputs(instances, max_len, vocab):
    max_num_mlm_preds = round(max_len * 0.15)
    all_token_ids, all_segments, valid_lens,  = [], [], []
    all_pred_positions, all_mlm_weights, all_mlm_labels = [], [], []
    nsp_labels = []
    for (token_ids, pred_positions, mlm_pred_label_ids, segments,
         is_next) in instances:
        all_token_ids.append(np.array(token_ids + [vocab['<pad>']] * (
            max_len - len(token_ids)), dtype='int32'))
        all_segments.append(np.array(segments + [0] * (
            max_len - len(segments)), dtype='int32'))
        valid_lens.append(np.array(len(token_ids)))
        all_pred_positions.append(np.array(pred_positions + [0] * (
            20 - len(pred_positions)), dtype='int32'))
        # Predictions of padded tokens will be filtered out in the loss via
        # multiplication of 0 weights
        all_mlm_weights.append(np.array([1.0] * len(mlm_pred_label_ids) + [
            0.0] * (20 - len(pred_positions)), dtype='float32'))
        all_mlm_labels.append(np.array(mlm_pred_label_ids + [0] * (
            20 - len(mlm_pred_label_ids)), dtype='int32'))
        nsp_labels.append(np.array(is_next))
    return (all_token_ids, all_segments, valid_lens, all_pred_positions,
            all_mlm_weights, all_mlm_labels, nsp_labels)
```

...

```{.python .input  n=8}
# Saved in the d2l package for later use
class _WikiTextDataset(gluon.data.Dataset):
    def __init__(self, paragraphs, max_len=128):
        # Input paragraphs[i] is a list of sentence strings representing a
        # paragraph; while output paragraphs[i] is a list of sentences
        # representing a paragraph, where each sentence is a list of tokens
        paragraphs = [d2l.tokenize(
            paragraph, token='word') for paragraph in paragraphs]
        sentences = [sentence for paragraph in paragraphs
                     for sentence in paragraph]
        self.vocab = d2l.Vocab(sentences, min_freq=5, reserved_tokens=[
            '<pad>', '<mask>', '<cls>', '<sep>'])
        # Get data for the next sentence prediction task
        instances = []
        for paragraph in paragraphs:
            instances.extend(_get_nsp_data_from_paragraph(
                paragraph, paragraphs, self.vocab, max_len))
        # Get data for the masked language model task
        instances = [(_get_mlm_data_from_tokens(tokens, self.vocab)
                      + (segments, is_next))
                     for tokens, segments, is_next in instances]
        # Pad inputs
        (self.all_token_ids, self.all_segments, self.valid_lens,
         self.all_pred_positions, self.all_mlm_weights,
         self.all_mlm_labels, self.nsp_labels) = _pad_bert_inputs(
            instances, max_len, self.vocab)

    def __getitem__(self, idx):
        return (self.all_token_ids[idx], self.all_segments[idx],
                self.valid_lens[idx], self.all_pred_positions[idx],
                self.all_mlm_weights[idx], self.all_mlm_labels[idx],
                self.nsp_labels[idx])

    def __len__(self):
        return len(self.all_token_ids)
```

```{.python .input  n=9}
# Saved in the d2l package for later use
def load_data_wiki(batch_size, max_len):
    num_workers = d2l.get_dataloader_workers()
    data_dir = d2l.download_extract('wikitext-2', 'wikitext-2')
    paragraphs = _read_wiki(data_dir)
    train_set = _WikiTextDataset(paragraphs, max_len)
    train_iter = gluon.data.DataLoader(train_set, batch_size, shuffle=True,
                                       num_workers=num_workers)
    return train_iter, train_set.vocab
```

```{.python .input  n=10}
batch_size, max_len = 512, 64
train_iter, vocab = load_data_wiki(batch_size, max_len)
```

```{.python .input  n=11}
for (tokens_X, segments_X, valid_lens_x, pred_positions_X, mlm_weights_X,
     mlm_Y, nsp_y) in train_iter:
    print(tokens_X.shape, segments_X.shape, valid_lens_x.shape,
          pred_positions_X.shape, mlm_weights_X.shape, mlm_Y.shape,
          nsp_y.shape)
    break
```

```{.json .output n=11}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "(512, 64) (512, 64) (512,) (512, 20) (512, 20) (512, 20) (512,)\n"
 }
]
```

## Summary

## Exercises

1. Try other sentence splitting methods, such as `spaCy` and `nltk.tokenize.sent_tokenize`. For instance, after installing `nltk`, you need to run `import nltk` and `nltk.download('punkt')` first.

## [Discussions](https://discuss.mxnet.io/t/5868)

![](../img/qr_bert-dataset.svg)
