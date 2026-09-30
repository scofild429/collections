---
title: "LLM Training"
org_id: "b1c8b071-5075-45a1-af42-8ab436510b96"
author: "Technical Discussion Notes"
---

# LLM Training

## Tokenizer

### Training a BPE tokenizer

Byte Pair Encoding (BPE) builds a vocabulary by repeatedly merging frequent adjacent units in a training corpus. The starting units depend on the implementation: characters for character-based BPE, or a representation of the 256 possible byte values for byte-level BPE. Normalization, pre-tokenization boundaries, special tokens, and stopping criteria also affect the resulting vocabulary.

A tokenizer is usually trained or selected before language-model training and then kept fixed. Vocabulary extension is possible, but it also requires updating the model's input embeddings and output vocabulary projection. BPE is one tokenizer family; WordPiece and Unigram are other choices. See the [Tokenizers component documentation](https://huggingface.co/docs/tokenizers/en/components).

| Choice | Typical tradeoff |
|---|---|
| Smaller vocabulary | Smaller embedding/output matrices, but often longer token sequences |
| Larger vocabulary | Often shorter sequences, but larger embedding/output matrices and more logits per position |
| Different training corpus | Different token coverage and segmentation, even at the same vocabulary size |

A larger vocabulary does not guarantee faster inference: sequence length, output projection cost, hardware, and language distribution all matter. Byte coverage can avoid unknown tokens for representable input, but token IDs need not equal raw byte values. Normalization may change text, so exact round-trip recovery is a property to test for a specific tokenizer, not a universal guarantee.

### From text to model input

For vocabulary size $V$, batch size $B$, sequence length $N$, and embedding width $d_{emb}$:

1. Tokenization maps text to integer IDs, shaped `[N]` or `[B, N]` after batching.
2. An embedding table $E\in\mathbb R^{V\times d_{emb}}$ gathers one row per ID.
3. The resulting tensor has shape `[N, d_emb]` or `[B, N, d_emb]`.

Tokenization commonly runs on the CPU and model computation on an accelerator, but this is an implementation choice. Repeated tokens reuse the same row; IDs identify vocabulary entries, not unique occurrences.

The same linear transformation is applied independently at each position:

$$
[N,d_{model}][d_{model},d_{out}]=[N,d_{out}].
$$

This shared computation supports variable sequence lengths. It does not provide unlimited context: position handling, training distribution, attention cost, KV-cache size, and implementation limits still constrain usable length.

### When embeddings are trainable

| Training phase | Possible embedding treatment |
|---|---|
| Pretraining | Usually trained along with the model |
| Continued pretraining | Usually trainable; vocabulary extension is optional |
| Supervised fine-tuning (SFT) | Trainable in full fine-tuning, often frozen in adapter-only recipes |
| RLHF or DPO | Either trainable or frozen, depending on the recipe |

Freezing embeddings is not a general solution to reward hacking or forgetting. These depend on the objective, data, optimization, and the rest of the model as well.

For a pure embedding lookup, gradients from repeated IDs add into the same row. The example below produces gradient `[2, 2, 2]` for row 1, `[1, 1, 1]` for row 4, and zero for the other rows:

```python
import torch

embedding = torch.nn.Embedding(6, 3)
ids = torch.tensor([1, 4, 1])
embedding(ids).sum().backward()
assert torch.equal(embedding.weight.grad[1], torch.full((3,), 2.0))
```

