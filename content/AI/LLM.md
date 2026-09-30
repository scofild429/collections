---
title: "LLM Training"
org_id: "b1c8b071-5075-45a1-af42-8ab436510b96"
author: "Technical Discussion Notes"
---

# LLM Training

## Tokenizer

### The Mechanics of Tokenizer Training (BPE)

Tokenizer training is a data-compression task that occurs completely independently of, and prior to, building the neural network.

#### A. The Lifecycle of a Byte Pair Encoding (BPE) Run

1.  **Initialization:** A massive training corpus is ingested. The vocabulary starts at the absolute smallest atomic units (typically the 256 unique raw UTF-8 bytes).
2.  **Frequency Evaluation:** The training script loops through the corpus to count the most frequently appearing adjacent pairs (e.g., `t` + `h`).
3.  **Recursive Merge:** The most frequent pair is permanently merged into a single new entry (`th`) and added to the vocabulary list.
4.  **Termination:** This loop recurses continuously until a hardcoded vocabulary limit is reached (e.g., 6,400 for MiniMind, or 128,000 for Llama 3).

#### B. Vocabulary Size vs. Tokenization Depth

- **Small Vocabularies (e.g., 6,400):** The tokenizer has fewer long words stored in its dictionary. It is forced to split text "deeper" into tiny, highly fragmented fragments. This reduces the parameter size of the model's embedding/output layers but demands more tokens to represent a sentence.
- **Large Vocabularies (e.g., 151,000):** The tokenizer can store complex words, phrases, and multi-byte characters as single tokens. It splits text less, resulting in fewer tokens per sentence and higher computational efficiency during inference.
- **Data Dependency:** Two tokenizers with identical vocabulary lengths will have completely different mappings depending on their training corpus (e.g., English-centric, Code-centric, or Chinese-centric).

### The Core Pipeline: From Text to LLM Input

The transition from raw natural language to numerical representations that a Transformer can process follows a strict, decoupled pipeline.

#### A. Step 1: Text-to-ID Mapping (CPU Space)

- **Mechanism:** The input text string (e.g., `"I love Programming"`) is intercepted by a CPU-side pre-processor running a pre-compiled tokenizer map (e.g., derived from `train_tokenizer.py`).
- **BPE Output:** The string is parsed into subwords/characters and translated into a 1D vector of unique token integer IDs based on the static vocabulary index (e.g., `[23, 67, 45]`).
- **Data Integrity:** Modern tokenizers utilize **Byte-Level BPE (BBPE)**. If a word or emoji is entirely missing from the vocabulary, it falls back to raw UTF-8 bytes (0-255). This guarantees that **no text is ever unidentified or dropped** (eliminating the old `<UNK>` tokens).

#### B. Step 2: ID-to-Vector Layer Lookup (GPU Space)

- **Mechanism:** The integer vector `[23, 67, 45]` enters the model's **Embedding Layer** (`embed_tokens`).

- **The Matrix Structure:** The embedding layer is a massive 2D lookup table of shape:

  $$
  \text{Shape} = [\text{Vocab Size}, \text{Hidden Dimension}]
  $$

  For example, a vocabulary size of 64,000 and a hidden width of 4,096 yields a matrix of shape `[64000, 4096]`.

- **Tensor Assembly:** The layer performs a fast programmatic row-lookup matching the indices. Instead of concatenating these vectors horizontally, it **stacks them vertically**.

- **Resulting Tensor:** It produces a 2D float tensor of shape `[Sequence Length, Hidden Dimension]` (e.g., `[3, 4096]`), representing the token features sequentially.

#### C. Step 3: LLM Ingestion & Variable Sequence Lengths

- **The Mathematical Trick:** Traditional deep learning models require fixed-size inputs. LLMs bypass this limitation via **Matrix Multiplication Broadcasting**.

- **The Formula:** When the input matrix passes into the first dense weight layers of the Transformer (which have a fixed shape of `[4096, 4096]`), the mathematical dot product preserves the sequence dimension ($N$):

  $$
  [N, 4096] \times [4096, 4096] = [N, 4096]
  $$

- **Flexibility:** The LLM is structurally rigid on its **width** (4096), but completely infinite and flexible on its **height/length** ($N$).

### Lifecycle of the Embedding Layer Across LLM Training Phases

The exact same physical weight matrix (`embed_tokens`) persists through every single downstream training phase, but its training status changes dramatically.

| Training Phase | Layer Status | Modification Type & Mechanism |
|----|----|----|
| 1\. Pre-Training | **Active** | Fully unfrozen. Initialized as pure random floating-point noise. Learns base semantics via backpropagation over trillions of tokens. |
| 2\. Continued PT | **Active** | Semi-unfrozen. Often undergoes "Embedding Surgery" where new rows are appended to the matrix to accommodate new languages or characters. |
| 3\. SFT | **Frozen** | Locked behind glass. Fine-tuning switches to Parameter-Efficient methods (like LoRA) targeting internal attention layers to adjust tone/format. |
| 4\. RLHF / DPO | **Frozen** | Strictly locked down. Freezing prevents "Reward Hacking" and safeguards against catastrophic forgetting of core vocabulary. |

#### A. How the Backward Pass Updates the Embedding Matrix (The Scatter Mechanism)

- **The Problem:** If a forward pass only calls 3 tokens out of 64,000, how does backpropagation know which rows to modify?

- **The Mechanism:** The framework's automatic differentiation engine utilizes a **Sparse Gradient Update** (conceptually mapped via programmatic operations like `scatter_add_`).

- **Row-Level Routing:** The gradient tensor returning from the LLM has an identical shape to the input tensor (`[3, 4096]`). The engine reads its execution log, identifies the precise token indices used (e.g., `23, 67, 45`), and adds the incoming gradients **only** to those exact rows:

  ``` python
  embedding_matrix.grad[23] += incoming_gradient[0]
  embedding_matrix.grad[67] += incoming_gradient[1]
  embedding_matrix.grad[45] += incoming_gradient[2]
  ```

- **Efficiency:** The remaining 63,997 uncalled rows in the embedding matrix receive an update value of exactly `0.0`, ensuring their historical semantic weights remain perfectly preserved until they are explicitly called in a future batch.

## Hidden dimension Vs. embedding dimension

