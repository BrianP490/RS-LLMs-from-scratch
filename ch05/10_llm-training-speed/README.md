# PyTorch Performance Tips for Faster LLM Training



Note that the book is written for education purposes, meaning the original code is kept purposefully simple. This is to aid readability and ensure compatibility across different hardware, including CPUs and GPUs. However, you might be curious about some more advanced PyTorch and GPU features to make the LLM training more performant.

This folder contains three code files that demonstrate performance optimizations for the LLM and the training function introduced in Chapter 5:

1. [`00_orig.py`](00_orig.py): The original Chapter 5 code for CPU and single-GPU training.  
   ➤ Run via: `python 00_orig.py`

2. [`01_opt_single_gpu.py`](01_opt_single_gpu.py): An optimized version for single-GPU training.  
   ➤ Run via: `python 01_opt_single_gpu.py`

3. [`02_opt_multi_gpu_ddp.py`](02_opt_multi_gpu_ddp.py): An optimized version for multi-GPU training using Distributed Data Parallel (DDP).  
   ➤ Run via: `torchrun --nproc_per_node=4 02_opt_multi_gpu_ddp.py`  
   (**Note:** To keep the changes minimal compared to `01_opt_single_gpu.py`, this script supports multi-processing only via `torchrun` as shown above. This means multi-GPU support is **not** supported via `python 02_opt_multi_gpu_ddp.py`)

**Note that these modifications take the training speed from 12,525 tokens per second (single A100) to 142,156 tokens per second (single A100) and 419,259 tokens per second (4x A100s).**

I plan to expand on the differences in a more detailed write-up sometime in the future. For now, the easiest way to see what improvements have been added to the code is to open the files in Visual Studio Code and look at the differences via the "Compare Selected" feature.

