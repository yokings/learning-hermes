# 02 - W8A8 联合量化
> Weight和Activation都量化到8bit，实现全INT8推理，速度最快，内存最少
> 难点：Activation分布存在严重outlier，直接量化掉点严重

---

## 经典基础工作

### 1. LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale
- **会议**：NeurIPS 2022
- **作者**：Tim Dettmers et al.
- **核心贡献**：
  - 首次系统性分析LLM activation outlier现象：特定特征通道出现在所有token中，数值极大
  - **混合精度分解**：99.9%权重用INT8，outlier特征通道（约0.1%）用FP16计算
  - 推理时从hidden states中抽取outlier，单独用FP16计算，其他用INT8
  - 精度几乎无损，70B模型可在单卡A100跑
- **链接**：https://arxiv.org/abs/2208.07339
- **代码**：bitsandbytes库

### 2. SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models
- **会议**：ICML 2023
- **作者**：Guangxuan Xiao et al. (MIT, NVIDIA)
- **核心贡献**：
  - 核心洞察：**量化难度可以在权重和激活之间迁移**
  - activation的outlier对应权重往往方差小，通过per-channel缩放，平滑activation outlier
  - 数学等价变换：`Y = (X diag(s)) * (diag(s)^{-1} W)`，缩放后X更平滑，W稍变
  - **无需混合精度**，真正的全W8A8量化，精度接近FP16
  - 工业界W8A8标准方案，TensorRT-LLM/vLLM都实现
- **链接**：https://arxiv.org/abs/2211.10438
- **代码**：https://github.com/mit-han-lab/smoothquant

### 3. ZeroQuant: Efficient and Affordable Post-Training Quantization for Large-Scale Generative Transformers
- **会议**：NeurIPS 2022
- **作者**：Zhewei Yao et al. (Microsoft)
- **核心贡献**：
  - (1) 轻量级PTQ不需要微调；(2) 知识蒸馏辅助量化；(3) 支持组量化
  - W8A8/W6A6量化，零精度损失，INT8加速
  - ZeroQuant-V2后续提出W4A8等更细粒度
- **链接**：https://arxiv.org/abs/2206.01861

---

## 2024-2025 最新进展

### 4. OmniQuant: Omnidirectionally Calibrated Quantization for Large Language Models
- **会议**：ICLR 2024
- **作者**：Wenqi Shao et al. (Shanghai AI Lab)
- **核心贡献**：
  - 统一框架同时优化**可学习的equivalent transformation**和**可学习的clipping参数**
  - 不需要SVD/混合精度，W8A8/W6A6/W4A4都能保持高精度
  - 基于梯度的优化，校准速度快
- **链接**：https://arxiv.org/abs/2308.13137
- **代码**：https://github.com/OpenGVLab/OmniQuant

### 5. LLM-FP4: 4-bit Floating-Point Quantization for Large Language Models
- **会议**：ICLR 2025
- **作者**：Jung Hwan Heo et al.
- **核心贡献**：
  - 提出FP4（4-bit浮点）量化，比INT4精度更好
  - 结合block-wise scaling和SmoothQuant思想
  - W4A8/W4A4都有很好效果
  - 硬件友好，FP4在新NVIDIA GPU上有原生支持
- **链接**：https://arxiv.org/abs/2401.14112

### 6. FP8-LM: Training and Serving FP8 Large Language Models
- **会议**：ICML 2024
- **作者**：Boris Ginsburg et al. (NVIDIA)
- **核心贡献**：
  - 系统性研究FP8 E4M3/E5M2格式在LLM训练和推理中的应用
  - 提出FP8训练recipe，混合精度不需要outlier提取
  - 生产级FP8推理，NVIDIA官方方案
- **链接**：https://arxiv.org/abs/2309.16553

### 7. INT8-MiniLLM: INT8 Quantization for Small-Scale Language Models
- **会议**：Preprint 2024
- **核心贡献**：发现小模型（1B-7B）W8A8掉点比大模型严重，提出针对性校准方法
- **链接**：https://arxiv.org/abs/2402.05060

### 8. QuiPT: Quantization with Perturbations for 4-bit Inference
- **会议**：ICLR 2025
- **作者**：Anonymous
- **核心贡献**：
  - W4A4量化，用Hadamard变换+随机扰动
  - 接近W4A16精度，速度更快
- **链接**：OpenReview

### 9. PerAdaQuant: Adaptive Per-Tensor Quantization for LLMs
- **会议**：ICML 2024 Workshop
- **核心贡献**：自适应决定per-tensor还是per-channel量化，自动搜索最优粒度
- **链接**：ArXiv

### 10. SpinQuant: LLM Quantization Learned via Rotating Basis
- **会议**：ICML 2024
- **作者**：Rang Meng et al.
- **核心贡献**：通过学习正交旋转变换，让权重和激活同时更易量化，W8A8/W4A4精度提升
- **链接**：https://arxiv.org/abs/2402.05973

---

## 关键技术问题与解决方案对比

| 问题 | 代表方法 | 核心思想 |
|------|---------|---------|
| **Activation outlier** | LLM.int8() | outlier通道分离FP16计算 |
| | SmoothQuant | 缩放迁移难度：激活→权重 |
| | RPTQ/RotateQuant | 正交变换打乱outlier |
| **权重分布不均** | GPTQ/AWQ | 二阶信息/salient weight保护 |
| | OmniQuant | 可学习缩放+clipping |
| | QuIP#/SpinQuant | Hadamard/正交变换让权重incoherent |
| **全低bit（W4A4）** | QuIP#/Atom | 非均匀量化+码本+Hadamard变换 |
| | FP8-LM | 用FP4/FP8浮点格式代替INT |

---

## 工业界部署方案

| 引擎 | W8A8支持 | FP8支持 | 说明 |
|------|---------|---------|------|
| **TensorRT-LLM** | ✅ SmoothQuant | ✅ | NVIDIA官方，性能最好 |
| **vLLM** | ✅ | ✅ 实验性 | 社区活跃，支持各种模型 |
| **SGLang** | ✅ | ✅ | 微软优化，RadixAttention |
| **llama.cpp** | ✅ Q8_0/Q4_0 | ❌ | CPU/端侧，GGUF |
| **TensorRT** | ✅ | ✅ | 通用推理 |
| **ONNX Runtime** | ✅ | ❌ | 跨平台 |

---

## 阅读建议

1. 首先理解 **outlier问题**：读LLM.int8()，搞懂为什么直接W8A8不行
2. 然后读 **SmoothQuant**：理解"难度迁移"这个核心insight，这是最优雅的思想
3. 再看OmniQuant/SpinQuant：学习怎么用可学习/正交变换做更好的量化
4. 工业界部署直接看TensorRT-LLM/vLLM的W8A8实现