In the architectural design of Large Language Models (LLMs), the relationship between the **Hidden Dimension** (often denoted as $d_{model}$ or $h$) and the **Embedding Dimension** (often denoted as $d_{emb}$) can be summarized in one sentence: In mainstream standard Transformer models (like the LLaMA and GPT series), they are typically strictly equal ($d_{model} = d_{emb}$); however, in architectures pursuing parameter optimization or specific system designs, they are intentionally decoupled ($d_{model} > d_{emb}$). Although they often share the same variable name in many engineering implementations, they carry entirely distinct physical meanings from the perspectives of deep learning logic and system computation.

### 1. Fundamental Conceptual Differences

- **Embedding Dimension ($d_{emb}$):** This is the feature representation at the "input layer" of the model. It determines the information capacity when each token in the vocabulary (such as subwords segmented by BPE or Tiktoken) is mapped into a continuous vector space. The shape of the corresponding Embedding matrix is $|V| \times d_{emb}$, where $|V|$ is the vocabulary size.
- **Hidden Dimension ($d_{model}$):** This is the processing bandwidth at the "intermediate layers" of the model. It determines the vector length used inside the Transformer Blocks (including the Self-Attention layers and Feed-Forward Network layers) when performing multidimensional feature extraction, information fusion, and attention weight computation.

### 2. Why Are They Typically Equal in Mainstream Models?

In the original Transformer architecture (and the vast majority of mainstream LLMs that followed), designers defaulted to $d_{model} = d_{emb}$. This is primarily driven by the following two architectural constraints:

#### A. Requirements of Residual Connections

Transformers heavily rely on residual connections to mitigate the vanishing gradient problem. The core formula is:

$$
x_{out} = x_{in} + \text{Sublayer}(x_{in})
$$

For the addition operation to be performed directly, the dimension of the input vector $x_{in}$ (coming from the Embedding layer or the previous layer) must be exactly identical to the dimension of the output from the Attention or MLP sublayers. If they were different, an additional linear projection matrix would have to be introduced before every residual addition, which would significantly increase the communication and computation overhead of Matrix Multiplication (GEMM).

#### B. Weight Tying

When training language models, it is common to share the same weight parameters between the Embedding matrix at the input layer and the Pre-Softmax linear projection matrix at the output layer (i.e., $W_{in} = W_{out}^T$). This design not only substantially reduces the number of parameters but also leverages the gradients during the decoding process to directly update the word embeddings. To implement this matrix transposition and reuse, $d_{emb}$ must perfectly match the $d_{model}$ output by the final layer of the Transformer.

### 3. Decoupled Design: When $d_{emb} \neq d_{model}$

As model sizes scale up, forcing $d_{emb} = d_{model}$ introduces severe parameter redundancy and VRAM bottlenecks. The vocabulary size $|V|$ is usually between $30,000$ and $128,000$ (or even larger). If the hidden dimension $d_{model}$ is expanded to a very large number (for instance, $12288$ in GPT-3), the parameter count for the Embedding layer alone would reach:

$$
128,000 \times 12288 \approx 1.57 \times 10^9 \text{ parameters}
$$

**Factorized Embedding Parameterization** This technique (widely used in models like ALBERT) decouples the two dimensions, enforcing $d_{emb} \ll d_{model}$. The system introduces an intermediate projection matrix $E_{proj}$ to break down the original one-step mapping into two steps:

1.  First, map the one-hot vector into a lower-dimensional Embedding space: $E_{in} \in \mathbb{R}^{|V| \times d_{emb}}$
2.  Then, project it into the high-dimensional hidden space via a linear transformation: $E_{proj} \in \mathbb{R}^{d_{emb} \times d_{model}}$

With this approach, the parameter count drops precipitously from $O(|V| \times d_{model})$ to $O(|V| \times d_{emb} + d_{emb} \times d_{model})$. This decoupling is logically very consistent: the context-independent representation of words (Embedding) does not require an extremely high dimension, because the static semantics of a single word are relatively limited; conversely, the context-dependent representations (Hidden States) require extremely high dimensions, as they must encode complex grammar, semantics, and reasoning logic across long contexts.

### 4. System-Level (HPC) Considerations for Training and Inference

From the perspective of High-Performance Computing (HPC) and memory hierarchies, the difference between these two layers has a decisive impact on compute allocation:

- **The Embedding Layer is Memory-Bandwidth Bound:** Word vector lookup is inherently a sparse memory read operation (Gather) and does not involve dense floating-point operations. A massive $d_{emb}$ directly leads to extremely high cache miss rates and VRAM bus pressure. Particularly in distributed training environments (such as multi-node training on Kubernetes clusters), giant Embedding tables usually require Tensor Parallelism splitting, which incurs additional All-Reduce communication overhead.
- **The Hidden Layer is Compute Bound:** The operations based on $d_{model}$ within Transformer Blocks are dense matrix multiplications (with a complexity of $O(d_{model}^2)$). Modern GPUs and AI accelerators are heavily optimized for large-scale dense matrix multiplications. Increasing $d_{model}$ can more effectively saturate the compute capacity (improving FLOPs utilization).

<a id="07364BC0-33F0-4FE2-9BA7-FA73E9A8A769"></a>

## Position

### Problem

#### 1. 核心痛点：Transformer 是天生的“词袋”

没有位置编码的 Transformer 存在一个致命的数学缺陷——排列等变性（Permutation Equivariance）。 自注意力机制本质上是对上下文进行无序的“加权求和”。这就导致输入“熊猫吃竹子”和“竹子吃熊猫”时，模型看到的仅仅是包含这三个词的无向图。

#### 2. 数学本质：同步置换（0差异）

当输入词序被打乱时，模型的输出矩阵只是在行顺序上做了对应的同步置换，但每一个词提取到的特征向量在数值上是 100% 相等的。

- 对“吃”这个词来说，它无法通过特征向量知道自己左边是主语还是右边是主语。
- 这种差异并非“常数差异”，而是毫无差异。在模型眼中，它只是一堆词汇的共现（Co-occurrence），没有语法，没有时间先后。

#### 3. 全阶段失效：贯穿模型生命周期的灾难

这种“词汇盲盒”现象会导致模型在没有任何位置约束的情况下，在所有训练和推理阶段全面失效：

- 掩码预训练（MLM/BERT）： 上下文打乱后，预测中间遮挡词的概率分布完全不变。
- 因果自回归（GPT/SFT/推理）： 只要面对相同的“可见历史词汇集合”，无论内部顺序如何，预测下一个词的结果完全一样。
- 强化学习（RL/RM）： 奖励模型对“人类战胜AI”和“AI战胜人类”这两个词汇相同但逻辑相反的句子，会给出绝对相等的 Reward 分数，导致梯度混乱。

#### 一句话总结：

