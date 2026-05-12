---
layout: post
title: "A Guide to LLM Architecture: From First Principles to Modern Efficiency"
date: 2025-08-01
image: "/assets/llm-architecture/architecture.png"
excerpt: "A deep, technical dive into the architectural evolution of decoder-based language models."
author: "Satish Yenumula"
tags: [Architecture]
---

## Introduction

As AI engineers, our progress is built on a deep, foundational understanding of model architecture. While the "Attention Is All You Need" paper (Vaswani et al., 2017) provided the blueprint, the journey from early transformers to today's state-of-the-art systems is not a simple story of scaling. It is a narrative of relentless optimization, where each new architectural refinement addresses a fundamental bottleneck, often by returning to first principles.

This image is a perfect metaphor for our journey: we'll start with the simple inputs and follow the path through the complex internals to understand how these amazing models truly work.

This article provides a deep-dive into this evolution. We will move beyond a simple catalog of changes and instead explore the core problems that drove innovation. We will ask *why* these changes were necessary, how they work, and how they culminated in the highly efficient and powerful models we see today, from the Llama series to Mistral's Mixture-of-Experts.

## Chapter 1: Scaling and Stability

The initial challenge for language models was straightforward: how to effectively scale and train deeper networks. The GPT series by OpenAI laid the groundwork.

- **GPT-1 (2018)** established the viability of the decoder-only, generative pre-training approach (Radford et al., 2018).
- **GPT-2 (2019)** demonstrated that scaling up parameters led to surprising new capabilities. Crucially, it also introduced **pre-normalization** to stabilize training in deeper networks (Radford et al., 2019).
- **GPT-3 (2020)** showed the power of immense scale, but its training required specialized techniques like sparse attention to remain computationally feasible (Brown et al., 2020).

This era highlighted a fundamental tension: scaling unlocks power, but also introduces instability and computational barriers.

### First Principles: Why is Training Deep Networks Hard?

At its core, training a neural network involves passing gradients backward through its layers. In a very deep network, these gradients can either shrink exponentially toward zero (vanishing gradients) or grow uncontrollably (exploding gradients). This makes it impossible for the model to learn effectively.

**Layer Normalization**, and specifically the **pre-normalization** variant used in GPT-2, is a direct solution. By normalizing the inputs to each transformer block, we ensure that the activations remain well-behaved, preventing the gradient signal from dying or exploding. This simple principle of maintaining stability is a non-negotiable prerequisite for building massive models.

## Chapter 2: The Quest for Efficiency - Doing More with Less

After GPT-3, the focus shifted from pure scaling to "smarter" architectures. The new goal was to achieve better performance with less computational cost, particularly during inference.

### 2.1 Rethinking Attention

**First Principles: The Quadratic Bottleneck.** The power of self-attention comes from its ability to relate every token to every other token. However, this means the computational and memory cost grows quadratically with the sequence length (O(n²)). During inference, the largest memory consumer is the Key-Value (KV) cache, which stores the key and value vectors for every token. For a model with 32 attention heads, this means storing 32 sets of K and V vectors.

To solve this, researchers developed more efficient attention variants:

- **Multi-Query Attention (MQA):** Proposed by Shazeer (2019), MQA makes a radical simplification: all query heads share a *single* Key and Value head. This reduces the KV cache size by a factor equal to the number of heads (e.g., 32x), dramatically speeding up inference. The trade-off is a potential loss in model quality.
- **Grouped-Query Attention (GQA):** As a brilliant compromise, GQA (Ainslie et al., 2023) groups query heads, with each group sharing one K/V head. An 8-group GQA, for instance, reduces the KV cache size by 8x. This provides most of MQA's speed benefits while retaining nearly all of the quality of standard multi-head attention, making it the new standard in models like Llama 2 and Mistral.

### 2.2 Optimizing the Feed-Forward Network (FFN)

**First Principles: Where is the Computational Bulk?** While attention is conceptually complex, the simple Feed-Forward Network (FFN) inside each transformer block contains the vast majority of the model's parameters. Making these FFNs more efficient is critical.

- **SwiGLU:** Early models used standard ReLU or GELU activation functions. However, research by Shazeer (2020) showed that GLU (Gated Linear Unit) variants, specifically **SwiGLU**, perform significantly better. SwiGLU uses a gating mechanism to control the information flow through the FFN, allowing the network to learn more complex relationships. It has become the de facto standard.

```python
import torch.nn as nn
import torch.nn.functional as F

class SwiGLU(nn.Module):
    def __init__(self, in_dim: int, hidden_dim: int, out_dim: int):
        super().__init__()
        self.w1 = nn.Linear(in_dim, hidden_dim, bias=False)
        self.w2 = nn.Linear(in_dim, hidden_dim, bias=False)
        self.w3 = nn.Linear(hidden_dim, out_dim, bias=False)

    def forward(self, x):
        return self.w3(F.silu(self.w1(x)) * self.w2(x))
```

- **Mixture-of-Experts (MoE):** The MoE paradigm (Shazeer et al., 2017) is the most significant leap in FFN efficiency.
    - **First Principles: Sparse Activation.** Instead of using one large FFN for every token, what if we had many smaller "expert" FFNs and dynamically chose which one(s) to use for each token? This is the core idea of a **Sparse Mixture-of-Experts (SMoE)** layer. A "router" network selects a small subset of experts (e.g., 2 out of 8) for each token.
    - **The Result:** The model can have a massive number of total parameters (e.g., Mixtral 8x7B has ~47B), but only a fraction are activated for any given token (~13B). This leads to dramatically faster inference compared to a dense model of similar size, as seen in the success of Mixtral (Jiang et al., 2024).

