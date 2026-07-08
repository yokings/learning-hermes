# 04 - Sub-4bit 低比特量化（3bit/2bit/1bit/三元）
> 极限压缩：让70B模型跑在24GB消费卡，大模型跑在手机/边缘设备
> 难点：bit越低，精度掉得越厉害，需要特殊技术补偿
---
## 背景：为什么追求低于4bit？
| 位宽 | 7B模型显存 | 13B模型 | 70B模型 | 适用场景 |
|------|----------|---------|---------|---------|
| FP16 | ~14GB | ~26GB | ~140GB | A100 80G |
| W8A16 | ~7GB | ~13GB | ~70GB | A100 80G |
| W4A16 | ~3.5GB | ~7GB | ~35GB | 4090 24G可跑70B |
| **W3A16** | ~2.6GB | ~5GB | ~26GB | 24G卡轻松跑70B |
| **W2A16** | ~1.75GB | ~3.5GB | ~17GB | 16G笔记本可跑70B |
| **W1.58/三元** | ~1.4GB | ~2.6GB | ~13GB | 手机/边缘端 |
| **W1/二值** | ~0.9GB | ~1.7GB | ~9GB | MCU级？ |
---
## 2023：低比特量化开端
### 1. GPTQ 3bit
- GPTQ最初论文就支持3bit，但精度掉得比较多
- 3bit W3A16在7B上PPL涨~0.3，还能用；13B以上稍好
### 2. SpQR: Sparse-Quantized Representation
- **会议**：NeurIPS 2023
- 3-4bit量化 + 少量outlier保留FP16，实现near-lossless
- W3精度损失<1%，但需要稀疏存储支持，速度慢
- **链接**：https://arxiv.org/abs/2306.03078
---
## 2024：2bit爆发年
### 3. QuIP#: 2bit Weight Quantization
- **会议**：ICML 2024
- **核心**：Hadamard变换让权重incoherent + E8格点码本
- W2A16在7B/13B上PPL损失很小，第一次让2bit真正可用
- W4A4效果也很好
- **链接**：https://arxiv.org/abs/2402.04396
### 4. AQLM: Additive Quantization
- **会议**：ICML 2024
- **核心**：加性向量量化，多个码本加和重构权重
- 2bit精度显著优于GPTQ，接近W4 GPTQ
- GPU推理加速支持
- **链接**：https://arxiv.org/abs/2401.06118
### 5. PB-LLM: Partially Binarized LLMs
- **会议**：ICLR 2024
- **核心**：部分二值化（Partially Binarized）
- 大部分权重1bit，关键权重通道保留高bit
- W1A16精度可用，速度快
- **链接**：https://arxiv.org/abs/2310.00034
### 6. BiLLM: Pushing the Limit of Post-Training Quantization for LLMs
- **会议**：ICML 2024
- **作者**：Wei Huang et al. (Tsinghua)
- **核心**：
  - 首个真正意义上**1-bit/二值**LLM的PTQ方案，不需要QAT
  - 发现权重分布有双峰结构，残差二值化
  - 1.0-1.5bit精度达到之前W2水平
  - 7B/13B上1bit保持不错的下游任务性能
- **链接**：https://arxiv.org/abs/2402.04291
- **代码**：https://github.com/Aaronhuang-777/BiLLM
### 7. BitNet: 1-bit LLM
- **会议**：Preprint 2023/2024
- **作者**：Microsoft Research
- **BitNet b1.58** (2024)：三元权重{-1, 0, +1}，即1.58bit
  - 从头训练1.58bit模型，不是PTQ
  - 性能与FP16 Llama相当
  - 推理速度快，内存省，能耗低
  - 适合端侧部署
