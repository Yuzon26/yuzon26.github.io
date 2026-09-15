---
title: "1.1 Self-Attention：原理、实现与面试复习"
author: yuzon
date: 2026-09-15 12:00:00 +0800
categories: [llm, self-attention]
tags: [llm, transformer, pytorch]
description: "Q/K/V、缩放点积、掩码与多头注意力的公式和实现，以及 KV Cache、GQA、FlashAttention 的原理与区别。"
toc: true
comments: false
pin: false
math: true
published: false
---

**Self-Attention 根据输入动态计算位置间的权重，对 Value 加权聚合，形成带上下文的表示。**

记号约定：**每行一个 token**，输入 $$X$$ 可以是词嵌入或中间层隐藏状态。

## 1. Q、K、V 的作用

“这个苹果很甜”与“苹果发布了新设备”中的“苹果”含义不同。同一个 token 的初始词嵌入不区分语境，需要吸收上下文来更新表示。

如果只对每个位置单独做线性变换或 MLP，它就无法读取其他位置。Self-Attention 增加了一个按内容决定的信息聚合过程：

| 分量 | 直观含义 | 在计算中的作用 |
| --- | --- | --- |
| Query，Q | 当前接收位置需要什么信息 | 与各个 K 匹配 |
| Key，K | 某个来源位置如何被匹配 | 与 Q 共同产生分数 |
| Value，V | 该来源位置提供什么信息 | 按权重聚合 |

各头的匹配关系由训练学习，并无预设的语法或指代分工。

三个向量由同一个输入经过不同的可训练映射得到：

$$
Q=XW_Q,\qquad K=XW_K,\qquad V=XW_V.
$$

**投影矩阵是参数，Q、K、V 是随输入变化的激活。** 同一层、同一头的投影矩阵在所有位置共享。

为什么不直接使用 $$XX^{\mathsf T}$$？因为“值得交换信息”不等于“原始向量相似”。学习不同的 Q、K 投影，使匹配成为有方向的关系：

$$
q_i k_j^{\mathsf T}
=x_iW_QW_K^{\mathsf T}x_j^{\mathsf T}.
$$

中间矩阵通常不对称，所以 $$i$$ 对 $$j$$ 的关注程度不必等于反方向。K 与 V 分开，则把“如何被找到”和“提供什么内容”分开建模。

## 2. 单头注意力：公式、维度与数值例子

### 2.1 完整计算链

设序列长度为 $$n$$，模型维度为 $$d$$。忽略 batch 和偏置：

| 张量 | 形状 |
| --- | --- |
| $$X$$ | $$n\times d$$ |
| $$W_Q,W_K$$ | $$d\times d_k$$ |
| $$W_V$$ | $$d\times d_v$$ |
| $$Q,K$$ | $$n\times d_k$$ |
| $$V$$ | $$n\times d_v$$ |
| $$QK^{\mathsf T},A$$ | $$n\times n$$ |
| $$Z=AV$$ | $$n\times d_v$$ |

核心公式为：

$$
\begin{aligned}
S&=\frac{QK^{\mathsf T}}{\sqrt{d_k}}+M,\\
A&=\operatorname{softmax}(S),\\
Z&=AV.
\end{aligned}
$$

其中 $$M$$ 是加性掩码，无须屏蔽时取 0。**softmax 沿 Key 的序列维度计算，也就是每一行、代码中的最后一维。**

$$
a_{ij}=\frac{\exp(s_{ij})}{\sum_{t=1}^{n}\exp(s_{it})},
\qquad
z_i=\sum_{j=1}^{n}a_{ij}v_j.
$$

第 $$i$$ 行是位置 $$i$$ 对各 Value 的聚合权重。没有注意力 dropout 且存在有效 Key 时，每行权重和为 1；它不是下一个 token 的词表概率。

### 2.2 为什么除以平方根

在各分量独立、均值为 0、方差为 1 的简化假设下：

$$
\operatorname{Var}(q\cdot k)=d_k,\qquad
\operatorname{Var}\left(\frac{q\cdot k}{\sqrt{d_k}}\right)=1.
$$

