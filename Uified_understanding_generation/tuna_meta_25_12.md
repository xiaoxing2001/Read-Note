# Tuna: Taming Unified Visual Representations for Native Unified Multimodal Models

**arXiv**: [2512.02014](https://arxiv.org/abs/2512.02014)

---

## Motivation

当前的统一多模态模型（UMMs）中，以 **Show-o2** 为代表的方法采用 **并行架构**（late-fusion），即分别使用语义编码器（如 SigLIP）与生成编码器（如 VAE）提取特征后再融合，容易导致表示格式不一致、任务偏好失衡等问题。

TUNA 提出一种 **级联架构**（cascaded design）：将 VAE 编码器的输出作为表示编码器（如 SigLIP-2）的输入，构建**统一的连续视觉表示空间**。该设计天然避免了语义与生成路径之间的格式错配，同时在理解与生成任务上实现更均衡的性能。大量实验与消融分析验证了该方案的优越性。

---

## Method Highlights

### 1. 模型架构

- **视觉输入**（图像或视频）首先通过 **Wan 2.2 的 3D 因果 VAE 编码器**，得到压缩后的连续隐变量（latent）；
- 该 latent 被送入 **修改后的 SigLIP-2 表示编码器**（替换其原始 patch embedding 为 1×1 卷积以对齐 token 长度）；
- 经过一个轻量 MLP 投影后，得到**统一的视觉表示**；
- 该表示与文本 token 拼接后，输入同一个 **LLM 解码器**（基于 Qwen-2.5）；
- **输出头分两路**：
  - **语言生成**：标准语言建模头（LM head）用于理解类任务（如 VQA、caption）；
  - **视觉生成**：一个共享 LLM 架构的 **流匹配头**（flow matching head）配合 VAE 解码器，用于图像/视频生成与编辑。

> 注：在生成任务中，视觉 token 会注入噪声（flow matching 所需），而在理解任务中使用干净 latent（t = 1）。

### 2. 三阶段训练策略

1. **阶段一（Representation + Flow Head Pretraining）**  
   - 冻结 LLM，仅训练 **表示编码器**（SigLIP-2） 与 **流匹配头**；
   - 使用 **图像 captioning** 与 **文本到图像生成**（T2I） 作为训练目标；
   - 目的：对齐语义理解与生成能力，初始化高质量的统一表示。

2. **阶段二（Full Model Continue Pretraining）**  
   - 解冻 LLM，端到端训练整个模型；
   - 扩展数据集：加入 **图像指令跟随、图像编辑、视频 caption** 等多任务数据；
   - 提升模型在复杂指令与跨模态推理上的能力。

3. **阶段三（Supervised Fine-Tuning, SFT）**  
   - 使用高质量的 **图像/视频指令数据、编辑数据、生成数据**；
   - 采用较小学习率（2e-5）进行精细化微调；
   - 进一步提升模型在实际应用场景中的表现与鲁棒性。

---

## Experiments

### 1. 主要结果

- 在 **1.5B 和 7B 两个规模** 上，TUNA 在以下任务均达到 **SOTA**：
  - **多模态理解**：MME、GQA、RealWorldQA、MMStar、ChartQA 等 9 个 benchmark；
  - **图像生成**：GenEval、DPG-Bench、OneIG-Bench；
  - **图像编辑**：ImgEdit-Bench、GEdit-Bench；
  - **视频理解**：MVBench、Video-MME 等；
  - **视频生成**：VBench（仅 1.5B 版本支持，7B 因计算成本未训练视频）。

> 注：7B 模型未使用视频数据训练，因此不支持视频生成，但在图像与理解任务上全面领先。

### 2. 消融实验关键发现

1. **级联架构 > 并行融合**（Show-o2）  
   TUNA 的级联设计在所有任务上均优于 Show-o2 的 late-fusion 方案。

2. **更强的表示编码器带来一致增益**  
   SigLIP-2（400M）与 DINOv3（800M）均优于早期 SigLIP（400M）；最终选用 SigLIP-2 是因**生成质量更优 + 模型更小**。

3. **理解与生成任务互相促进**  
   联合训练（理解 + 生成）的模型在两类任务上均优于**仅用单一任务数据训练**的模型，验证了**协同增强**（mutual enhancement）效应。

### 3. 表示分析（CKNNA）

- 通过 CKNNA（Compatibility of Kernel Neighborhoods via Alignment）分析发现：
  - **Show-o2 的统一表示严重偏向语义理解**（与 SigLIP-2 高对齐，但与 SD3-Medium 生成模型对齐弱）；
  - **TUNA 的表示在理解与生成之间更均衡**，与两个参考模型均有较高对齐；
- 原因：Show-o2 的 late-fusion 导致生成分支信号被压制（CKNNA 生成分支仅 0.07），而 TUNA 的级联设计实现**逐层融合**，保留丰富生成信息。

---

## 其他思考

- **代码与模型未开源**（截至论文发布时）；
- 官方项目页：[https://tuna-ai.org](https://tuna-ai.org)（可关注后续更新）；
- 该工作为“原生统一多模态模型”（native UMM）提供了一种高效、简洁且可扩展的范式，有望成为下一代多模态基础模型的重要方向。

---