- **链接**：BitNet: https://arxiv.org/abs/2310.11453
- **BitNet b1.58**: https://arxiv.org/abs/2402.17764
### 8. OneBit: Towards Extremely Low-bit Large Language Models
- **会议**：Preprint 2024
- **核心**：1-bit LLM，结合Hadamard变换 + 知识蒸馏
- W1A16比之前的二值化方法效果好
- **链接**：https://arxiv.org/abs/2402.11295
### 9. TernaryLLM: 1.58bit Ternary Weight Quantization
- **会议**：ICLR 2025
- **核心**：PTQ三元权重{-1,0,+1}，不需要从头训练
- 精度接近BitNet b1.58，但不需要重训
- **链接**：OpenReview
---
## 2024-2025：更好的低bit + 加速
### 10. AQLM++ / AQLM Inference Speedup
- AQLM的2bit/3bit GPU kernel优化
- 2bit推理速度达到W4 Marlin ~70%
- **链接**：https://github.com/Vahe1994/AQLM
### 11. QuIP# with Fast Inference
- QuIP#的Hadamard变换融合到GEMM kernel
- 减少额外开销，速度接近纯W4量化
### 12. DQ-ViT/LLM: Dynamic Quantization
- 动态混合精度，低bit基础上少量权重高bit补偿
- 2bit基础+1%高bit → 接近W4精度
### 13. SliM-LLM: Structured Low-bit Mixture-of-Quantization
- **会议**：ICML 2024
- **核心**：结构化混合低bit，不同通道不同bit，硬件友好
- W2A16高精度，且速度快于非结构化低bit
- **链接**：https://arxiv.org/abs/2405.14834
### 14. L-MBP: Low-bit Model Bootstrap
- 用W16模型引导W2/W1模型训练，低bit QAT成本低
- 少数据QAT达到很好的2bit效果
### 15. MoDQ: Mixed-order Data-free Quantization
- 无数据/少数据的极低位量化
- 不需要校准数据，适合没有公开数据集的模型
---
## 极低位量化的关键技术
### 技术1：Incoherence Processing（让权重分布"无规律"）
```
问题：权重有很强的相关性/ outliers，直接低bit量化误差大
解决方案：正交/Hadamard变换
  Y = W X
  = W H H^T X  （H是正交Hadamard矩阵，H^T H = I）
  = (W H) (H^T X)
变换后 W H 的分布更均匀（incoherent），更容易量化
代表方法：QuIP、QuIP#、OneBit、SpinQuant
```
### 技术2：非均匀/码本量化
```
问题：INT1/INT2均匀量化精度不够
解决方案：用学习的码本（codebook）做向量量化
  - K-means码本（AQLM）
  - E8格点码本（QuIP#）
  - 多码本加和（AQLM/AQ）
  - 自适应码本（非均匀量化）
```
### 技术3：混合精度/部分高bit
```
问题：所有层/通道都低bit不行，某些部分特别敏感
解决方案：
  - 部分通道保留FP16/INT8（SpQR、PB-LLM）
  - outlier单独存储，其他低bit
  - 自动搜索混合精度配置
```
### 技术4：残差量化
```
先量化 → 残差 → 再量化残差 → 加起来
类似Boosting思想，逐层降低误差
BiLLM的核心就是残差二值化
```
### 技术5：蒸馏辅助
```
用FP16/W4模型作为老师，指导低bit学生模型
QAT场景效果特别好
OneBit、BitNet都用到了蒸馏
```
---
## 三元（1.58bit）为什么是特殊的？
三元权重是 {-1, 0, +1} 三个值：
- 数学上：log2(3) ≈ **1.58 bit** per weight
- 硬件上：只需要 {-1, 0, +1} 三个状态，乘法可以用加法代替！
  - 不需要乘法器 → 芯片面积小、能耗低、速度快
- 推理时GEMM变成累加：
  ```
  y_i = Σ w_ij x_j = Σ (+x_j) - Σ (-x_j)  // 只有加减，没有乘法
  ```
- 端侧/手机芯片特别喜欢三元，能效比极高
BitNet b1.58证明了从头训练的三元模型性能媲美FP16 LLaMA。
---
## 二值化（1bit）的困难
1bit只有{-1,+1}两个值：
- **表示能力极限低**：每个权重只能表示正负，量级信息丢了
- **乘法变成符号判断**：快，但精度损失严重
- 现有方案需要残差/部分高bit补偿才能work
- 纯1bit PTQ目前在7B以上还是掉点明显，QAT/从头训效果更好
---
## 硬件实现现状
| 位宽 | 现有开源kernel | 速度对比FP16 |
|------|---------------|-------------|
| W8A16 | cuBLAS/GEMM | ~2x faster |
| W4A16 | Marlin/AutoAWQ/AutoGPTQ | ~3-4x faster |
| W3A16 | QuIP#/AQLM | ~2-3x（还在优化） |
| W2A16 | QuIP#/AQLM | ~1.5-2.5x（有额外开销） |
| W1.58三元 | BitNet官方 | ~4-6x（只有加减！） |
| W1/二值 | 研究阶段 | 理论上更快，但kernel还不成熟 |
关键问题：非均匀/码本量化/Hadamard变换带来额外开销，实际速度up未必有理论上那么大。**硬件算法协同设计**空间很大。
---
## 研究机会
1. **W2A16精度继续提升**：现有2bit比W4还是差一点，能不能接近W4？
2. **W2以下PTQ不用训练**：BitNet需要从头训，能不能PTQ就达到QAT效果？
3. **更快的低bit kernel**：QuIP#/AQLM的Hadamard开销能不能消掉？
4. **三元PTQ**：BitNet是从头训，有没有好的post-training三元量化？
5. **低bit + KV Cache压缩**：权重W2 + KV W2，70B模型压到8GB，手机跑
6. **低bit下Agent/工具调用能力保持**：现在PPL看着不错，但实际工具调用准确率可能掉很多
---
## 阅读顺序建议
```
入门：
  GPTQ 3bit → SpQR → QuIP# → AQLM
二值/三元：
  PB-LLM → BiLLM → BitNet → BitNet b1.58
优化方向：
  SliM-LLM → AQLM kernel → 看Marlin怎么写W4 kernel
```