没有位置编码，Transformer 就只是一个瞎摸词块的盲盒；引入位置编码（如RoPE强行注入坐标时间戳），才打破了这种对称性，赋予了模型理解“秩序”和“逻辑”的灵魂。

### 核心细节：维度与高低频的精妙分工

假设一个词向量有 512 维。模型不是用同一个波来记录位置，而是把这 512 个维度（抽屉）分配给了不同频率的波：

#### 低维度（例如第 0, 1, 2 维）：使用高频波

- 用法： 这里使用的是周期极短的正弦/余弦波。位置只要往后挪动 1 步，算出来的数值就会发生剧烈跳变（比如从 0.8 瞬间变成 -0.9）。
- 物理意义（秒针）： 赋予模型极高的局部微观分辨率。这让模型能够极其敏锐地察觉到“这两个词是紧挨着的”，从而精准区分局部的语法关系（如主谓宾连缀）。

#### 高维度（例如第 510, 511 维）：使用低频波

- 用法： 这里使用的是周期极长、极其平缓的波。位置挪动 1 步，数值几乎毫无变化；只有跨越几十上百个词，数值才会有可见的改变。
- 物理意义（时针）： 赋予模型全局宏观定位能力。即使高频波像秒针一样转了几百圈已经乱套了，模型依然能通过低频波判断出“这个词大致在文章的开头，那个词在文章的末尾”。

### 发展历程

#### 第一阶段：原初的固定公式（Transformer 2017）

- 做法：正弦/余弦绝对位置编码（Sinusoidal PE）。使用一套写死的数学公式，生成位置向量，加到词向量上。
- 当时的理由：Transformer 刚刚被发明，作者为了打破“词袋诅咒”，需要一个能表达位置的信号。选择三角函数是因为它能表示无限的长度，且值域永远在 $[-1, 1]$ 之间，不会因为句子太长而导致数值爆炸。
- 为什么被放弃？早期的研究者发现，虽然公式很优雅，但在当时的短文本评测任务（如机器翻译）中，“写死的公式”表现并没有比“让模型自己学”更好。

#### 第二阶段：迷信算力的“可学习”时代（BERT、GPT-1/2/3）

- 做法：可学习的绝对位置编码（Learned Absolute PE）。研究人员抛弃了三角函数，直接初始化一个参数矩阵（比如 \$2048 ×768\$）。第 1 个词加第 1 个随机向量，第 100 个词加第 100 个随机向量，让反向传播自己去更新它们。
- 转换的理由：当时的深度学习有一种“万物皆可学”的信仰。既然词语的含义（Embedding）可以由模型在海量数据中自己学出来，凭什么位置信息要人类用数学公式硬性规定？事实证明，在固定长度的 Benchmark 上，可学习 PE 确实能榨取稍微高一点的准确率。
- 致命危机爆发：零外推能力（Zero Extrapolation）。随着 GPT-3 的出现，大家发现不对劲了。这种方法把模型的“视力”锁死了。如果训练时准备了 2048 个位置的向量，当用户输入第 2049 个词时，模型去查表，拿到的是一个未经任何训练的初始随机噪声。结果就是输出瞬间乱码。为了让模型支持 4096 长度，科技巨头们只能重新初始化一个更大的矩阵，从头开始烧钱训练。

#### 第三阶段：认知的觉醒 —— “相对距离”才是语言的本质

在可学习绝对 PE 碰壁后，语言学的一个核心常识被重新重视：平移不变性（Translation Invariance）。

- 核心反思：人类在阅读时，根本不关心“我”这个字在全书的绝对位置是第 1 个还是第 10001 个。我们在乎的是，“我”在“爱”的前面 1 个位置。有意义的不是绝对坐标，而是相对距离。
- 过渡期解法：相对位置编码（如 T5、ALBERT、DeBERTa）。研究者开始直接在 Attention 的计算公式里打补丁。当词 $i$ 关注词 $j$ 时，直接把它们的距离 $(i-j)$ 算出来，映射成一个偏置（Bias）加到注意力分数里。
- 为什么没有一统天下？（工程灾难）：相对位置编码在理论上非常完美，但在工程计算上极其丑陋。它破坏了 Transformer 原本可以高度并行化的矩阵乘法运算（FlashAttention 等底层算子很难对它进行极致优化），导致模型训练和推理速度大打折扣。

#### 第四阶段：伟大的综合 —— RoPE（旋转位置编码，2021 至今）

此时，大语言模型陷入了一个不可能三角：

1.  我们想要绝对位置的计算效率（直接在输入端处理，不影响底层 Attention 的矩阵乘法）。
2.  我们想要相对位置的表达能力（符合人类语言逻辑）。
3.  我们想要无尽的外推潜力（不能像“可学习参数”那样被长度锁死）。

RoPE (Rotary Position Embedding) 的出现，用一种极其优雅的数学几何方法，同时满足了这三点。

- 它的做法（回归“不学习”的数学）：它彻底抛弃了“可学习”的路线，回归了第一阶段的固定数学公式。但它不再是把位置向量“加”到词向量上，而是用位置 $m$ 乘以一个频率 \$θ\$，构造一个旋转矩阵，去“旋转”词向量。
- 为什么它赢得了时代？
  - 输入端注入绝对位置：第 $m$ 个词旋转 $m\cdot\theta$ 度，第 $n$ 个词旋转 $n\cdot\theta$ 度。计算效率极高。
  - Attention 计算自然浮现相对距离：当计算点积（内积）时，根据三角函数公式，两个旋转后的向量的夹角恰好是 \$(m-n)ċθ\$。模型在底层算力上做的是绝对位置的运算，但在高层语义上感受到的却是相对距离。
  - 完美的外推基础：因为它是纯固定的数学公式（没有任何可训练的位置参数），只要修改公式里的基频（也就是我们前面讨论的 YaRN 和 NTK-Aware 插值技术），就能不花一分钱算力，将模型的认知长度从 4K 暴涨到 128K 甚至 1M。

### RoPE

RoPE（旋转位置编码，Rotary Position Embedding）是目前大语言模型（如 LLaMA, Qwen, Mistral 等）事实上的位置编码标准。它的出现完美解决了 Transformer 架构中绝对位置计算效率与相对位置语义表达之间的矛盾。以下是对 RoPE 的数学原理、工程实现及其突出优势的结构化总结。

#### 一、 数学原理：用绝对旋转表达相对距离

RoPE 的根本目标是寻找一个位置编码函数 \$f\$，使得两个注入了绝对位置信息的词向量，其内积（Attention 的计算核心）仅与它们的相对位置有关：