![VS compare](https://sebastianraschka.com/images/LLMs-from-scratch-images/bonus/llm-training-speed/vs-code-compare.png)

![PyTorch Tips](https://sebastianraschka.com/images/LLMs-from-scratch-images/bonus/pytorch-tips/pytorch-tips.webp?1)


&nbsp;
## Single GPU speed comparisons

As mentioned above, I plan to elaborate more on the changes in the future. For now, this section contains a simple performance overview in terms of tokens/second for each modification. All experiments were run on A100 GPUs.

&nbsp;
### Baseline

Note that `00_orig.py` servers as the baseline and contains no significant modification and uses the code from Chapter 5 as is besides the following:

- 4 times larger context length (which explains the relatively large memory footprint of `00_orig.py` compared to Chapter 5);
- 4-times batch size changes (another contributor to the relatively large memory footprint of `00_orig.py`);
- a larger public domain book to increase the training data size. 

The hyperparameters are not very optimized for minimizing loss and reducing overfitting, and the text generated by the LLM at the very end may not be super sophisticated; however, this shouldn't matter as the main takeaway is the `tok/sec` metric that serves as a speed reference here (higher is better).

```bash
ubuntu@159-13-52-60:~$ python 00_orig.py
PyTorch version: 2.6.0+cu124
Using cuda
CUDA version: 12.4

Ep 1, Step 000000, Train: 9.535, Val: 9.609, Step tok/sec: 7238, Avg tok/sec: 0
Ep 1, Step 000015, Train: 6.201, Val: 6.152, Step tok/sec: 12545, Avg tok/sec: 12545
Ep 1, Step 000030, Train: 5.663, Val: 5.688, Step tok/sec: 12490, Avg tok/sec: 12517
Ep 1, Step 000045, Train: 5.316, Val: 5.362, Step tok/sec: 12541, Avg tok/sec: 12525
Every effort moves you, and's, and I am not be a

...

Ep 15, Step 000735, Train: 0.227, Val: 6.818, Step tok/sec: 11599, Avg tok/sec: 12248
Ep 15, Step 000750, Train: 0.300, Val: 6.895, Step tok/sec: 12530, Avg tok/sec: 12253
Ep 15, Step 000765, Train: 0.150, Val: 6.914, Step tok/sec: 12532, Avg tok/sec: 12259
Every effort moves you like best to think which he held in the room in him, the interest was the night, the realities of the affairs Bulstrode's duty, now!' the fact is another man, conquests

Allocated memory: 2.5069 GB
Reserved memory: 26.2617 GB
```

Note that `01_opt_single_gpu.py` contains all the modifications listed sequentially below. 

The comparison is always based on the average tok/sec and allocated memory after the first epoch from the previous section.

&nbsp;
### 1. Create causal mask on the fly

- Instead of saving the causal mask, this creates the causal mask on the fly to reduce memory usage (here it has minimal effect, but it can add up in long-context size models like Llama 3.2 with 131k-input-tokens support)

Before:
- `Avg tok/sec: 12525`
- `Reserved memory: 26.2617 GB`

After:
- `Avg tok/sec: 12526`
- `Reserved memory: 26.2422 GB`

&nbsp;
### 2. Use  tensor cores

- Uses tensor cores (only works for Ampere GPUs like A100 and newer)

Before:
- `Avg tok/sec: 12526`
- `Reserved memory: 26.2422 GB`

After:
- `Avg tok/sec: 27648`
- `Reserved memory: 26.2422 GB`

&nbsp;
### 3. Fused AdamW optimizer

- Uses the fused kernels for `AdamW` by setting `fused=True`

Before:
- `Avg tok/sec: 27648`
- `Reserved memory: 26.2422 GB`

After:
- `Avg tok/sec: 28399`
- `Reserved memory: 26.2422 GB`

&nbsp;
### 4. Pinned memory in the data loader

- Uses `pin_memory=True` in the data loaders to pre-allocate and re-use GPU memory

Before:
- `Avg tok/sec: 28399`
- `Reserved memory: 26.2422 GB`

After:
- `Avg tok/sec: 28402`
- `Reserved memory: 26.2422 GB`

&nbsp;
### 5. Using bfloat16 precision

- Switches from 32-bit float to 16-bit brain float (bfloat16) precision

Before:
- `Avg tok/sec: 28402`
- `Reserved memory: 26.2422 GB`

After:
- `Avg tok/sec: 45486`
- `Reserved memory: 13.7871 GB`

&nbsp;
### 6. Replacing from-scratch code by PyTorch classes

- Replaces the LayerNorm and GeLU from-scratch implementation by PyTorch's native implementations

Before:
- `Avg tok/sec: 45486`
- `Reserved memory: 13.7871 GB`

After:
- `Avg tok/sec: 55256`
- `Reserved memory: 11.5645 GB`

&nbsp;
### 7. Using FlashAttention

- Uses PyTorch's self-attention function with FlashAttention instead of our from-scratch multi-head attention implementation.


Before:
- `Avg tok/sec: 55256`
- `Reserved memory: 11.5645 GB`

After:
- `Avg tok/sec: 91901`
- `Reserved memory: 5.9004 GB`

&nbsp;
### 8. Using `pytorch.compile`

- Uses `torch.compile(model)`. Note that the first iterations are always slow before it picks up speed. Since the `Avg tok/sec` measurement only includes the first row from the average calculation, we now use the `Step tok/sec` at the end of epoch 1.


Before:
- `Avg tok/sec: 91901`
- `Reserved memory: 5.9004 GB`

After:
- `Step tok/sec: 112046`
- `Reserved memory: 6.1875 GB`

&nbsp;
### 9. Vocabulary padding

- Here, we slightly increase the vocabulary size from 50,257 to 50,304, which is the nearest multiple of 64. This tip was suggested to me by my former colleague Carlos Mocholi, who mentioned that it originally came from Andrej Karpathy (likely from [this post](https://x.com/karpathy/status/1621578354024677377)). Karpathy's recommendation is based on an interaction with the PyTorch team, who gave advice on `torch.compile` as mentioned by [Bertrand Maher](https://www.linkedin.com/feed/update/urn:li:activity:7309569006057795584?commentUrn=urn%3Ali%3Acomment%3A%28activity%3A7309569006057795584%2C7309754284185669632%29&dashCommentUrn=urn%3Ali%3Afsd_comment%3A%287309754284185669632%2Curn%3Ali%3Aactivity%3A7309569006057795584%29). A good resource for this are [NVIDIA's guidelines on tensor shapes](https://docs.nvidia.com/deeplearning/performance/mixed-precision-training/index.html#tensor-core-shape), where batch sizes and linear layer dimensions are commonly chosen as multiples of certain values. Furthermore, the vocab-padding trick was described by NVIDIA's Megatron team a long time ago (see the 2019 [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/abs/1909.08053) paper).

Before:
- `Step tok/sec: 112046`
- `Reserved memory: 6.1875 GB`

After:
- `Step tok/sec: 127345`
- `Reserved memory: 5.8906 GB`

&nbsp;
### 10. Increasing the batch size

- Lastly, we increase the batch size to the largest power of 2 supported by the GPU

Before:
- `Step tok/sec: 127345`
- `Reserved memory: 5.8906 GB`

After:
- `Step tok/sec: 142156`
- `Reserved memory: 22.5078 GB`


&nbsp;
## Multi-GPU speed comparisons

This may not be an entirely fair comparison as we now use 4 GPUs instead of 1, but using distributed data parallelism, the fastest multi-GPU technique that can be used if the training is not bottle-necked by limited GPU memory, can, of course, result in noticeable speed-ups:

Before (single GPU):
- `Step tok/sec: 142156`
- `Reserved memory: 22.5078 GB`

After (4 GPUs):
- `Step tok/sec: 419259`
- `Reserved memory: 22.7969 GB`