### 2.3 Streamlining Normalization

**First Principles: Can We Simplify?** We know normalization is crucial for stability. But is the standard LayerNorm doing more work than necessary?
- **RMSNorm (Root Mean Square Normalization):** Zhang and Sennrich (2019) found that the mean-centering operation in LayerNorm was responsible for very little of the performance gain. By removing it and only normalizing by the root-mean-square of the activations, **RMSNorm** achieves the same stability with 7-64% less computation. This efficiency gain, with no quality loss, has made it a standard component in Llama, Mistral, and other modern models.

```python
import torch
import torch.nn as nn

class RMSNorm(nn.Module):
    def __init__(self, dims: int, eps: float = 1e-6):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(dims))
        self.eps = eps

    def _norm(self, x):
        return x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)

    def forward(self, x):
        return self._norm(x.float()).type_as(x) * self.weight
```

## Chapter 3: Better Representation

Beyond efficiency, another line of innovation focused on representing information more effectively within the model.

### 3.1 Rethinking Positional Information

**First Principles: How Does a Model Perceive Order?** The transformer architecture is inherently parallel and permutation-invariant; it doesn't know the order of tokens. The initial solution was to add a learned "positional embedding" to each token. However, this approach struggles to generalize to sequence lengths not seen during training.

- **Rotary Positional Embeddings (RoPE):** Introduced by Su et al. (2021), RoPE offers a more elegant solution. Instead of adding an embedding, it *rotates* the query and key vectors based on their absolute position. This naturally injects relative positional information into the self-attention calculation (e.g., the relationship between token 5 and token 7 is the same as between token 12 and token 14). This method has shown superior performance and is a cornerstone of modern LLMs.

### 3.2 Stabilizing Attention Dynamics

**First Principles: Well-Behaved Vectors.** The dot product between query and key vectors can result in very large or small values, leading to unstable training.
- **QK-Norm:** To address this, some recent models like Gemma (Gemma Team, 2024) apply an additional RMSNorm to the query (q) and key (k) vectors *before* the attention calculation. This ensures the vectors are well-scaled, further improving training stability.

## Chapter 4: Theory to Practice 

These individual innovations converged to create the modern LLM architecture, exemplified by the leading open-source models.

- **The Llama Blueprint:** Meta's Llama (Touvron et al., 2023) was a landmark, popularizing the now-standard combination of **RMSNorm, SwiGLU, and RoPE**.
- **Efficiency at Scale:** Llama 2 (Touvron et al., 2023) integrated **Grouped-Query Attention (GQA)**, allowing its 70B model to be served far more efficiently.
- **Pushing Forward:** Mistral AI's models demonstrated the raw power of these new architectures. **Mistral 7B** (Jiang et al., 2023) used GQA and Sliding Window Attention to outperform models twice its size. **Mixtral 8x7B** (Jiang et al., 2024) then used a **Mixture-of-Experts** layer to achieve top-tier performance while using only a fraction of its parameters at inference time, setting a new standard for computational efficiency.

This convergence on a shared set of architectural tenets (RMSNorm, SwiGLU, RoPE, GQA) underscores their proven value. The frontier of competition is now shifting towards data quality, training techniques, and post-training alignment.

## Conclusion

The evolution of LLM architecture is a powerful illustration of scientific progress. It's a journey from brute-force scaling to intelligent, efficient design, driven by a commitment to first principles. By repeatedly asking "why" and tackling fundamental bottlenecks in stability, computation, and representation, researchers have developed models that are not only more powerful but also more accessible than ever before. As we look to the future, this relentless quest for principled, efficient design will continue to shape the landscape of artificial intelligence.

## References

1.  Vaswani, A., et al. (2017). Attention Is All You Need. *arXiv:1706.03762*.
2.  Radford, A., et al. (2018). Improving Language Understanding by Generative Pre-Training. *OpenAI Blog*.
3.  Radford, A., et al. (2019). Language Models are Unsupervised Multitask Learners. *OpenAI Blog*.
4.  Brown, T., et al. (2020). Language Models are Few-Shot Learners. *arXiv:2005.14165*.
5.  Zhang, B., & Sennrich, R. (2019). Root Mean Square Layer Normalization. *arXiv:1910.07467*.
6.  Shazeer, N. (2020). GLU Variants Improve Transformer. *arXiv:2002.05202*.
7.  Su, J., et al. (2021). RoFormer: Enhanced Transformer with Rotary Position Embedding. *arXiv:2104.09864*.
8.  Shazeer, N. (2019). Fast Transformer Decoding: One Write-Head is All You Need. *arXiv:1911.02150*.
9.  Ainslie, J., et al. (2023). GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints. *arXiv:2305.13245*.
10. Shazeer, N., et al. (2017). Outrageously Large Neural Networks: The Sparsely-Gated Mixture-of-Experts Layer. *arXiv:1701.06538*.
11. Touvron, H., et al. (2023). LLaMA: Open and Efficient Foundation Language Models. *arXiv:2302.13971*.
12. Touvron, H., et al. (2023). Llama 2: Open Foundation and Fine-Tuned Chat Models. *arXiv:2307.09288*.
13. Jiang, A. Q., et al. (2023). Mistral 7B. *arXiv:2310.06825*.
14. Jiang, A. Q., et al. (2024). Mixtral of Experts. *arXiv:2401.04088*.
15. Gemma Team, Google. (2024). Gemma: Open Models Based on Gemini Research and Technology. *arXiv:2403.08295*.