$$
\langle f(\mathbf{q}, m), f(\mathbf{k}, n) \rangle = g(\mathbf{q}, \mathbf{k}, m-n)
$$

1.  1\. 二维空间的复数推导（欧拉公式的绝佳应用）

    在二维空间中，我们将 Query 向量 $\mathbf{q} = (q_0, q_1)$ 映射为复数 \$*q̃* = q<sub>0</sub> + iq<sub>1</sub>\$。RoPE 将位置信息 $m$ 转化为一个旋转角度 \$mθ\$，通过乘以复平面上的旋转因子 $e^{im\theta}$ 来注入绝对位置：

    $$
    f(\mathbf{q}, m) = \tilde{q} e^{im\theta}
    $$

    $$
    f(\mathbf{k}, n) = \tilde{k} e^{in\theta}
    $$

    计算这两个复向量的内积（即取 $\tilde{q} (\tilde{k})^*$ 的实部）：

    $$
    \text{Re}[ (\tilde{q} e^{im\theta}) (\tilde{k} e^{in\theta})^* ] = \text{Re}[ \tilde{q} \tilde{k}^* e^{i(m-n)\theta} ]
    $$

    在这个结果中，绝对位置 $m$ 和 $n$ 已经解耦，被完全转化为相对位置 \$(m-n)\$。

2.  2\. 推广到多维空间（分块对角矩阵）

    对于隐层维度为 $d$ 的向量（$d$ 必须为偶数），RoPE 将其两两分组，划分为 $d/2$ 个二维子空间。每个子空间分配一个不同的基础频率（Base Frequency）\$θ_i\$。在数学形式上，这等价于对输入向量乘以一个稀疏的分块对角旋转矩阵 \$R<sub>Θ, m</sub>\$：

    $$
    R_{\Theta, m} = \text{diag}\left( \begin{pmatrix} \cos m\theta_0 & -\sin m\theta_0 \\ \sin m\theta_0 & \cos m\theta_0 \end{pmatrix}, \dots, \begin{pmatrix} \cos m\theta_{d/2-1} & -\sin m\theta_{d/2-1} \\ \sin m\theta_{d/2-1} & \cos m\theta_{d/2-1} \end{pmatrix} \right)
    $$

    其中，每一组维度的频率计算公式为：

    $$
    \theta_i = b^{-2i/d}, \quad i \in \{0, 1, \dots, d/2-1\}
    $$

    传统上，基频 $b$ 默认为 \$10000\$。

#### 二、 实现方法：极致稀疏化与逐元素并行

如果直接进行矩阵乘法 \$R<sub>Θ, m</sub> **x**\$，时间复杂度将是 \$O(d<sup>2</sup>)\$。但在现代深度学习框架中，RoPE 利用了该矩阵极度稀疏的特性，将其转化为时间复杂度仅为 $O(d)$ 的逐元素（Element-wise）操作。对于输入向量 \$**x** = \[x<sub>0</sub>, x<sub>1</sub>, …, x<sub>d-1</sub>\]\$：

1.  1\. 向量重排（交错取反）

    定义一个辅助函数 =rotate<sub>half</sub>=，将相邻维度的元素交换并对前半部分取反：

    $$
    \text{rotate\_half}(\mathbf{x}) = [-x_1, x_0, -x_3, x_2, \dots, -x_{d-1}, x_{d-2}]
    $$

2.  2\. 逐元素乘加（Element-wise Operations）

    将原本的矩阵乘法等价展开为：

    $$
    f(\mathbf{x}, m) = \mathbf{x} \odot \cos(m\Theta) + \text{rotate\_half}(\mathbf{x}) \odot \sin(m\Theta)
    $$

    (注：$\odot$ 为 Hadamard 乘积，即逐元素相乘)

3.  3\. 张量广播并行（Broadcasting）

    在工程代码（如 PyTorch）中，我们会预先计算好最大序列长度下的 `cos_cached` 和 `sin_cached` 张量。在 Forward pass 时，利用框架的 Broadcasting 机制，可以在一个时钟周期内对整个 Batch、所有 Attention Heads 以及所有 Sequence Tokens 同时完成旋转操作。这完美兼容并支持了 FlashAttention 等底层算子融合技术。

#### 三、 突出优势与外推基频（Base Frequency Extrapolation）

RoPE 能够一统天下的原因在于它同时打破了计算效率和长度泛化能力的物理限制。

- 优势 1：计算效率与语义表达的完美统一 模型在输入端执行的是绝对位置的计算（加法与逐元素乘法，速度极快且天然适合并行），但在 Attention 层面上，计算结果自然呈现出严格的相对位置衰减（距离越远，内积的期望值越趋近于零），这高度符合人类语言中“局部依赖性强”的先验直觉。
- 优势 2：参数零负担 RoPE 是纯粹的数学变换，没有任何需要通过反向传播更新的参数（可学习的 PE 会占用显存并增加梯度计算）。
- 优势 3：通过“外延基频”实现无缝上下文窗口扩展 这是 RoPE 在大模型时代最不可替代的优势。当模型需要处理超过其训练长度（例如从 4K 扩展到 128K）的文本时，传统的 PE 会因为遭遇未初始化的随机向量或无法解析的极小 Attention 分数而崩溃。而 RoPE 只需要修改其外延基频（Base Frequency \$b\$）。

在基频公式 $\theta_i = b^{-2i/d}$ 中：

- 物理意义：$b$ 决定了位置编码在各个维度上的“波长”。波长越长，旋转相同角度需要跨越的绝对位置距离就越大。
- 外推危机：如果 $m$ 极大（远超训练长度），$m\theta_i$ 的角度会超出模型训练时见过的分布，导致语义匹配混乱。
- 修改基频（RoPE Base Scaling / NTK-Aware）：如果我们将 $b$ 从 $10000$ 放大到 $500000$ 甚至 \$1000000\$（如 LLaMA-2 Long 或 CodeLlama 的做法），会导致 $\theta_i$ 整体变小。
  - 对于高频维度（较小的 \$i\$），波长本身很短，受到 $b$ 放大的影响较小，模型依然能精准分辨相邻词语（局部相对位置不变）。
  - 对于低频维度（较大的 \$i\$），波长被大幅拉长。原本在 128K 长度下会超出范围的巨大旋转角度，被重新“压缩”到了模型熟悉的 $0 \sim 2\pi$ 周期内。
