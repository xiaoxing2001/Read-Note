
---
# EMMA: Efficient Multimodal Understanding, Generation, and Editing with a Unified Architecture

**arXiv**: [2512.04810](https://arxiv.org/abs/2512.04810)
---
## Motivation
EMMA 的核心目标是构建一个**高效的统一多模态架构**，能够同时胜任理解（understanding）、生成（generation）和编辑（editing）三大任务。不同于仅在任务层面进行统一（如通过桥接机制连接预训练视觉编码器与生成解码器），EMMA 从**架构层面**出发，重点解决统一模型中理解分支与生成分支的特征融合与任务解耦问题。

具体而言：
- EMMA 借鉴了 BAGEL、Janus 等工作的思路，但提出在**相同压缩率（32×）**下，对语义编码器（Und-Enc）和生成编码器（Gen-Enc）提取的视觉 token 进行**通道级拼接（channel-wise concatenation）**，而非传统的 token 级拼接。这有效减少了视觉 token 数量（例如在图像编辑中减少 5 倍），同时保留语义与细节信息。
- 在网络结构上，采用**共享-解耦（shared-and-decoupled）设计**：浅层参数（如 Q/K 投影）共享以促进跨任务知识迁移，深层参数（如 V 投影）解耦以满足任务特异性建模需求（理解侧重语义，生成还需高频细节）。

---

## Method Highlights

### 1. 模型架构

- **视觉特征融合**：语义编码器（SigLIP2）与生成编码器（DCAE）均采用 **32× 压缩率**，使得两者输出的 token 序列长度一致。EMMA 在**通道维度**拼接这两组 token（而非拼接 token 序列），从而在不增加 token 数的前提下融合语义与细节信息。
- **Adapter 机制**：拼接后的视觉 token 分别通过**理解 Adapter** 和 **生成 Adapter** 映射到统一 LLM（基于 Qwen3-4B）的输入空间。
- **共享-解耦网络**：
  - 浅层：共享注意力中的 Q/K 投影权重，促进跨任务协同；
  - 深层：V 投影及后续 FFN 层任务解耦，适应不同建模目标。
- **MoE 视觉编码器**：在 SigLIP2 基础上引入 **Mixture-of-Experts（MoE）**，包含一个通用专家（Versatile Expert）和一个 STEM 专家（STEM Expert）。Router 动态选择专家，显著提升对图表、公式等 STEM 图像的理解能力（仅增加 ~50M 参数）。
- **多任务输出头**：
  - 理解任务：采用**下一 token 预测**（next-token prediction）；
  - 生成/编辑任务：采用**流匹配（flow matching） + 速度预测（velocity prediction）**。

---

### 2. 数据构建

#### 多模态理解数据（I2T）
- **对齐数据**：LLaVA-558K
- **预训练（PT）**：LAION + 内部图文对，使用 re-captioning 模型重写描述
- **SFT**：LLaVA-OneVision、FineVision 等开源高质量 VQA 数据 + 内部构建数据（涵盖文档、图表、OCR、数学等）
- **高质量 SFT（QT）**：从 SFT 中筛选高分样本，按任务均衡采样
- **STEM 专家微调（ET）**：15M STEM 相关样本
- **Router 微调（RT）**：3M 混合样本（STEM + 通用）

#### 文生图数据（T2I）
- **PT**：LAION + 内部数据，按美学质量过滤
- **SFT/QT**：高分辨率（≥1K）、高美学分图像，**平衡通用图像与人像**；为缓解文本渲染数据稀缺，**合成部分带文字图像**

#### 图像编辑数据（IT2I）
- 利用开源数据集（如 X2I2、OmniEdit）
- **自建合成数据**：使用 Qwen-Image-Edit 生成编辑对，并通过 VLM 和人脸相似度进行过滤（见图 3、4）
- **排除 GPT-Image-Edit-1.5M**：因其破坏主体一致性，可能误导 GEdit 评估

---

### 3. 训练策略（五阶段）

| 阶段 | 目标 | 训练内容 | 关键设置 |
|------|------|--------|--------|
| **Stage 0** | 对齐 | 仅训练理解分支 Adapter | 冻结视觉编码器和 LLM |
| **Stage 1** | 预训练 | 除 DCAE 外全参数训练 | 图像 512×512，理解/生成 batch 1:1 |
| **Stage 2** | SFT | 除 DCAE 外全参数训练 | 理解支持原生分辨率，生成支持 1K 多尺度；后期加入编辑数据，三任务 1:1:1 混合 |
| **Stage 3** | 质量微调（QT） | 三任务联合训练 | LR = 1e⁻⁵ |
| **Stage 4** | MoE 微调 | 仅训练 STEM Expert 或 Router | 冻结其余参数 |

> 注：DCAE（生成用自编码器）在所有阶段均**保持冻结**，因其已在生成模型中预训练好。

---

## Experiments

### 1. 主要结果

EMMA-4B（基于 Qwen3-4B）在三大任务上均达到 SOTA 或极具竞争力：

- **多模态理解**（11 个 benchmark）：
  - MMBench、MMMU、MMVet 等
  - **超越 BAGEL-7B**（如 MMVet: 73.0 vs 67.2）
  - **媲美专用模型**：优于 InternVL3.5（+2.6% avg）、略优于 Qwen3-VL（+0.4% avg）

- **文生图生成**：
  - **GenEval**: 0.91（无 prompt rewriting / RL），超过 Qwen-Image（0.87, 20B）和 BAGEL-7B（0.82）
  - **DPG-Bench**: 85.63，领先所有统一模型

- **图像编辑**：
  - **GEdit-Bench-EN**: 6.53，略优于 BAGEL-7B（6.52）
  - **关键优势**：仅需 **1/5 视觉 token** 即可达到更好或相当效果

### 2. 新兴能力（Emergent Capabilities）

- **零样本支持中文指令**：尽管训练中未使用中文 T2I/编辑数据，但因理解分支接触过中文图文对，模型能正确解析并执行中文编辑指令。
- **复杂编辑指令泛化**：训练数据均为单步指令，但模型能正确执行多步复合指令（如“换衣服+加墨镜+换背景”），归因于多跳推理（chain-of-thought）数据的训练效应。

---

## 总结

EMMA 通过 **高倍压缩自编码器 + 通道级特征融合 + 共享-解耦架构 + MoE 视觉编码器**，在**更低参数量（4B）和更少视觉 token**下，实现了对多模态统一模型的显著突破。其设计为未来高效、通用的多模态基础模型提供了重要范式。
--- 