维度增大时，未经缩放的点积更容易让 softmax 过度尖锐，使梯度变小。缩放控制的是分数的尺度，**并没有把 Q、K 归一化，因此不是余弦相似度**。公式与这一解释见[原论文第 3.2.1 节](https://arxiv.org/html/1706.03762v7#S3.SS2.SSS1)。

### 2.3 手算一次

设某个 Query 的缩放后分数为：

$$
s=[0,\ln 2,\ln 3],
\qquad
a=\operatorname{softmax}(s)=\left[\frac16,\frac13,\frac12\right].
$$

若三个来源位置的 Value 为：

$$
v_1=[6,0],\quad v_2=[0,3],\quad v_3=[2,4],
$$

那么输出为：

$$
z=\frac16[6,0]+\frac13[0,3]+\frac12[2,4]=[2,3].
$$

注意力不是挑选一个词再原样复制，而是混合多个位置的信息。Value 可以有负分量，聚合结果也不要求处处非负。

## 3. 掩码：屏蔽的是信息来源

### 3.1 Causal Mask：不能看到未来

对自回归语言模型，位置 $$i$$ 只能读取当前位置及之前的位置：

$$
M_{ij}=
\begin{cases}
0,&j\le i,\\
-\infty,&j>i.
\end{cases}
$$

三个位置的掩码为：

$$
M=
\begin{bmatrix}
0&-\infty&-\infty\\
0&0&-\infty\\
0&0&0
\end{bmatrix}.
$$

行表示 Query，列表示 Key，所以**屏蔽上三角，保留对角线**。沿用上面的分数，若第三个来源不可见，softmax 变为 $$[1/3,2/3,0]$$，输出变为 $$[2,2]$$。

为什么能看到自己？通常训练采用“当前位置预测下一个 token”的目标：

| 模型输入 | 我 | 在 | 学习 |
| --- | --- | --- | --- |
| 对应标签 | 在 | 学习 | 注意力 |

输入与标签错开一位，因此读取当前位置不会泄漏下一个 token。

训练时，完整输入已经给定，因此可以一次计算所有位置，再用掩码限制依赖。生成时，下一个输入取决于上一步采样结果，因此标准自回归解码仍需逐 token 推进。

### 3.2 Padding Mask：忽略补齐位置

Padding Mask 屏蔽批处理中为了对齐长度加入的无效 Key；Causal Mask 限制时间方向。二者可能同时存在。

几个实现陷阱：

- **在 softmax 前屏蔽。** 分数写成 0 并不代表权重为 0，因为 $$e^0=1$$。
- **避免一整行都是负无穷。** 直接 softmax 会出现未定义的归一化，常产生 NaN；左填充与因果掩码叠加时尤其需要检查。
- **屏蔽 Key 不会自动清零 padding Query 的输出。** 无效输出位置应在后续处理或损失计算中忽略。
- **布尔值语义取决于 API。** PyTorch SDPA 的布尔 mask 中 True 表示允许；nn.MultiheadAttention 的布尔 attn_mask / key_padding_mask 中 True 表示屏蔽。见 [SDPA](https://docs.pytorch.org/docs/2.8/generated/torch.nn.functional.scaled_dot_product_attention.html) 与 [MultiheadAttention 文档](https://docs.pytorch.org/docs/2.8/generated/torch.nn.MultiheadAttention.html)。

## 4. 多头注意力：分头计算，再合并信息

单个头只产生一套注意力分布。多头让不同投影子空间分别计算匹配关系，而不是把同一套权重复制多次。

标准配置常取 $$d_k=d_v=d_h=d/h$$，其中 $$h$$ 是头数。加入 batch 大小 $$B$$ 后，完整维度流如下：

| 步骤 | 形状 |
| --- | --- |
| 输入 X | $$(B,n,d)$$ |
| 合并投影得到 QKV | $$(B,n,3d)$$ |
| 拆成 Q、K、V | 各为 $$(B,n,d)$$ |
| 拆头并转置 | 各为 $$(B,h,n,d_h)$$ |
| 分数与注意力权重 | $$(B,h,n,n)$$ |
| 每个头的输出 | $$(B,h,n,d_h)$$ |
| 转置并合并头 | $$(B,n,d)$$ |
| 输出投影 | $$(B,n,d)$$ |

第 $$r$$ 个头与最终输出分别是：

$$
H_r=\operatorname{Attention}(XW_Q^{(r)},XW_K^{(r)},XW_V^{(r)}),
$$

$$
\operatorname{MHA}(X)=\operatorname{Concat}(H_1,\ldots,H_h)W_O.
$$

拼接本身只排列分量，$$W_O$$ 才进一步混合各头结果。把 $$W_O$$ 按头拆成块后，也可以写成：

$$
\operatorname{MHA}(X)=\sum_{r=1}^{h}H_rW_O^{(r)}.
$$

这解释了为什么“各头映射回模型维度后相加”与“拼接后投影”可以等价。参见[原论文多头注意力定义](https://arxiv.org/html/1706.03762v7#S3.SS2.SSS2)。

**固定 d 时，多头不会把投影参数量简单扩大 h 倍。** 忽略偏置，Q、K、V、O 四个投影合计为 $$4d^2$$。例如 $$d=768,h=12$$ 时，头维度为 64，投影参数为 2,359,296；但朴素实现的注意力权重仍有 $$Bhn^2$$ 个元素。

## 5. Transformer 中的注意力

### 5.1 Self-Attention 与 Cross-Attention

| 类型 | Q 来源 | K、V 来源 | 注意力矩阵 |
| --- | --- | --- | --- |
| Self-Attention | 当前序列 | 同一序列 | $$n\times n$$ |
| Cross-Attention | 目标序列 | 另一组表示 | $$n_q\times n_k$$ |

self 指信息来源相同，不是只能关注自己。Cross-Attention 的输出长度跟随 Query，Key 与 Value 的序列长度必须一致；例如翻译解码器以自身状态为 Q，以编码器输出为 K、V。

### 5.2 位置、残差、归一化和 FFN

**不带位置相关信息、也不带方向性掩码的自注意力具有置换等变性**：重排输入，只会相应重排输出，无法单靠它区分词序。位置编码等机制用于提供顺序信息。

Attention 只负责跨位置的信息混合。Transformer 还需要残差、归一化和逐位置 FFN。以一种 Pre-LN 写法为例，省略 dropout：

$$
U=X+\operatorname{MHA}(\operatorname{LN}(X)),
$$

$$
Y=U+\operatorname{FFN}(\operatorname{LN}(U)).
$$

FFN 在各位置共享参数，先升维、经过非线性再降维，加工注意力汇入的上下文。**Pre-LN 在子层前归一化；原论文的 Post-LN 在残差相加后归一化。**

## 6. PyTorch 实现

实现支持标准 MHA 和完整序列的因果掩码，输入为**无 padding 的浮点张量**。位置编码、残差、归一化、dropout 和 KV Cache 不包含在此模块中。

`qkv` 将三个投影合并为一个线性层。PyTorch `Linear` 计算 $$xW^{\mathsf T}+b$$，权重存储方向与公式中的右乘矩阵相反。

```python
import torch
from torch import nn
from torch.nn import functional as F


class SelfAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        if d_model <= 0 or num_heads <= 0 or d_model % num_heads:
            raise ValueError("d_model must be positive and divisible by num_heads")
        self.num_heads = num_heads
        self.head_dim = d_model // num_heads
        self.qkv = nn.Linear(d_model, 3 * d_model, bias=False)
        self.out = nn.Linear(d_model, d_model, bias=False)

    def forward(self, x, causal=False):
        if x.ndim != 3 or x.size(1) == 0:
            raise ValueError("expected nonempty x with shape (B, N, D)")
        b, n, d = x.shape
        q, k, v = self.qkv(x).chunk(3, dim=-1)
        q, k, v = [
            t.reshape(b, n, self.num_heads, self.head_dim).transpose(1, 2)
            for t in (q, k, v)
        ]  # (B, H, N, Dh)

        scores = (q @ k.transpose(-2, -1)) / self.head_dim**0.5
        if causal:
            blocked = torch.ones(n, n, dtype=torch.bool, device=x.device).triu(1)
            scores = scores.masked_fill(blocked, float("-inf"))
        weights = scores.softmax(dim=-1)
        z = weights @ v
        z = z.transpose(1, 2).contiguous().view(b, n, d)
        return self.out(z)


def demo():
    torch.manual_seed(0)
    model = SelfAttention(32, 4)
    x = torch.randn(2, 5, 32, requires_grad=True)
    q, k, v = [
        t.reshape(2, 5, 4, 8).transpose(1, 2)
        for t in model.qkv(x).chunk(3, dim=-1)
    ]

    # Check both mask modes against the official attention operator.
    for causal in (False, True):
        ref = F.scaled_dot_product_attention(
            q, k, v, dropout_p=0.0, is_causal=causal
        )
        ref = model.out(ref.transpose(1, 2).contiguous().view(2, 5, 32))
        torch.testing.assert_close(model(x, causal), ref, atol=1e-6, rtol=1e-5)

    # Changing future tokens must not affect earlier outputs.
    changed = x.detach().clone()
    changed[:, 3:] += 10 * torch.randn_like(changed[:, 3:])
    y = model(x, causal=True)
    torch.testing.assert_close(y[:, :3], model(changed, True)[:, :3])

    y.square().mean().backward()
    assert x.grad is not None and torch.isfinite(x.grad).all()
    for p in model.parameters():
        assert p.grad is not None and torch.isfinite(p.grad).all()
    print("attention checks passed")


if __name__ == "__main__":
    demo()
```

**实现要点：** softmax 沿最后一维；K 转置最后两维；因果掩码屏蔽上三角；合并头前恢复为 $$(B,n,h,d_h)$$ 的轴顺序。

工程中可用 `F.scaled_dot_product_attention` 替换分数计算、掩码、softmax 和 Value 聚合，由框架选择可用实现。**QKV 与输出投影仍需单独完成**；推理时显式设置 `dropout_p=0.0`。上述手写版本会显式存储注意力矩阵。

## 7. 复杂度与推理：为什么长上下文贵

### 7.1 计算量与显存

在标准 MHA、$$hd_h=d$$ 下，单层主要成本是：

| 部分 | 计算量级 |
| --- | --- |
| QKV 与输出投影 | $$O(Bnd^2)$$ |
| QK 转置乘法与 AV | $$O(Bn^2d)$$ |
| 朴素实现显式存储注意力权重 | $$O(Bhn^2)$$ |

总计算量级为 $$O(Bnd^2+Bn^2d)$$，未计 FFN。固定模型宽度和头数，序列长度翻倍，投影部分约翻倍，注意力核心约四倍；不能据此断言整个模型耗时必然四倍。

例如 $$B=1,h=12,n=4096$$，单个 FP16 注意力矩阵就有 $$12\times4096^2$$ 个元素，占 **384 MiB**，还没有计入其他激活、梯度或临时张量。这是显式保存矩阵的估算，不代表所有实现都需要这块存储。

### 7.2 KV Cache：复用历史计算

对固定前缀的因果模型，在确定性推理中，新 token 不会改变旧位置的表示，因此每一层已经计算出的 K、V 可以缓存。

解码新位置时，计算新 Q/K/V，把新 K、V 追加到缓存，再让新 Q 读取全部可见 K、V。历史 Q 不需要参与新位置的计算，所以通常不缓存 Q。

Prefill 处理整段输入；逐 token Decode 的 Query 长度通常为 1。缓存消除了旧位置投影及其层内处理的重复计算，但新 Query 仍需读取历史 K、V，标准稠密注意力的单步核心计算仍随上下文长度增长。见 [Hugging Face 缓存说明](https://huggingface.co/docs/transformers/main/cache_explanation)。

缓存显存的基本估算为：

$$
\text{KV bytes}\approx 2LBn h_{kv}d_hs,
$$

其中 $$L$$ 为层数，$$h_{kv}$$ 为 KV 头数，$$s$$ 为每个元素的字节数，2 对应 K 和 V。此式不包含缓存分配器、分页和对齐开销。

**带缓存时不能照搬方形掩码。** Query 与 Key 长度不同，允许关系应按绝对位置确定。对于一次只处理一个新 token、且缓存中仅含历史和当前位置的情形，所有缓存 Key 都可见；某些算子的非方形 is_causal 对齐规则可能不符合这种需求。

### 7.3 MHA、MQA、GQA 与 FlashAttention

| 名称 | 核心变化 | 主要目的 |
| --- | --- | --- |
| MHA | 各 Query 头有对应的 K、V 头 | 标准多头注意力 |
| MQA | 所有 Query 头共享一组 K、V | 减小 KV 缓存与读取量 |
| GQA | 每组 Query 头共享一组 K、V | 在共享程度与模型质量间折中 |
| FlashAttention | 分块计算、减少显存读写，避免完整存储注意力矩阵 | 提高注意力计算的效率 |

MHA 中 $$h_{kv}=h$$，MQA 中 $$h_{kv}=1$$，通常所说的中间型 GQA 满足 $$1<h_{kv}<h$$。GQA 改变了头的共享结构，不能把一个已训练 MHA 模型随意删掉 K、V 头后假定效果不变。见 [GQA 论文](https://arxiv.org/abs/2305.13245)。

FlashAttention 保持精确稠密注意力的数学目标，允许正常的浮点数值差异；它不是稀疏近似，也没有把一般稠密注意力的算术复杂度改成线性。KV Cache、GQA 和 FlashAttention 处理不同问题，可以组合使用。见 [FlashAttention 论文](https://arxiv.org/abs/2205.14135)。

## 8. 面试前自测

| 问题 | 回答要点 |
| --- | --- |
| 30 秒解释 Self-Attention？ | 对输入做 QKV 投影，QK 点积缩放并加掩码，softmax 得到权重，再对 V 聚合；多头独立计算后拼接投影。 |
| Q、K 维度必须相同吗？V 呢？ | 点积要求 Q、K 的每头维度相同；V 的维度可不同，但 K、V 的序列长度要一致。 |
| 注意力矩阵对称吗？ | 通常不对称；Q、K 投影不同，逐行 softmax 与掩码也会影响对称性。 |
| 为什么多头不是多做 h 倍计算？ | 标准配置固定总维度 d，每头维度缩为 d/h；但头数会影响显式权重存储和实际执行开销。 |
| Self-Attention 为什么需要位置信息？ | 无位置与方向性约束时具有置换等变性，无法仅凭内容确定顺序。 |
| 训练可并行，为什么生成仍逐步进行？ | 训练输入已知；生成的新输入依赖前一步输出，因果掩码本身不会消除该依赖。 |
| KV Cache 缓存什么？为什么不缓存 Q？ | 每层历史 K、V；新位置只需要新 Q 与历史 K、V，不使用历史 Q。 |
| 注意力权重能直接解释模型结论吗？ | 只能观察一次信息加权，还受到 V、输出投影、残差和后续层影响，不能直接当成完整因果解释。 |

## 参考资料

- [Attention Is All You Need](https://arxiv.org/pdf/1706.03762)：重点看 3.2 节注意力、3.5 节位置编码与第 4 节复杂度。
- [3Blue1Brown 官方图文稿](https://www.3blue1brown.com/lessons/attention/)：上下文更新与 Q/K/V。
- [李宏毅 Self-Attention 课件](https://speech.ee.ntu.edu.tw/~hylee/ml/ml2021-course-data/self_v7.pdf)：矩阵计算与位置编码。
- [PyTorch SDPA](https://docs.pytorch.org/docs/2.8/generated/torch.nn.functional.scaled_dot_product_attention.html) 与 [MultiheadAttention](https://docs.pytorch.org/docs/2.8/generated/torch.nn.MultiheadAttention.html)：张量维度与掩码约定。
- [Hugging Face KV Cache](https://huggingface.co/docs/transformers/main/cache_explanation)、[GQA](https://arxiv.org/abs/2305.13245)、[FlashAttention](https://arxiv.org/abs/2205.14135)。

- 入门视频：3Blue1Brown [直观解释注意力机制，Transformer 的核心](https://www.bilibili.com/video/BV1TZ421j7Ke/)。
- 知乎专栏：[动图轻松理解 Self-Attention](https://zhuanlan.zhihu.com/p/619154409)。
- Attention 经典论文：[视频讲解](https://www.bilibili.com/video/BV1xoJwzDESD/)。
- 李宏毅老师讲解自注意力：[视频讲解](https://www.bilibili.com/video/BV1L142187HH?p=2)。
- 手绘图解 Transformer 代码：[图文讲解](https://zhuanlan.zhihu.com/p/366592542)。