- 结论：仅仅通过将基频常数调大（不需要改变模型结构），配合极少量的长文本微调（甚至零微调），就能让模型在底层数学逻辑上将更长上下文的旋转刻度映射回其已有的认知区间内，实现了以极低算力成本达成数倍甚至数十倍的上下文窗口扩展。

### YaRN 核心机制总结

YaRN 是当前最优的 RoPE 长文本上下文扩展方案。它通过分离解决“坐标寻址错乱”与“注意力分布弥散”两个正交问题，实现了极低成本（甚至 Zero-shot）的上下文窗口外推。

#### 1. 核心痛点与问题 (The Problem)

长文本扩展面临两个物理层面的灾难：

维度坐标畸变 (位置越界)  
文本扩展后（如 4K $\to$ 128K），低频维度会遇到未见过的巨大旋转角度。

- /线性插值 (PI) 的缺陷/：全局等比例压缩破坏了高频维度，导致模型丧失相邻词感知（局部视力受损）。
- /NTK-Aware 的缺陷/：通过修改基频动态压缩，虽保护了部分高频，但仍是全局非线性妥协。

注意力弥散 (熵增危机)  
序列变长导致冗余 Token 激增。海量噪音词的微小 Logits 聚沙成塔，强行稀释了目标信号词的 Softmax 概率，导致模型输出分布变得异常平坦，“抓不住重点”。

#### 2. 数学原理 (The Principle)

YaRN 是一套“组合拳”，分别用两项技术针对性地解决上述两个问题。

1.  2.1 频率分段插值 (解决寻址畸变)

    定义维度的波长为 \$λ_d = \$。计算其与模型原始训练长度 $L$ 的比值 \$x<sub>d</sub> = \$。根据 $x_d$ 将维度彻底“分而治之”：

    高频区 ($x_d \le \alpha$)  
    波长极短，转速极快，仅负责局部绝对相邻语义。\*策略：完全不压缩（保持原样，外推）\*。

    低频区 ($x_d \ge \beta$)  
    波长极长，一圈都没转完，负责全局跨度。\*策略：线性缩放 \$1/s\$（纯内插）\*。

    中频区 ($\alpha < x_d < \beta$)  
    \*策略：平滑混合过渡\*。

2.  2.2 注意力温度缩放 (解决熵增危机)

    为对抗海量冗余 Token 带来的 Softmax 扁平化，YaRN 在 Attention 计算阶段，给所有 Logits 统一乘以一个大于 1 的缩放标量 \$t\$（$t$ 通常与扩展倍数 $s$ 正相关）。

    $$
    \text{Attention} = \text{Softmax}\left( t \cdot \frac{QK^T}{\sqrt{d}} \right) V
    $$

    \*几何意义\*：拉高 Logits 的绝对对比度，强制 Softmax 分布重新变得尖锐（Sharp），完美恢复预训练时短文本的信噪比分布形状。

#### 3. 工程实现 (The Implementation)

YaRN 最优雅的工程体现是在底层实现上摒弃了低效的 `if-else` 分支流，利用 `clamp` 截断构建连续的 Ramp 坡道函数，将三段逻辑统一为纯粹的张量数据流，极致压榨 GPU 并行算力。

``` python
import torch
import math

def get_yarn_frequencies(dim, base=10000, L=4096, s=8, alpha=1.0, beta=32.0):
    """
    无分支 (Branchless) 的 YaRN 频率计算实现
    """
    # 1. 生成所有维度的索引 d: [0, 2, ..., dim-2]
    d = torch.arange(0, dim, 2).float()

    # 2. 计算原始频率 theta 与波长比例 x
    theta_old = base ** (-d / dim)
    wavelengths = 2 * math.pi / theta_old
    x = wavelengths / L

    # 3. Ramp 函数：通过一行 clamp 实现三段分流
    # 大于 1 的截断为 1 (高频保持)，小于 0 的截断为 0 (低频内插)，中间平滑
    gamma = torch.clamp((beta - x) / (beta - alpha), min=0.0, max=1.0)

    # 4. 动态加权计算新频率
    theta_new = gamma * theta_old + (1 - gamma) * (theta_old / s)

    return theta_new
```

<a id="64FD034D-BF19-4737-9BA4-AB473346A338"></a>

## Flash Attention

### Normal Softmax

``` python
import torch
X = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
X_exp_sum = X.exp().sum()
X_softmax_hand = torch.exp(X) / X_exp_sum
print(X_softmax_hand)
```

    tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])

### Safe Softmax

``` python
import torch
X = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
X_max = X.max()
X_exp_sum_sub_max = torch.exp(X-X_max).sum()
X_safe_softmax_hand = torch.exp(X - X_max) / X_exp_sum_sub_max
print(X_safe_softmax_hand)
```

    tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])

### Online Softmax

``` python
import torch
X = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
X_pre = X[:-1]
print('input x')
print(X)
print(X_pre)
print(X[-1])

# we calculative t-1 time Online Softmax
X_max_pre = X_pre.max()
X_sum_pre = torch.exp(X_pre - X_max_pre).sum()

# we calculative t time Online Softmax
X_max_cur = torch.max(X_max_pre, X[-1]) # X[-1] is new data
X_sum_cur = X_sum_pre * torch.exp(X_max_pre - X_max_cur) + torch.exp(X[-1] - X_max_cur)

# final we calculative online softmax
X_online_softmax = torch.exp(X - X_max_cur) / X_sum_cur
print('online softmax result: ', X_online_softmax)
```

    input x
    tensor([-0.3000,  0.2000,  0.5000,  0.7000,  0.1000,  0.8000])
    tensor([-0.3000,  0.2000,  0.5000,  0.7000,  0.1000])
    tensor(0.8000)
    online softmax result:  tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])

### block online softmax

``` python
import torch
X = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
X_block = torch.split(X, split_size_or_sections = 3 , dim = 0) 
print(X)
print(X_block)

# we parallel calculate  different block max & sum
X_block_0_max = X_block[0].max()
X_block_0_sum = torch.exp(X_block[0] - X_block_0_max).sum()

X_block_1_max = X_block[1].max()
X_block_1_sum = torch.exp(X_block[1] - X_block_1_max).sum()

# online block update max & sum
X_block_1_max_update = torch.max(X_block_0_max, X_block_1_max) # X[-1] is new data
X_block_1_sum_update = X_block_0_sum * torch.exp(X_block_0_max - X_block_1_max_update) \
    + torch.exp(X_block[1] - X_block_1_max_update).sum() # block sum

X_block_online_softmax = torch.exp(X - X_block_1_max_update) / X_block_1_sum_update
print(X_block_online_softmax)

```

    tensor([-0.3000,  0.2000,  0.5000,  0.7000,  0.1000,  0.8000])
    (tensor([-0.3000,  0.2000,  0.5000]), tensor([0.7000, 0.1000, 0.8000]))
    tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])

