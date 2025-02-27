# Annotated Transformer 2025

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1o1LaeWzI-hL9yqzvEYzLFfyBKvWm8oGi)




This repository contains an updated implementation of the [Annotated Transformer](https://nlp.seas.harvard.edu/annotated-transformer/), designed to serve as an educational resource for understanding the Transformer architecture. It introduces improvements to the original implementation, including support for English-to-Chinese translation, integration of a custom-trained RoBERTa tokenizer, and visualization using Plotly. This project is targeted at students, researchers, and practitioners who are exploring the foundations of Transformer-based models.

In this tutorial, the Transformer architecture is first introduced, followed by positional encoding, embedding, the feed-forward network, attention, the encoder, and the decoder stacks. A full model is created based on the introduced components. Finally, a toy example and a real world example are given, with attention visualization examples appended.


References:
* *[v2022: Austin Huang, Suraj Subramanian, Jonathan Sum, Khalid Almubarak,
   and Stella Biderman](https://nlp.seas.harvard.edu/annotated-transformer/).*
* *[v2018: Sasha Rush](https://nlp.seas.harvard.edu/2018/04/03/attention.html).*


<p align="center">
    <img src="images/paper.png" alt="Transformer" 
width="600"/>
</p>


## Features

1. **English-to-Chinese Translation Task**
    * Includes a preprocessing pipeline for English-Chinese dataset.
    * Implements a sequence-to-sequence Transformer for translation.

2. **Custom RoBERTa Tokenizer**
    * Trains a tokenizer using the `transformers` library.
    * Handles tokenization and detokenization tailored to the English-Chinese dataset.

3. **Enhanced Batching and Data Collation**
    * Dynamic padding for variable-length sequences.
    * Efficient collation and batching for improved training performance.

4. **Optimizer and Learning Rate Scheduler**
    * Use AdamW as the optimizer.
    * Learning rate scheduler: Cosine annealing with warmup

5. **Visualization with Plotly**
    * Replaces `Altair` with `Plotly` for interactive and high-quality attention visualizations.
    * Fixes bugs in attention visualization and improves interpretability.



## Dependencies

This tutorial is tested in:
* python=3.12.7
* CUDA=11.8


Core python packages:
```
pandas==2.2.3
datasets==3.0.1
plotly==5.24.1
torch==2.4.1
transformers==4.45.2

GPUtil==1.4.0
```
> Other versions may also work.

<br>

## Tokenizer

Train tokenizer from an old one

> RoBERTa is a widely used pretrained language model: https://huggingface.co/docs/transformers/en/model_doc/roberta
> <br>We can use RoBERTa tokenizer as the base tokenizer and train it on the custom data so that it can better adapt to the specific domain.

``` python
from transformers import AutoTokenizer
old_tokenizer = AutoTokenizer.from_pretrained("FacebookAI/roberta-base")    # load RoBERTa tokenizer

# Set traindataset
...

# Set train data iterator
def get_training_corpus(batch_size=1000):
    for i in range(0, len(train_dataset), batch_size):
        samples = train_dataset[i : i + batch_size]
        yield samples["english"] + samples["chinese"]

# Train data iterator
training_corpus = get_training_corpus()

# Train tokenizer from an old one with specified vocabulary size
tokenizer = old_tokenizer.train_new_from_iterator(training_corpus, vocab_size=10000)

```

<br>

## Create Transfromer Model

To create a Transformer model:

``` python
device = 0      # set gpu device id
if not torch.cuda.is_available():   # use cpu if cuda is not available
    device = 'cpu'
print(f'device-{torch.device(device)} is used.')

model = create_model(
    src_vocab_size=tokenizer.vocab_size,    # source vocabulary size
    tgt_vocab_size=tokenizer.vocab_size,    # target vocabulary size
    embed_dim=512,                          # embedding dim: d_model
    num_layers=6,                           # number of encoder/decoder layers
    num_heads=8,                            # number of attention heads
    dropout=0.1,                            # dropout ratio
    pre_norm=True,                          # pre-norm or post-norm
    device=device,                          # device to place the model
)

# Tie source and target embedding weights when necessary
model.tie_weights()
```



## Learning Rate Scheduler


Cosine Annealing with Warmup:
> This creates a schedule with a learning rate that decreases following the values of the cosine function between the
initial $lr$ set in the optimizer to $0$, after a warmup period during which it increases linearly between $0$ and the initial $lr$ set in the optimizer.
> <br>Reference code can be found at: https://github.com/huggingface/transformers/blob/v4.45.2/src/transformers/optimization.py#L144

``` python
from torch.optim import Optimizer
from functools import partial

def cosine_schedule_with_warmup_lr_lambda(
    current_step: int, *, num_warmup_steps: int, num_training_steps: int, num_cycles: float = 0.5
):
    if current_step < num_warmup_steps:
        return float(current_step) / float(max(1, num_warmup_steps))
    progress = float(current_step - num_warmup_steps) / float(max(1, num_training_steps - num_warmup_steps))
    return max(0.0, 0.5 * (1.0 + math.cos(math.pi * float(num_cycles) * 2.0 * progress)))


def cosine_schedule_with_warmup(
    optimizer: Optimizer, num_warmup_steps: int, num_training_steps: int, num_cycles: float = 0.5, last_epoch: int = -1
):
    lr_lambda = partial(
        cosine_schedule_with_warmup_lr_lambda,
        num_warmup_steps=num_warmup_steps,
        num_training_steps=num_training_steps,
        num_cycles=num_cycles,
    )
    return LambdaLR(optimizer, lr_lambda, last_epoch)
```



## Train the Model

Train the model by several epochs:

``` python
train_epochs(
    model,
    train_dataloader,
    criterion,              # loss compute
    optimizer,              # optimizer
    lr_scheduler,           # learning rate scheduler
    num_epochs,             # number of epochs
    save_checkpoint=True,   # save model checkpoint or not
)
```

The successful training process should be like:
```
Epoch [1/10]: 100%|██████████| 1382/1382 [10:47<00:00,  2.13it/s, loss=5.5, lr=0.000999] 
Epoch [2/10]: 100%|██████████| 1382/1382 [10:48<00:00,  2.13it/s, loss=4.49, lr=0.00097] 
Epoch [3/10]: 100%|██████████| 1382/1382 [10:47<00:00,  2.13it/s, loss=4.06, lr=0.000883]
Epoch [4/10]: 100%|██████████| 1382/1382 [10:46<00:00,  2.14it/s, loss=3.68, lr=0.00075] 
Epoch [5/10]: 100%|██████████| 1382/1382 [10:45<00:00,  2.14it/s, loss=3.58, lr=0.000587]
...
```





## Results

Given one English sentence input:

```
<s>The Fed apparently could not stomach the sell-off in global financial markets in January and February, which was driven largely by concerns about further tightening.</s>
```

with Chinese target:

```
<s>美联储显然无法消化1月和2月的全球金融市场抛售，而这一抛售潮主要是因为对美联储进一步紧缩的担忧导致的。</s>
```

Using simple greedy decoding, we can see the improvements from epoch 1 to 5:

```
Epoch  1:     <s>美联储在2009年1月1日，而全球金融市场也因此无法获得的代价，因此，美联储对未来的担忧也因此因此因此更深层次地发生。</s></s></s></s></s></s></s></s></s></s></s></s>
Epoch  2:     <s>美联储显然无法在1月和2月全球金融市场上购买全球金融市场，这主要由进一步紧缩的担忧。</s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s>
Epoch  3:     <s>美联储显然不会在1月和2日全球金融市场出售全球金融市场，而这主要是因为担心进一步紧缩。</s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s>
Epoch  4:     <s>美联储显然不会在1月和2月全球金融市场出售，而这一数字主要是因为担心进一步紧缩。</s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s>
Epoch  5:     <s>美联储显然无法在1月和2月和2月的全球金融市场上做出裁决，这主要是因为担心进一步收紧。</s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s></s>
...
```

