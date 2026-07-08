# 05 - 混合精度与自动化量化
> 不同层/通道/Token用不同精度，自动搜索最优配置
> 解决"一刀切"量化精度掉点问题，同时保持速度
---
## 背景：为什么需要混合精度？
均匀量化（所有层/权重都INT4/INT8）简单，但问题是：
- **不同层敏感度差异巨大**：第一层/最后一层/注意力输出层特别敏感，量化掉点严重；MLP up_proj相对不敏感
- **不同通道差异大**：少量outlier通道决定了大部分误差，其他通道可以压更低
- **不同token差异大**：简单token（标点、常见词）可以低bit，难token（复杂推理）需要高bit
- **不同任务差异大**：代码/数学/Agent任务对量化更敏感，闲聊对话不敏感
手动混合精度（"第一层不量化，outlier通道不量化"）是经验法则，泛化差，自动化是必然趋势。
---
## 一、粒度：混合精度可以在哪些维度做？
| 粒度 | 说明 | 精度 | 硬件友好度 |
|------|------|------|-----------|
| **Per-Layer** | 每层一个精度 | 中 | ✅ 非常友好，容易加速 |
| **Per-Channel** | 每个输出通道一个精度 | 较好 | ✅ 硬件支持 |
| **Per-Group** | 每128/64个权重一组 | 好 | 🟡 现在主流W4都是group-wise |
| **Per-Token/Activation** | 每个token激活用不同精度 | 好 | 🟡 需要动态调度 |
| **Per-Head** | 注意力每个头不同精度 | 较好 | 🟡 |
| **Per-Weight（非均匀）** | 每个权重不同bit/码本 | 最好 | ❌ 硬件不友好，速度慢 |
---
## 经典工作
### 1. LLM.int8() 混合精度分解
- **会议**：NeurIPS 2022
- **粒度**：Per-Channel outlier分离
- 0.1% outlier通道FP16，其他99.9% INT8
- 第一个真正work的LLM混合精度方案
- **链接**：https://arxiv.org/abs/2208.07339
### 2. Outlier Weight Quantization (OWQ)
- **会议**：ICML 2024
- **粒度**：Per-Weight outlier
- 识别outlier权重，3bit基础上outlier用更高精度
- 加速比纯INT3快，精度更好
- **链接**：https://arxiv.org/abs/2305.17340
### 3. SpQR: Sparse-Quantized Representation
- **会议**：NeurIPS 2023
- **粒度**：Per-Group + 稀疏outlier
- 3-4bit group-wise，少量outlier权重FP16稀疏存储
- Near-lossless W3量化
- **链接**：https://arxiv.org/abs/2306.03078
---
## 自动搜索：AutoML for Quantization
### 4. AutoQuant: Automatic Mixed-Precision Quantization for LLMs
- **会议**：ICLR 2024
- **作者**：Zhewei Yao et al. (Microsoft)
- **核心贡献**：
  - 基于Hessian traces自动搜索每层最优bit-width
  - 帕累托最优：精度vs速度/显存tradeoff曲线
  - 不需要跑完整下游任务评估，快速预测
  - W4/W8混合配置，精度比纯W4高，速度比纯W8快