### batch online softmax

``` python
import torch
torch.manual_seed(42)
X_batch = torch.randn(4, 6)
_, d = X_batch.shape

X_batch_block_0 = X_batch[:, :d//2]
X_batch_block_1 = X_batch[:, d//2:]

# we parallel calculate  different block max & sum
X_batch_0_max, _ = X_batch_block_0.max(dim = 1, keepdim = True)
X_batch_0_sum = torch.exp(X_batch_block_0 - X_batch_0_max).sum(dim = 1, keepdim = True)

X_batch_1_max, _ = X_batch_block_1.max(dim = 1, keepdim = True)
X_batch_1_sum = torch.exp(X_batch_block_1 - X_batch_1_max).sum(dim = 1, keepdim = True)

# online batch block update max & sum
X_batch_1_max_update = torch.maximum(X_batch_0_max, X_batch_1_max) # 逐个元素找最大值
X_batch_1_sum_update = X_batch_0_sum * torch.exp(X_batch_0_max - X_batch_1_max_update) \
    + torch.exp(X_batch_block_1 - X_batch_1_max_update).sum(dim = 1, keepdim = True) # block sum

X_batch_online_softmax = torch.exp(X_batch - X_batch_1_max_update) / X_batch_1_sum_update
print(X_batch_online_softmax)
```

    tensor([[0.4256, 0.2742, 0.1525, 0.0075, 0.1221, 0.0180],
            [0.1926, 0.0404, 0.2870, 0.1012, 0.1228, 0.2560],
            [0.0881, 0.2934, 0.0264, 0.2155, 0.1964, 0.1802],
            [0.3666, 0.0882, 0.0908, 0.1541, 0.0717, 0.2286]])

``` python
import torch
torch.manual_seed(42)
X_batch = torch.randn(4, 6)
_, d = X_batch.shape

X_batch_block_0 = X_batch[:, :d//2]
X_batch_block_1 = X_batch[:, d//2:]

# we parallel calculate  different block max & sum
X_batch_0_max, _ = X_batch_block_0.max(dim = 1, keepdim = True)
X_batch_0_sum = torch.exp(X_batch_block_0 - X_batch_0_max).sum(dim = 1, keepdim = True)

X_batch_1_max, _ = X_batch_block_1.max(dim = 1, keepdim = True)
X_batch_1_sum = torch.exp(X_batch_block_1 - X_batch_1_max).sum(dim = 1, keepdim = True)

# online batch block update max & sum
X_batch_1_max_update = torch.maximum(X_batch_0_max, X_batch_1_max) # 逐个元素找最大值
X_batch_1_sum_update = X_batch_0_sum * torch.exp(X_batch_0_max - X_batch_1_max_update) \
                                       + X_batch_1_sum * torch.exp(X_batch_1_max - X_batch_1_max_update) 

X_batch_online_softmax = torch.exp(X_batch - X_batch_1_max_update) / X_batch_1_sum_update
print(X_batch_online_softmax)
```

    tensor([[0.4256, 0.2742, 0.1525, 0.0075, 0.1221, 0.0180],
            [0.1926, 0.0404, 0.2870, 0.1012, 0.1228, 0.2560],
            [0.0881, 0.2934, 0.0264, 0.2155, 0.1964, 0.1802],
            [0.3666, 0.0882, 0.0908, 0.1541, 0.0717, 0.2286]])

<a id="0DE9F01A-CAFE-4A64-8156-1D46179C4ECB"></a>

## Objective Function

### Maximize the objective function:

$$
J(\pi_\theta) = \mathbb{E}_{\tau \sim \pi_\theta} [R(\tau)] = \sum_{\tau} R(\tau)P_{\theta}(\tau)
$$

### $P_{\theta}(\tau)$ in one trajectory under policy $\pi$: \$

We write the probability of a trajectory with MDP for $\tau = (s_0, a_0, \dots, s_T, a_T)$ :

- $\tau$: Trajectory
- $\pi_{\theta}$: Policy parameterized by $\theta$
- $P(s_0)$: Probability of the initial state
- $\pi_{\theta}(a_t | s_t)$: Probability of taking action $a_t$ in state $s_t$ (The Policy)
- $P(s_{t+1} | s_t, a_t)$: Probability of transitioning to $s_{t+1}$ given $s_t$ and $a_t$ (The Transition Function/Model)

Taking the gradient of the log-probability:

Since the initial state distribution $\rho_0$ and the environment dynamics $P(s_{t+1}|s_t, a_t)$ do not depend on the policy parameters $\theta$, their gradients are zero:

### Final Merged Expression for Loss funtion

So our loss function can be defined: in a rollout, we have N steps, each step has T possible actions

### Using advantage function $A$ replace reward $R(\theta)$ as baseline

- Action-value function: $Q_{theta}(s,a)$: the excepted reward for action $a$ under state $s$

- State-value function: $V_{\theta}(s)$: the excepted reward under state $s$

- Advantage functin: $A_{\theta}(s, a)$: under state $s$ , how much advance(better) is it to take action $a$ comparing to other actions.

#### Monte Carlo: Cover all the steps

using G to estaminate Q: A = G - V

#### Temporal Difference: only cover one step

using $Q=r+ \gamma V(s_{t+1}​)$, so $A = \delta_t^{V} = Q-V = r_t + \gamma V(s_{t+1}) - V(s_t)$

#### General Advantage Estimation(GAE) : for generall case

one step

$$
A^{1}_{\theta}(s_{t}, a) = r_{t} + \gamma \cdot V_{\theta}(s_{t+1}) - V_{\theta}(s)
$$

two steps

$$
A^{2}_{\theta}(s_{t}, a) = r_{t} + \gamma \cdot V_{\theta}(s_{t+1}) + \gamma^{2} \cdot V_{\theta}(s_{t+2}) - V_{\theta}(s)
$$

three steps

$$
A^{3}_{\theta}(s_{t}, a) = r_{t} + \gamma \cdot V_{\theta}(s_{t+1}) + \gamma^{2} \cdot V_{\theta}(s_{t+2}) + \gamma^{3} \cdot V_{\theta}(s_{t+3}) - V_{\theta}(s)
$$

T steps

