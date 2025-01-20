# multi-GPU training/fine-tuning with DDP and FSDP

## what GPU training to do?

### VRAM requirements

VRAM requirements depends on:

- model size?
- accuracy needed?

## model size?

*NOTE* remember these numbers

8 bits ⇒ 1 byte

16 bit number ⇒ 2 bytes

***formula to calculate VRAM?***

### VRAM required = model parameters (e.g. 7b/405b) + gradients + optimizer states + activations

if i am loading a [16 bit] model ⇒ memory required for every trainable parameter:

model parameter [16 bits → 2 bytes]

gradient [16 bits → 2 bytes]

momentum term [32 bits → 4 bytes]

adaptive term [32 bits → 4 bytes]

parameter in 32 bits → [4 bytes]

activations [depends on context and batch size] 

***how to update model parameters?***

use gradient descent method like SGD

but SGD can get stuck in local minima (not stable)

## adam optimizer

so an improvement could be:

1. gradients ⇒ use a historical average
2. amplify small, dampen big gradients (use historical variance)

now for every trainable parameter we need to store:

1. gradient history (to calculate momentum term)
2. gradient variance history (to calculate adaptive term)

## activation size

depends on batch and sequence length

GPUs must fit the model + gradients + optimizer states + at least 1 batch

*NOTE* start with enough GPUs to fit **model + grads + optimizer** and start with 1 batch and then increase the size

## accuracy needed?

if i am loading a ***quantized*** [4 bit] model ⇒ memory required for every trainable parameter:

model parameter [4 bits → 0.5 bytes]

gradient [16 bits → 2 bytes]

momentum term [32 bits → 4 bytes]

adaptive term [32 bits → 4 bytes]

parameter in 32 bits → [4 bytes]

activations [depends on context and batch size] 

so only quantizing weights will not reduce the required memory but using LoRA will 

LoRA ⇒ freezing the weight of original model + training low rank adapters

[1024 * 1024] (1m parameters)⇒ [8 * 1024] (8k parameters)+ [1024 * 8] (8k parameters) (total 16k parameters)

LoRA savings ⇒ **1/64** parameters reduction

if i am loading a ***quantized*** [4 bit] model w/ ***LoRA*** (QLoRA) ⇒ memory required for every trainable parameter:

model parameter [4 bits → 0.5 bytes]

gradient [16 bits → 2 bytes] * 1/64

momentum term [32 bits → 4 bytes] * 1/64

adaptive term [32 bits → 4 bytes] * 1/64

parameter in 32 bits → [4 bytes] * 1/64

activations [depends on context and batch size] 

***accuracy?***

BEST → full fine-tuning

GOOD → LoRA

OK → QLoRA

## GPU setup

- minimize cost?
- maximize training speed?

## GPU setup

1. you can always do ***Model Parallel → stack multiple GPUs with layers distributed (GPUs are idle and waiting) → only 1 GPU working at a time. so bad utilization***

1. now you want more speed and does the model fit on a single GPU?

if yes (single GPU) → distributed data parallel (replicates whole model on multiple GPUs so that you can feed data in parallel)

if no (need multiple GPUs) → fully sharded data parallel

MP → low communication overhead, low utilization for multi-GPU

DDP → modest communication overhead (gradients)

FSDP → high communication overhead (weights in forward pass + gradients gather-scatter in backprop)

## deepspeed from microsoft

extension of pytorch FSDP but does things a bit differently

deepspeed stage 1 → shard optimizers

deepspeed stage 2 → shard gradients, optimizers

deepspeed stage 3 → shard weights, gradients, optimizers [RECOMMENDED using LoRA]

## huggingface accelerate → wrapper for pytorch ddp and fsdp

supports deepspeed as well

```python
accelerate launch train/train_ddp.py
```

gradient_checkpointing

device_map

use_reentrant → optimizes training

accelerate config