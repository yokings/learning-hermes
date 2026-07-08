# 01 - Weight-only 量化（仅权重量化）
> Activation保持FP16/BF16，仅压缩权重，是目前工业界最成熟、部署最广泛的方案
> W4A16是目前的主流，精度损失极小

---

## 经典基础工作

### 1. GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers
- **会议**：ICLR 2023
- **作者**：Elias Frantar et al. (IST Austria)
- **核心贡献**：
  - 提出基于OBQ（Optimal Brain Quantization）的近似二阶信息量化方法
  - W4/3bit量化，精度接近FP16，速度快
  - 首个真正能在LLM上工作的PTQ方法
  - 开源：AutoGPTQ（最流行的W4量化实现之一）
- **链接**：https://arxiv.org/abs/2210.17323
- **代码**：https://github.com/IST-DASLab/gptq

### 2. AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration
- **会议**：MLSys 2024
- **作者**：Ji Lin et al. (MIT)
- **核心贡献**：
  - 发现权重中只有1%的"salient weights"（对应大activation的权重）对量化精度影响大
  - 保护/缩放这些salient weights，其他正常量化
  - W4A16精度优于GPTQ，校准更快（不需要Hessian）
  - 开源：AutoAWQ，支持vLLM/SGLang加速
- **链接**：https://arxiv.org/abs/2306.00978
- **代码**：https://github.com/mit-han-lab/llm-awq

### 3. QLoRA: Efficient Finetuning of Quantized LLMs
- **会议**：NeurIPS 2023
- **作者**：Tim Dettmers et al. (University of Washington)
- **核心贡献**：
  - 4bit NormalFloat（NF4）数据类型，针对正态分布权重优化
  - Double Quantization：量化常数本身也量化，进一步省显存
  - Paged Optimizers：类似CPU内存分页，处理梯度checkpointing显存峰值
  - 单卡4090就能微调65B模型！
  - 虽然主要讲微调，但NF4量化本身被广泛使用
- **链接**：https://arxiv.org/abs/2305.14314
- **代码**：https://github.com/artidoro/qlora

---

## 2024-2025 最新SOTA

### 4. QuIP#: Even Better LLM Quantization with Hadamard Incoherence and Lattice Codebooks
- **会议**：ICML 2024
- **作者**：Albert Tseng et al. (Cornell)
- **核心贡献**：
  - 在QuIP基础上改进，使用**Hadamard变换**让权重分布均匀incoherent
  - **格点码本（Lattice Codebook）**：E8 lattice等，比k-means码本更高效
  - W2A16/W4A16/W4A4都达到当时SOTA，W2位宽下保持很高精度
  - 缺点：Hadamard变换带来额外推理开销
- **链接**：https://arxiv.org/abs/2402.04396
- **代码**：https://github.com/facebookresearch/quip-sharp

### 5. AQLM: Extreme Compression of Large Language Models via Additive Quantization
- **会议**：ICML 2024
- **作者**：Vage Egiazarian et al. (Yandex)
- **核心贡献**：
  - 将向量量化中的**加性量化（Additive Quantization）**引入LLM
  - 多个码本加和表示权重，W2/1bit精度显著优于GPTQ/AWQ
  - 支持GPU推理加速
- **链接**：https://arxiv.org/abs/2401.06118
- **代码**：https://github.com/Vahe1994/AQLM

### 6. Atom: Low-bit Quantization for Efficient and Accurate LLM Serving
- **会议**：ICML 2024
- **作者**：Yilong Zhao et al. (Together AI)
- **核心贡献**：
  - **混合粒度量化**：per-channel + per-group结合，平衡精度和速度
  - 提出**Atom**：4bit W4A16，同时有高精度和高速度
  - 开源了高效CUDA kernel，速度比Marlin还快
  - 直接集成在vLLM中，生产级可用
- **链接**：https://arxiv.org/abs/2310.19102
- **代码**：https://github.com/efeslab/Atom