$$
A^{T}_{\theta}(s_{t}, a) = r_{t} + \gamma \cdot V_{\theta}(s_{t+1}) + \gamma^{2} \cdot V_{\theta}(s_{t+2}) + \gamma^{3} \cdot V_{\theta}(s_{t+3}) + ...+ \gamma^{T} \cdot V_{\theta}(s_{t+T}) - V_{\theta}(s)
$$

Set

$$
\delta_{t}^{v} = r_{t} + \gamma \cdot V_{\theta}(s_{t+1}) - V_{\theta}(s)
$$

then t

$$
\delta_{t+1}^{v} = r_{t+1} + \gamma \cdot V_{\theta}(s_{t+2}) - V_{\theta}(s+1)
$$

$$
A^{1}_{\theta}(s_{t}, a) = \delta_{t}^{v}
$$

$$
A^{2}_{\theta}(s_{t}, a) = \delta_{t}^{v} + \gamma \delta_{t+1}^{v}
$$

$$
A^{3}_{\theta}(s_{t}, a) = \delta_{t}^{v} + \gamma \delta_{t+1}^{v} + \gamma^{2} \delta_{t+2}^{v}
$$

$$
A^{T}_{\theta}(s_{t}, a) = \delta_{t}^{v} + \gamma \delta_{t+1}^{v} + \gamma^{2} \delta_{t+2}^{v} + ... + \gamma^{T} \delta_{t+T}^{v}
$$

GAE combine those all together,

$$
A_{\theta}^{GAE}(s_t, a) = (1-\lambda)(A_{\theta}^{1} + \lambda A_{\theta}^{2} + \lambda^{2} A_{\theta}^{3} + ... + \lambda^{T} A_{\theta}^{T}) = \sum_{b=0}^{\infty } (\gamma \lambda)^{b}\delta^{b}_{t+b}
$$

$$
A_t^{GAE} = \sum_{l=0}^{\infty} (\gamma \lambda )^l \delta_{t+l}^V
$$

<a id="5B79EAED-01A9-4F1F-9AD7-B69C78A44FC4"></a>

## VPG: Vanilla Policy Gradient

$$
g = \mathbb{E}_{\tau \sim \pi_\theta} \left[ \sum_{t=0}^{T} \nabla_\theta \log \pi_\theta(a_t|s_t) \hat{A}_t \right]
$$

- **Replace**: Instead of using total Reward, we use Advantage function.
- **Advantage \$*Â*\_t\$**: Often replaced by the return $G_t$ or $Q(s,a) - V(s)$ to reduce variance.
- **Problem**: High variance and extremely sensitive to step size. One "bad" update can collapse the policy's performance.

<a id="50F66E3A-067B-4939-88CA-43C342463A36"></a>

## TRPO: Trust Region Policy Optimization

- In order to use the old data by improtant exampling, TRPO solves the stability issue by ensuring the new policy doesn't move too far from the old policy.
- introducing the ratio: $r = \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}$ and using KL Divergence as a constraint.

$$
\max_\theta \mathbb{E}_{t} \left[ \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)} \hat{A}_t \right]
$$

$$
\text{subject to } \mathbb{E}_{t} [KL(\pi_{\theta_{old}}(\cdot|s_t) || \pi_\theta(\cdot|s_t))] \leq \delta
$$

- **Replace**: we use the ratio from important exampling of current to old policy
- **Key Idea**: It defines a "Trust Region" by KL divergence to keep the update in line

<a id="E4A4F6F9-D5E9-4E99-ACD4-44D069FAAFDE"></a>

## PPO: Proximal Policy Optimization

### Critical model loss (policy model)

PPO is the industry standard for LLM fine-tuning (RLHF). It mimics TRPO's stability but uses a much simpler "clipped" objective function .

$$
L^{CLIP}(\theta) = \mathbb{E}_t \left[ \min(r_t(\theta)\hat{A}_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon)\hat{A}_t) \right]
$$

Where:

- **Probability Ratio from improtant example**: $r_t(\theta) = \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}$
- \*Epsilon $\epsilon$ cut \*: Usually set to 0.1 or 0.2.

The clipped PPO policy loss with Generalized Advantage Estimation (GAE) is defined as:

$$
\mathcal{L}_{\text{policy}} = - \mathbb{E}_{t} \Bigg[ \min \Bigg( r_t(\theta) \sum_{l=0}^{T-t} (\gamma \lambda)^l \delta_{t+l}, \; \operatorname{clip} \big( r_t(\theta), 1 - \epsilon, 1 + \epsilon \big) \sum_{l=0}^{T-t} (\gamma \lambda)^l \delta_{t+l} \Bigg) \Bigg]
$$

### Actor model Loss (value model)

use MSE, $(V-Q)^2$, where V is the output of value model and Q is the expected value of current step, we use Q = A + V to share the advantage calculation from above.

The value function (critic) is trained by minimizing the mean squared error between the predicted value and the estimated return:

$$
\mathcal{L}_{\text{value}} = \mathbb{E}_{t} \left[ \left( V_\psi(s_t) - ( A_t + V_\psi(s_t) ) \right)^2 \right]
$$

<a id="55FE69BE-E330-44AA-8A81-FA25D6D68565"></a>

## GRPO

because the GAE needs reward model and value model: $A_{\theta}(s, a) = r +  V_{\theta}(s+1) - V_{\theta}(s)$, GRPO use only reward model to normalize many generated answers

- Generating the answers: $o_1, o_2, o_3, ... o_N$
- Geting the rewards of all answers: $r_1, r_2, r_3, ... r_N$, calculate the mean $M_{r}$.
- Normalize all rewards as advantage value: $\tilde{A} = \frac{ \sum_{i}^{N} r_{i} - M_r}{std(r)}$

$$
\mathcal{J}_{GRPO}(\theta) = \mathbb{E}_{q \sim P(Q), \{o_i\}_{i=1}^G \sim \pi_{\theta_{old}}(O|q)} \left[ \frac{1}{G} \sum_{i=1}^G \sum_{t=1}^{|o_i|} \left( \min \left( \rho_{i,t}(\theta) \tilde{A}_i, \text{clip}(\rho_{i,t}(\theta), 1-\epsilon, 1+\epsilon) \tilde{A}_i \right) - \beta \mathbb{D}_{KL} \right) \right]
$$

<a id="66ec81c0-f137-4b6e-a12a-d8f2fc4303f4"></a>

## DPO

### Mathflow

在大型语言模型 (LLM) 的对齐 (Alignment) 过程中，DPO 避免了复杂的强化学习，其核心数学思想可以总结为以下极其优雅的逻辑闭环：

#### 1. 逻辑概括