The default PyTorch gradient representation is dense; `sparse=True` is an explicit option with optimizer restrictions. Moreover, tied output weights receive gradients through the vocabulary projection too. Even zero lookup gradients do not guarantee unchanged parameters when momentum or weight decay is active. See [Embedding documentation](https://docs.pytorch.org/docs/2.14/generated/torch.nn.Embedding.html).

## Embedding width, hidden width, and compute

| Symbol | Meaning |
|---|---|
| $d_{emb}$ | Width of each row in the token embedding table |
| $d_{model}$ | Width of the Transformer residual stream |
| $d_{ff}$ | Intermediate width inside the feed-forward network |
| $d_{head}$ | Width of an attention head; not necessarily the embedding width |

Many Transformers use $d_{emb}=d_{model}$. Residual addition requires matching shapes at each addition, but a smaller embedding can be projected into the residual width once before the blocks. It does not require an extra projection at every residual connection.

Direct input/output weight tying is simplest when the widths match. Factorized designs can use additional projections to tie a smaller embedding table. With an input projection, the embedding parameter count becomes

$$
Vd_{emb}+d_{emb}d_{model}
$$

instead of $Vd_{model}$. [ALBERT](https://arxiv.org/abs/1909.11942) is an example of factorized embedding parameterization. As a hypothetical size calculation, $128{,}000\times12{,}288=1{,}572{,}864{,}000$ parameters for an unfactorized table; this is not a statement about a particular model's vocabulary configuration.

Embedding gathers often depend heavily on memory bandwidth. Large matrix multiplications can be compute-bound, but small-batch autoregressive decoding can also be bandwidth-bound. Sharding and communication costs depend on the partitioning scheme, batch size, and hardware; neither a particular collective nor distributed sharding is inherently required by an embedding layer.

<a id="07364BC0-33F0-4FE2-9BA7-FA73E9A8A769"></a>

## Position

### 为什么需要位置信息？

在没有位置相关信号的**无掩码双向自注意力**中，同步置换输入 token 会使输出按同样方式置换。逐位置前馈网络也保留这种排列等变性。它描述的是输出如何随输入排列变化，并不意味着所有输出向量都相等。

需要注意适用条件：固定的 causal mask 限制每个位置只能看到前缀，任意置换会改变可见关系，因此不能据此断言“没有显式位置编码的自回归模型，对相同词集合一定产生相同预测”。无显式位置编码的因果模型也可能学习位置相关行为；参见 [NoPE 研究](https://arxiv.org/abs/2305.19466)。

显式位置方法提供位置或距离信号，但不同方法的效果和外推能力仍需实验验证。

### 常见方法及其限制

| 方法 | 注入方式 | 注意事项 |
|---|---|---|
| Sinusoidal absolute PE | 将固定的正余弦位置向量加到输入表示上 | 公式可计算任意位置，不等于模型能可靠外推 |
| Learned absolute PE | 查询可训练的位置表 | 超出表范围通常会索引失败；可扩表并继续训练，不必必然从头训练 |
| Relative position methods | 在注意力中加入相对距离相关项 | 有多种实现；不能统一描述为同一种 bias 或同一种性能开销 |
| RoPE | 按位置旋转 query/key 的成对维度 | 点积具有相对位置结构，但不保证无限上下文 |

这些方法是不同设计选择，不是前一种被彻底淘汰的线性历史。原始 [Transformer](https://arxiv.org/abs/1706.03762) 比较过学习式和正弦位置编码，报告了相近结果。

### RoPE 的数学结构

设旋转维度 $d$ 为偶数，基数 $b>1$，第 $i$ 对维度的角频率为

$$
\theta_i=b^{-2i/d},\qquad i=0,\ldots,d/2-1.
$$

在位置 $m$，该维度对旋转 $m\theta_i$ **弧度**。通常旋转的是每个 attention head 的 query/key，且可能只旋转部分 head 维度，而不是直接旋转原始 token embedding。

对二维旋转矩阵 $R(\alpha)$，有

$$
(R(m\theta)q)^T(R(n\theta)k)=q^TR((n-m)\theta)k.
$$

因此，在给定未旋转的 $q,k$ 时，位置项通过相对位移进入点积；点积仍然依赖内容。它不是“只由距离决定”的相似度，也不保证随距离单调下降。参见 [RoFormer](https://arxiv.org/abs/2104.09864)。

高频维度对的相位变化较快，低频维度对变化较慢。可以借助秒针/时针理解多尺度信号，但不能把某对维度严格指定为“局部语法”或“全局语义”。增大 $b$ 会降低 $i>0$ 的频率，$i=0$ 的频率仍为 1；这也不保证所有新位置都回到训练中熟悉的相位分布。

以下是单个向量、相邻维度配对的教学实现。某些模型使用 split-half 配对；更换约定时必须同时匹配权重布局。

```python
import torch

def rope_vector(x, position, base=10000.0):
    if x.ndim != 1 or x.numel() == 0 or x.numel() % 2:
        raise ValueError("Expected a nonempty vector with even width")
    if not x.is_floating_point() or base <= 1:
        raise ValueError("Expected floating-point input and base > 1")
    pairs = x.reshape(-1, 2)
    frequencies = base ** (-torch.arange(
        0, x.numel(), 2, device=x.device, dtype=x.dtype
    ) / x.numel())
    angle = position * frequencies
    c, s = angle.cos(), angle.sin()
    return torch.stack((pairs[:, 0] * c - pairs[:, 1] * s,
                        pairs[:, 0] * s + pairs[:, 1] * c), dim=-1).flatten()
```

### YaRN：频率混合与注意力缩放

单纯把所有位置除以扩展倍数 $s$，等价于把所有频率除以 $s$。这会同时改变高频的局部位置差异。YaRN 根据原训练长度 $L$ 内各频率完成的旋转圈数，保留高频部分、缩放低频部分，并在中间做过渡。

对原始频率 $\theta_i$，圈数为 $L\theta_i/(2\pi)$。对应给定圈数 $r$ 的维度对索引为

$$
i(r)=\frac{d\log(L/(2\pi r))}{2\log b}.
$$

常见设置使用 `beta_fast=32`、`beta_slow=1`。它们是圈数阈值，不能直接拿来作为“波长除以上下文长度”的阈值。下面展示常见的索引 ramp 形式；实际模型的配置和变体应以对应实现为准。参考 [YaRN 论文](https://arxiv.org/html/2309.00071v2)及 [Transformers RoPE 实现](https://github.com/huggingface/transformers/blob/v4.51.3/src/transformers/modeling_rope_utils.py)。

```python
import math
import torch

def yarn_frequencies(dim, base=10000.0, length=4096, factor=8.0,
                     beta_fast=32.0, beta_slow=1.0):
    if dim <= 0 or dim % 2 or base <= 1 or length <= 0 or factor < 1:
        raise ValueError("Invalid rotary dimension, base, length, or factor")
    if not 0 < beta_slow < beta_fast:
        raise ValueError("Expected 0 < beta_slow < beta_fast")

    def correction_index(rotations):
        return dim * math.log(length / (2 * math.pi * rotations)) / (2 * math.log(base))

    low = max(0, math.floor(correction_index(beta_fast)))
    high = min(dim - 1, math.ceil(correction_index(beta_slow)))
    pair_index = torch.arange(dim // 2, dtype=torch.float64)
    blend = ((pair_index - low) / max(high - low, 0.001)).clamp(0, 1)
    original = base ** (-2 * pair_index / dim)
    return original * (1 - blend) + (original / factor) * blend
```

YaRN 还引入注意力温度调整。常见幅度因子为 $m=1+0.1\log s$；如果同时将旋转后的 $Q,K$ 乘以 $m$，点积会乘以 $m^2$。论文中的温度写法 $\operatorname{softmax}(QK^T/(t\sqrt d))$ 对应 $m=\sqrt{1/t}$。这是一种经验设计，不是严格恢复短上下文分布的定理；扩展后的质量和内存开销仍需评估。

<a id="64FD034D-BF19-4737-9BA4-AB473346A338"></a>

## Flash Attention

### From ordinary softmax to online normalization

Ordinary softmax is $p_i=e^{x_i}/\sum_j e^{x_j}$. Direct exponentiation can overflow. Subtracting the maximum gives the mathematically equivalent stable form

$$
p_i=\frac{e^{x_i-m}}{\ell},\qquad m=\max_j x_j,\quad \ell=\sum_j e^{x_j-m}.
$$

For two blocks with summaries $(m_A,\ell_A)$ and $(m_B,\ell_B)$, merge them using

$$
m=\max(m_A,m_B),\qquad
\ell=e^{m_A-m}\ell_A+e^{m_B-m}\ell_B.
$$

A block of size one gives scalar online softmax; larger blocks give block online softmax. Applying the same recurrence independently along each row gives batched online softmax. This implementation combines those cases:

```python
import torch

def block_softmax(x, block_size=3):
    """Educational two-pass softmax over the last axis of finite float input."""
    if x.ndim == 0 or x.shape[-1] == 0 or block_size <= 0:
        raise ValueError("Expected a nonempty last axis and positive block size")
    if not x.is_floating_point() or not torch.isfinite(x).all():
        raise ValueError("This example requires finite floating-point inputs")
    work = x.double() if x.dtype == torch.float64 else x.float()
    maximum = normalizer = None
    for block in work.split(block_size, dim=-1):
        block_max = block.amax(dim=-1, keepdim=True)
        block_sum = (block - block_max).exp().sum(dim=-1, keepdim=True)
        if maximum is None:
            maximum, normalizer = block_max, block_sum
        else:
            merged_max = torch.maximum(maximum, block_max)
            normalizer = (normalizer * (maximum - merged_max).exp()
                          + block_sum * (block_max - merged_max).exp())
            maximum = merged_max
    return (work - maximum).exp() / normalizer

x = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
assert torch.allclose(block_softmax(x), torch.softmax(x, dim=-1))
```

The final line revisits the logits: a normalization summary alone cannot emit all softmax probabilities. This example deliberately excludes `-inf` masks; production attention kernels must handle masked tiles and fully masked rows without evaluating undefined `-inf - (-inf)` expressions.

### From softmax summaries to attention

For attention, maintain a weighted numerator as well:

$$
u_A=\sum_{j\in A}e^{x_j-m_A}v_j,\qquad
u=e^{m_A-m}u_A+e^{m_B-m}u_B,\qquad o=u/\ell.
$$

This allows tiled computation of $\operatorname{softmax}(QK^T/\sqrt{d})V$ without storing the full attention matrix in high-bandwidth memory. [FlashAttention](https://arxiv.org/abs/2205.14135) uses this structure with hardware-aware tiling and backward recomputation. It computes dense attention exactly in mathematical terms, subject to floating-point differences; it does not turn dense attention's quadratic arithmetic into linear attention. The Python example demonstrates the normalization identity, not a fast GPU kernel.

<a id="0DE9F01A-CAFE-4A64-8156-1D46179C4ECB"></a>

## Objective Function

For a finite trajectory $\tau=(s_0,a_0,\ldots,s_{T-1},a_{T-1},s_T)$ with parameter-independent initial distribution and dynamics,

$$
P_\theta(\tau)=\rho_0(s_0)\prod_{t=0}^{T-1}
\pi_\theta(a_t|s_t)P(s_{t+1}|s_t,a_t),
$$

$$
J(\theta)=\mathbb E_{\tau\sim\pi_\theta}[R(\tau)],\qquad
\nabla_\theta\log P_\theta(\tau)=\sum_{t=0}^{T-1}
\nabla_\theta\log\pi_\theta(a_t|s_t).
$$

Here $T$ is the number of actions in the trajectory, not the number of possible actions. In an autoregressive LLM, the state is the prompt plus generated prefix, and the action is the next token.

The state value $V^\pi(s)$ is expected return from a state; $Q^\pi(s,a)$ conditions additionally on the action; $A^\pi(s,a)=Q^\pi(s,a)-V^\pi(s)$ measures improvement relative to the policy's average action at that state.

Using reward $r_t$ for the transition $s_t\to s_{t+1}$, a correct $n$-step estimate is

$$
\hat A_t^{(n)}=\sum_{k=0}^{n-1}\gamma^k r_{t+k}
+\gamma^n V(s_{t+n})-V(s_t).
$$

It sums **rewards** along the intermediate steps and bootstraps once at the end. With $\delta_t=r_t+\gamma V(s_{t+1})-V(s_t)$, finite-rollout GAE is

$$
\hat A_t=\sum_{l=0}^{T-t-1}(\gamma\lambda)^l\delta_{t+l}.
$$

Use zero bootstrap at true terminal states and the appropriate estimated continuation value at nonterminal truncations. Full derivations and executable GAE/PPO examples are in [[RL]].

<a id="5B79EAED-01A9-4F1F-9AD7-B69C78A44FC4"></a>

## VPG: Vanilla Policy Gradient

For an undiscounted finite episode, an advantage-based estimator is

$$
\hat g=\sum_{t=0}^{T-1}\nabla_\theta\log\pi_\theta(a_t|s_t)\hat A_t.
$$

For a discounted start-state objective, the corresponding trajectory expression includes an outer $\gamma^t$ factor. A state-dependent baseline can reduce variance without biasing the exact policy gradient; approximate advantage estimates can introduce bias. The advantage is treated as a fixed target during the actor update. Step size and estimation noise affect stability.

<a id="50F66E3A-067B-4939-88CA-43C342463A36"></a>

## TRPO: Trust Region Policy Optimization

With data from $\pi_{old}$, define $\rho_t(\theta)=\pi_\theta(a_t|s_t)/\pi_{old}(a_t|s_t)$. The practical surrogate problem is

$$
\max_\theta\;\mathbb E_t[\rho_t(\theta)\hat A_t],\qquad
\text{subject to }\mathbb E_t[D_{KL}(\pi_{old}(\cdot|s_t)\|\pi_\theta(\cdot|s_t))]\leq\delta.
$$

This uses sampled old-policy states and an approximate optimization procedure. The action ratio does not exactly transform the old state distribution into the new one, and the practical average-KL constraint is not an unconditional monotonic-improvement guarantee. See [TRPO](https://arxiv.org/abs/1502.05477).

<a id="E4A4F6F9-D5E9-4E99-ACD4-44D069FAAFDE"></a>

## PPO: Proximal Policy Optimization

The **actor** minimizes the negative clipped policy surrogate:

$$
\mathcal L_{actor}=-\mathbb E_t[\min(\rho_t\hat A_t,
\operatorname{clip}(\rho_t,1-\epsilon,1+\epsilon)\hat A_t)].
$$

The **critic** predicts returns and can minimize

$$
\hat R_t=\operatorname{stopgrad}(V_{old}(s_t)+\hat A_t),\qquad
\mathcal L_{value}=\mathbb E_t[(V_\psi(s_t)-\hat R_t)^2].
$$

The target must be fixed during this update. Writing $(V_\psi-(\hat A+V_\psi))^2$ without detachment cancels the value prediction and gives the wrong gradient. PPO clipping discourages some excessive policy changes but does not impose a hard bound on every probability ratio or KL divergence. See [PPO](https://arxiv.org/abs/1707.06347) and [[RL]].

<a id="55FE69BE-E330-44AA-8A81-FA25D6D68565"></a>

## GRPO: Group Relative Policy Optimization

Generate $G$ responses to the **same prompt**, assign rewards $r_1,\ldots,r_G$, and define each response's advantage using its own reward:

$$
\bar r=\frac1G\sum_i r_i,\qquad
\hat A_i=\frac{r_i-\bar r}{\operatorname{std}(r_1,\ldots,r_G)+\varepsilon}.
$$

The small denominator stabilizer is an implementation choice. Equal rewards give zero reward-based advantages; the group supplies no relative preference signal. Rewards may come from rules, verifiers, or a learned model. GAE also does not inherently require a learned reward model.

The original outcome-supervised GRPO objective averages each response's token terms by its length, then averages across responses:

$$
\mathcal J=\mathbb E\left[\frac1G\sum_{i=1}^G\frac1{|o_i|}
\sum_{t=1}^{|o_i|}\left(\min(\rho_{i,t}\hat A_i,
\operatorname{clip}(\rho_{i,t},1-\epsilon,1+\epsilon)\hat A_i)
-\beta\hat D_{KL,i,t}\right)\right].
$$

GRPO removes a separately trained value estimator by using group-relative rewards. The KL term regularizes against a reference policy; that reference is conceptually distinct from the old rollout policy. Later variants may change normalization or KL treatment, so identify the variant when comparing results. See [DeepSeekMath](https://arxiv.org/html/2402.03300v3).

<a id="66ec81c0-f137-4b6e-a12a-d8f2fc4303f4"></a>

## DPO: Direct Preference Optimization

### 从 KL 正则化奖励目标到偏好损失

固定 prompt $x$，设 $\beta>0$，参考策略为 $\pi_{ref}$，且配分函数有限。在参考策略的支持集内，考虑

$$
J_x(\pi)=\mathbb E_{y\sim\pi(\cdot|x)}[r(x,y)]
-\beta D_{KL}(\pi(\cdot|x)\|\pi_{ref}(\cdot|x)).
$$

定义

$$
Z(x)=\sum_y\pi_{ref}(y|x)e^{r(x,y)/\beta},\qquad
\pi^*(y|x)=\frac{\pi_{ref}(y|x)e^{r(x,y)/\beta}}{Z(x)}.
$$

代入并合并对数项，得到

$$
J_x(\pi)=\beta\log Z(x)-\beta D_{KL}(\pi(\cdot|x)\|\pi^*(\cdot|x)).
$$

对于固定奖励，$Z(x)$ 与待优化策略无关，因此最大化原目标等价于最小化这个 KL。若允许在完整分布空间中优化，最优解为 $\pi^*$；受参数化模型和训练过程限制，实际不一定能达到它。

反解奖励：

$$
r(x,y)=\beta\log\frac{\pi^*(y|x)}{\pi_{ref}(y|x)}+\beta\log Z(x).
$$

DPO 用策略 $\pi_\theta$ 参数化隐式奖励。这里是奖励的重新参数化，并不是证明任意当前策略都已经等于原奖励的最优策略。

采用 Bradley–Terry **建模假设**，偏好概率为

$$
P(y_w\succ y_l|x)=\sigma(r(x,y_w)-r(x,y_l)).
$$

同一 prompt 下相减消去了仅依赖 $x$ 的项，于是得到

$$
\mathcal L_{DPO}=-\mathbb E_{(x,y_w,y_l)}\log\sigma\left(
\beta\left[\log\frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)}
-\log\frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)}\right]\right).
$$

其中回答的对数概率是回答 token 条件对数概率之和；原始形式不是按回答长度取平均。训练时要正确屏蔽 prompt 和 padding，并使用数值稳定的 `logsigmoid`。

[DPO](https://arxiv.org/html/2305.18290v3) 可以直接使用离线偏好对，不必单独训练一个显式奖励模型。但偏好数据并不天然满足 Bradley–Terry 假设，有限数据和优化也不保证得到“完美人类偏好分布”。

## On-Policy Distillation

A topic for further study: generate prefixes with the student, then train it using a teacher's outputs or token distributions on those prefixes. This exposes the teacher to contexts the student actually visits. A complete algorithm must specify the sampling policy, divergence direction, token/sequence weighting, teacher access, and whether gradients through sampling are estimated or ignored. This heading does not yet specify a particular published method or a validated training recipe.

## Tensor Cores and mixed precision

Tensor Core instructions perform matrix multiply-accumulate operations with supported input and accumulation types that depend on the GPU architecture and instruction. Low-precision inputs with FP32 accumulation are common, but it is misleading to describe every multiplication as first rounded to FP16 and then expanded to FP32. Product precision and accumulation behavior must be checked for the selected instruction.

FP32 accumulation reduces some rounding error; it does not make the computation exact. Input quantization, output casts, optimizer state, master weights, and loss scaling are separate considerations. See the [NVIDIA PTX matrix instruction documentation](https://docs.nvidia.com/cuda/parallel-thread-execution/#warp-level-matrix-instructions-mma).

## Import provenance

<!-- Org properties: {"craft_id": "B9ADD087-5200-4579-AD9C-0E2F6BFD911A", "imported": "2026-09-27"} -->

Source: [AI in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=B9ADD087-5200-4579-AD9C-0E2F6BFD911A)