- **链接**：https://arxiv.org/abs/2310.03736
- **代码**：https://github.com/mit-han-lab/autoquant
### 5. GPTQ-S: Automatic Multi-Bit Quantization
- 基于GPTQ，自动搜索不同层bit数
- 基于灵敏度分析，敏感层给高位宽
- 速度比AutoQuant快
### 6. HAWQ-V2 / HAWQ-LLM
- **会议**：NeurIPS/ICLR系列
- 基于Hessian迹的混合精度，经典CV方法迁移到LLM
- HAWQ-LLM专门针对大语言模型优化
- **链接**：https://arxiv.org/abs/2305.14266
### 7. LLM-MQ: Mixed-precision Quantization with Learned Bit-widths
- 可学习bit-width，梯度下降自动优化
- 端到端训练bit配置
### 8. Q-GaLore: Quantized GaLore
- 混合精度+低秩梯度，量化训练
- 训练时也做混合精度，节省显存
---
## 非均匀量化（Non-uniform Quantization）
非均匀量化也是混合精度的一种——所有权重相同bit，但码本不均匀分布，适配权重分布。
### 9. AQLM: Additive Quantization
- **会议**：ICML 2024
- 加性向量量化，学习非均匀码本
- W2效果极佳，见Sub4bit章节
- **链接**：https://arxiv.org/abs/2401.06118
### 10. QuIP# Lattice Codebook
- **会议**：ICML 2024
- E8/Lambda格点码本，非均匀，数学最优
- **链接**：https://arxiv.org/abs/2402.04396
### 11. QServe: W4A8KV4 Quantization
- **会议**：ISCA 2025
- **核心**：非均匀4bit激活量化，专门优化服务场景
- 4bit权重+8bit激活+4bitKV，全系统低bit
- 高效kernel，比FP16快6-8x
- **链接**：https://arxiv.org/abs/2405.04532
---
## 动态量化（运行时自适应）
静态混合精度提前决定好bit-width；动态量化**运行时**根据输入决定精度。
### 12. D2Quant: Dynamic Dual-precision Quantization
- 运行时动态决定哪些token/层用高bit哪些用低bit
- 难token用高bit，简单token用低bit，平均速度up
### 13. MiKV: Mixed-precision KV Cache
- 动态KV精度，见KV Cache章节
### 14. AdaptiveQuant: Adaptive Quantization for LLMs
- 根据输入复杂度动态切换量化精度
- 推理时实时调整，不需要提前校准
### 15. Quant-Array: Early Exit with Quantization
- 结合早退机制，简单层低bit，高层高位宽
---
## 硬件感知量化（Hardware-aware）
搜索精度配置时不仅考虑精度，还考虑**实际硬件速度**。
### 16. HASP: Hardware-Aware Automatic Quantization
- 考虑CUDA kernel在不同bit-width/shape下的实际延迟
- 不是bit越低越快——某些矩阵形状下W4比W8慢
- 搜索Pareto最优精度-速度点
### 17. AnyPrecision: Adaptive Bit-width for Inference
- 编译时自动选择最优kernel和bit-width
- 与编译器协同（TVM/TensorRT）
### 18. QServe硬件感知设计
- 不仅算法，还设计对应kernel，系统级最优
---
## 零样本/无数据量化
不需要校准数据就能量化——适合私有模型/闭源模型场景。
### 19. HQQ: Half-Quadratic Quantization
- **会议**：Preprint 2024
- 零样本（不需要校准数据），几秒完成量化
- 精度媲美需要校准的方法
- **链接**：https://arxiv.org/abs/2310.02980
### 20. ZeroQuant-V3: Zero-shot Quantization
- 不需要校准数据，基于权重统计信息
- 零样本W8A8/W4A8
### 21. LLM.int8() zero-shot部分
- LLM.int8()本身不需要校准，基于outlier统计
---
## 关键问题与挑战
| 问题 | 现状 |
|------|------|
| **搜索成本高** | AutoQuant/HAWQ搜索需要几小时，能不能更快？ |
| **速度-精度tradeoff不凸** | bit-width和实际速度不是线性关系，kernel差异大 |
| **跨模型泛化** | A模型搜到的配置，B模型能不能直接用？ |
| **硬件差异** | 不同GPU/CPU上最优配置不同 |
| **动态调度开销** | 运行时切换精度的调度开销不能太大 |
| **非均匀量化速度** | 码本量化精度高但速度慢，kernel难写 |
---
## 开源工具
| 工具 | 功能 |
|------|------|
| **AutoAWQ** | 支持per-group量化 |
| **GPTQModel** | 支持混合bit AutoGPTQ |
| **TorchAO** | 支持多种混合精度配置 |
| **llama.cpp** | GGUF支持混合位宽（Q2_K/Q3_K/Q4_K等混合） |
| **TensorRT-LLM** | 支持INT4/INT8混合精度，自动图优化 |
| **vLLM** | 支持Marlin W4 + FP8混合 |
---
## 研究机会
1. **快速精度预测器**：不用跑PPL就能预测量化后的精度损失，搜索加速几个数量级
2. **硬件感知搜索**：考虑kernel实际速度，不是bit越低越好
3. **跨模型迁移**：小模型上搜的策略迁移到大模型
4. **运行时动态混合精度**：根据输入/负载动态调整，推理过程中也能变bit
5. **非均匀量化kernel**：码本/Hadamard量化的高效CUDA核，缩小和INT4速度差距
6. **任务感知量化**：针对特定任务（代码/数学/Agent）搜索最优配置
---
## 阅读建议
1. 入门读 **SpQR** → **AutoQuant**，理解混合精度的动机和自动搜索思路
2. 然后读 **HAWQ-V2/HAWQ-LLM**，学习基于Hessian的经典方法
3. 系统方向读 **QServe**，看硬件算法协同设计
4. 自己动手：在HQQ/AutoQuant上改改，尝试一个新的搜索策略，这是容易出成果的方向