- \*目标转换\*：在 RLHF 中，原始目标是最大化奖励同时受到 KL 散度惩罚。经过数学推导，这完全等价于最小化当前策略与理论最优策略 $\pi^*$ 之间的 KL 散度。
- \*反向定义奖励\*：在这个极值约束条件（KL 散度为 0）下，可以从理论最优策略的公式中，反向推导出完全由模型自身概率表达的“隐式奖励”函数。
- \*消除配分函数\*：将这个“隐式奖励”代入人类偏好模型（Bradley-Terry 模型）中，相减操作完美抵消了在真实世界中极难计算的配分函数 \$Z(x)\$。
- \*最大似然估计\*：最终，使用计算出的偏好概率进行交叉熵分类，直接更新大语言模型的参数。通过这种方式，DPO 将强化学习的“试错与探索”转变成了监督学习的“拟合完美分布”。

#### 2. 数学推导全过程

- **原始目标：定义优化方向** RLHF 的原始目标是让回答获得最大奖励 \$r(x,y)\$，同时不能偏离参考模型 $\pi_{ref}$ 太远：

  $$
  \max_{\pi_{\theta}} \mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_{\theta}(\cdot|x)} \left[ r(x, y) \right] - \beta \mathbb{D}_{KL} \left[ \pi_{\theta}(\cdot|x) || \pi_{ref}(\cdot|x) \right]
  $$

- **极值约束：推导理论最优策略 (从 Max 到 Min 的转换)** 为了找到理论最优策略，我们对原始目标函数进行代数变形。首先，将奖励项转化为对数形式并提取 \$β\$：

  $$
  \max_{\pi_\theta} \beta \, \mathbb{E}_{y \sim \pi_\theta} \left[ \log \exp\left(\frac{r(x,y)}{\beta}\right) - \log \frac{\pi_\theta(y|x)}{\pi_{ref}(y|x)} \right]
  $$

  合并对数项：

  $$
  \max_{\pi_\theta} \beta \, \mathbb{E}_{y \sim \pi_\theta} \left[ \log \frac{\pi_{ref}(y|x) \exp(\frac{r(x,y)}{\beta})}{\pi_\theta(y|x)} \right]
  $$

  为了在分子中构造一个合法的概率分布，引入归一化常数（配分函数） \$Z(x) = ∑\_y π\_{ref}(y\|x) exp ()\$，并定义理论最优策略（完美模范分布）：

  $$
  \pi^*(y|x) = \frac{1}{Z(x)} \pi_{ref}(y|x) \exp\left(\frac{r(x,y)}{\beta}\right)
  $$

  此时分子可以替换为 \$Z(x) π^\*(y\|x)\$，并将对数乘法拆解（注意是加上 \$log  Z(x)\$）：

  $$
  \max_{\pi_\theta} \beta \, \mathbb{E}_{y \sim \pi_\theta} \left[ \log \frac{\pi^*(y|x)}{\pi_\theta(y|x)} \right] + \beta \log Z(x)
  $$

  将对数内的分子分母翻转，提取负号以匹配 KL 散度的标准定义 \$𝔻\_{KL}(P\|\|Q) = 𝔼\[log (P/Q)\]\$：

  $$
  \max_{\pi_\theta} \left( -\beta \, \mathbb{D}_{KL} (\pi_\theta(y|x) || \pi^*(y|x)) + \beta \log Z(x) \right)
  $$

  最大化一个函数等价于最小化其相反数，这导致整体符号翻转（Max 视角转为 Min 视角）：

  $$
  \min_{\pi_\theta} \left( \beta \, \mathbb{D}_{KL} (\pi_\theta(y|x) || \pi^*(y|x)) - \beta \log Z(x) \right)
  $$

  由于 $Z(x)$ 不包含待优化参数 \$θ\$，它对梯度计算没有影响。因此我们证明了，原始最大化目标完全等价于：

  $$
  \min_{\pi_\theta} \mathbb{D}_{KL} (\pi_\theta(y|x) || \pi^*(y|x))
  $$

  且当 KL 散度为 0 时，得到唯一的理论最优解 \$π^\*\$。

- **变量代换：反向定义隐式奖励** 由于算不出 \$Z(x)\$，我们将 $\pi^*$ 的定义式两边取对数，反向求出奖励函数 \$r(x,y)\$：

  $$
  r(x,y) = \beta \log \frac{\pi^*(y|x)}{\pi_{ref}(y|x)} + \beta \log Z(x)
  $$

  在这里，我们将理论最优 $\pi^*$ 直接替换为我们正在训练的当前模型 \$π_θ\$，得到了完全由模型自身概率表达的隐式奖励（Implicit Reward）。

- **消除配分函数：代入人类偏好模型** 人类偏好数据符合 Bradley-Terry 模型（Sigmoid 函数 \$σ\$）：

  $$
  P(y_w \succ y_l | x) = \sigma(r(x, y_w) - r(x, y_l))
  $$

  将上一步的隐式奖励代入，相减时常数项 $\beta \log Z(x)$ 被完美抵消：

  $$
  P(y_w \succ y_l | x) = \sigma \left( \beta \log \frac{\pi_{\theta}(y_w|x)}{\pi_{ref}(y_w|x)} - \beta \log \frac{\pi_{\theta}(y_l|x)}{\pi_{ref}(y_l|x)} \right)
  $$

- **最终目标：最大似然估计与 Loss 函数** 为了让当前模型符合人类偏好，我们对上述偏好概率取负对数（最大似然估计），得到最终的 DPO 损失函数：

  $$
  \mathcal{L}_{DPO}(\pi_{\theta}; \pi_{ref}) = - \mathbb{E}_{(x, y_w, y_l)} \left[ \log \sigma \left( \beta \log \frac{\pi_{\theta}(y_w|x)}{\pi_{ref}(y_w|x)} - \beta \log \frac{\pi_{\theta}(y_l|x)}{\pi_{ref}(y_l|x)} \right) \right]
  $$

## On-Policy Distillation

## Craft import — AI — 2026-09-27

<!-- Org properties: {"craft_id": "B9ADD087-5200-4579-AD9C-0E2F6BFD911A", "imported": "2026-09-27"} -->

Source: [AI in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=B9ADD087-5200-4579-AD9C-0E2F6BFD911A)

### Tensor Core for Mixed Precision

- They load the inputs as FP16 (saving memory bandwidth).
- Inside the silicon matrix multiplier, the multiplication happens at FP16.
- Crucially, the result is instantly expanded and added to the accumulator in FP32 (full precision)