### 7. VS-Quant: Per-vector Scaling for Accurate 4-bit LLM Quantization
- **会议**：2024 (Preprint)
- **作者**：Saleh Ashkboos et al. (ETH Zurich)
- **核心贡献**：
  - 发现per-token activation outlier导致per-group量化精度损失
  - **Per-vector scaling**：更细粒度的缩放因子
  - W4A16精度超越AWQ/QuIP#，且硬件友好
  - outlier channel不需要单独处理
- **链接**：https://arxiv.org/abs/2405.16109

### 8. HQQ: Half-Quadratic Quantization for Large Language Models
- **会议**：2024 (Preprint)
- **作者**：Mohammad H. Khatab et al.
- **核心贡献**：
  - 基于半二次优化的快速PTQ，**不需要校准数据**（zero-shot！）
  - 校准速度极快（几秒），精度媲美需要校准的方法
  - 支持任意bit-width（W8/W4/W3/W2）
  - 开源：hqq库，与TorchAO集成
- **链接**：https://arxiv.org/abs/2310.02980
- **代码**：https://github.com/mobiusml/hqq

### 9. SpQR: A Sparse-Quantized Representation for Near-Lossless LLM Weight Compression
- **会议**：NeurIPS 2023
- **作者**：Tim Dettmers et al.
- **核心贡献**：
  - **稀疏+量化混合**：大部分权重3-4bit，少量outlier权重保留FP16稀疏存储
  - 首次实现W3 near-lossless quantization（PPL损失<1%）
  - 识别outlier权重的模式
- **链接**：https://arxiv.org/abs/2306.03078
- **代码**：https://github.com/facebookresearch/spqr

### 10. GPTVQ: Fast Finetuning-based Post-Training Quantization for Large Language Models
- **会议**：ICLR 2025 (Spotlight)
- **作者**：Jung Hwan Heo et al.
- **核心贡献**：
  - 将向量量化和少量微调结合
  - W2A16精度大幅超越PTQ方法，接近FP16
  - 量化时间短于GPTQ
- **链接**：https://arxiv.org/abs/2402.15050

---

## 其他重要工作

| 论文 | 会议 | 年份 | 核心亮点 |
|------|------|------|---------|
| **SqueezeLLM** | Preprint | 2023 | 基于密度的非均匀量化，W4/W3/W2 |
| **OWQ** | ICML 2024 | 2024 | Outlier-Aware Weight Quantization，outlier权重FP16，其他W4，加速推理 |
| **PB-LLM** | ICLR 2024 | 2024 | 部分二值化，大部分1bit，关键部分高bit |
| **BiLLM** | ICML 2024 | 2024 | 1.58bit二值化LLM，首个极端压缩下还能保持性能的工作 |
| **OneBit** | Preprint | 2024 | 1-bit LLM，结合Hadamard变换和蒸馏 |
| **T-MARS** | ICLR 2025 | 2024 | Masking out Salient Tokens加速AWQ校准，校准更快 |
| **TorchAO** | Library | 2024 | Meta官方量化库，集成各种W4/W8算法，PyTorch原生 |

---

## 阅读顺序建议

```
入门必读：
  GPTQ (2023) → AWQ (2023) → QLoRA (2023) → SpQR (2023)

最新SOTA：
  QuIP# (2024) → AQLM (2024) → Atom (2024) → VS-Quant (2024) → HQQ (2024)
```

## 开源实现参考

| 库 | 支持方法 | 特点 |
|---|---------|------|
| **AutoGPTQ** | GPTQ | 最流行，生态好 |
| **AutoAWQ** | AWQ | 速度快，vLLM支持 |
| **llama.cpp** | 多种 | GGUF格式，CPU/GPU/端侧 |
| **TorchAO** | 多种 | PyTorch官方，灵活 |
| **bitsandbytes** | NF4/8bit | QLoRA使用，易用 |
| **Marlin** | W4A16 kernel | 目前最快的W4 CUDA kernel |
