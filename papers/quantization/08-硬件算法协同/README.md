# 08 - 硬件算法协同设计
> 算法和硬件/kernel/编译器协同设计，不仅精度高，实际部署也快
> 系统顶会（OSDI/NSDI/ISCA/MICRO/MLSys/ASPLOS）最爱
> 含金量最高，但需要写CUDA/系统代码
---
## 背景：为什么硬件协同很重要？
很多算法论文的"快"是假的——只算理论OPS减少，实际跑起来因为：
- 非均匀量化需要查表，内存访问不连续
- Hadamard变换有额外开销
- 细粒度混合精度需要动态调度
- 没有高效CUDA kernel，速度甚至比FP16慢
硬件算法协同设计就是：**从一开始就考虑硬件特性设计量化算法，同时为算法写高效kernel**。
---
## 高效量化Kernel工作
### 1. Marlin: Fast 4-bit Inference Kernel
- **会议**：Preprint 2024
- **作者**：Elias Frantar (GPTQ作者) et al.
- **核心贡献**：
  - 目前最快的W4A16 CUDA kernel
  - 针对Ampere/Hopper GPU优化（张量核心、内存合并访问、warp级tiling）
  - 支持group-wise量化，无额外开销
  - 比之前的W4 kernel快2-3倍
  - 已集成到vLLM/TensorRT-LLM/AutoAWQ
- **链接**：https://github.com/IST-DASLab/marlin
### 2. ExLlama/ExLlamaV2
- 社区实现的快速W4/W3 kernel
- ExLlamaV2支持混合bit（不同层不同bit），速度极快
- 在消费级GPU（3090/4090）上优化特别好
- 很多推理前端用这个
### 3. GPTQModel/Marlin优化
- Marlin基础上优化，支持更多格式和模型
### 4. BitNet 1.58 Kernel
- **会议**：Preprint 2024
- 三元权重{-1,0,+1}，GEMM变成纯累加（不需要乘法）
- 官方kernel速度是FP16的~4-6倍，能耗只有1/10
- ARM/NPU端侧也有对应优化
- **链接**：https://github.com/microsoft/BitNet
### 5. Atom W4 Kernel
- Atom算法配套的高效W4A16 kernel
- 速度比Marlin还略快（某些shape）
- 和vLLM集成
### 6. QuIP# Inference Optimization
- QuIP#一开始Hadamard变换开销大，后来优化把Hadamard融合进GEMM kernel
- 2bit推理速度接近W4
### 7. QServe: W4A8KV4 System
- **会议**：ISCA 2025
- **核心**：系统级量化+kernel，W4A8KV4全栈优化
- 四个关键技术：
  1. W4A8非均匀量化算法
  2. KV4 4bit KV Cache量化
  3. 专用CUDA kernel
  4. 内存访问优化
- 比FP16快6-8x，精度损失极小
- 吞吐量提升显著
- **链接**：https://arxiv.org/abs/2405.04532
---
## 编译器级量化优化
### 8. Torch Compile + Quantization
- PyTorch 2.0 torch.compile自动融合量化算子
- 动态生成高效kernel
### 9. TVM/Ansor自动调优
- 自动搜索最优量化kernel schedule
- 支持多种硬件（GPU/CPU/端侧NPU）
### 10. TensorRT量化
- NVIDIA官方编译器，图优化+量化融合
- W8A8/FP8性能最好
- TensorRT-LLM是专门针对LLM的版本
### 11. GGML/llama.cpp 量化
- 纯C实现，跨平台
- Q4_K/Q5_K/Q6_K等格式针对CPU/ARM NEON优化
- 支持Metal（Apple Silicon）、CUDA、Vulkan
### 12. MLC LLM
- 基于TVM Unity，自动量化编译多端部署
- 支持iPhone/Android/Web
---
## 硬件感知量化算法
算法设计阶段就考虑硬件特性：
### 13. SmoothQuant硬件友好设计
- SmoothQuant是per-channel缩放，硬件原生支持，不需要额外kernel
- 这也是为什么SmoothQuant能快速部署的原因
### 14. Hardware-aware AWQ/GPTQ
- 在量化时就考虑目标硬件的kernel效率
- 不是只追求PPL最低，而是PPL-speed Pareto最优
### 15. HASP: Hardware-Aware Automatic Search
- 搜索混合精度时考虑实际硬件延迟
- 不是bit越低越快——某些shape下W4比W8慢
### 16. FP8（E4M3/E5M2）
- NVIDIA H100/RTX40系列**硬件原生支持**FP8
- 不需要模拟，张量核心直接跑FP8 GEMM
- 速度比INT8还快，精度更好
- FP8-LM/Transformer Engine做了大量优化
- 现在工业界部署的首选方向之一
---
## PIM/近存计算量化
Processing-in-Memory（内存中计算），解决内存墙：
- DRAM中直接做低bit计算，不需要搬数据
- 三元/二值权重特别适合PIM
- ISCA/MICRO顶会常有这类工作
---
## 稀疏+量化协同
结构化稀疏（N:M稀疏）和量化结合：
- 2:4稀疏 + W8 = 等效W4速度，精度更高
- NVIDIA Ampere+ 硬件支持2:4稀疏
- Gear-KV就是稀疏+量化联合
---
## CPU/边缘端量化优化
### 17. llama.cpp Q4_K/Q5_K
- ARM NEON指令集优化
- Apple Metal加速
- CPU上W4速度比FP16快3-4x
### 18. Intel AMX (Advanced Matrix Extensions)
- Sapphire Rapids+ 支持INT8/INT4/FP16 GEMM
- AMX优化的量化kernel
### 19. ARM SVE/SME量化
- ARM v9 SVE2指令集支持低bit计算
- 手机/服务器ARM芯片都用
### 20. RISC-V 扩展
- RISC-V矩阵扩展+量化指令
- 端侧芯片方向
---
## 关键挑战
| 挑战 | 说明 |
|------|------|
| **算法-硬件迭代不同步** | 硬件新指令（FP8/INT4）刚出，算法还没跟上；新算法出来硬件/kernel还没优化 |
| **kernel开发成本高** | 写高效CUDA kernel需要很强的系统能力，不是每个算法研究者都能做 |
| **跨硬件泛化** | 一个kernel在A100快，在4090/端侧NPU不一定快 |
| **量化算子融合** | 量化/反量化/激活/注意力怎么融合成一个kernel，减少内存访问 |
| **动态batch/变长序列** | LLM推理batch size和序列长度都是动态的，kernel需要处理这种不规则性 |
| **PagedAttention + 量化协同** | 分页KV和量化怎么结合，页内量化、缩放因子存储等问题 |
---
## 系统顶会偏好
OSDI/NSDI/ISCA/MICRO/ASPLOS这类系统顶会喜欢的工作模式：
```
提出问题 → 观察现有系统瓶颈 → 设计新的量化算法（硬件友好）
                     ↓
      设计/实现对应kernel/系统优化 → 端到端集成到推理引擎
                     ↓
      真实硬件上做详尽评测：速度↑ 显存↓ 精度不掉
```
**纯粹的算法（只报PPL提升）系统顶会不会中，必须有系统实现和实际速度/端到端提升。**
---
## 开源学习资源
如果想入门写量化kernel：
1. **Marlin源码**：最好的W4 kernel学习材料，注释清晰
2. **FlashAttention-2/3**：学习GPU kernel优化技巧
3. **Cutlass**：NVIDIA官方GEMM模板库，大部分高性能kernel基于Cutlass
4. **llama.cpp**：CPU/端侧kernel学习，纯C不依赖CUDA，更容易读
5. **vLLM/TensorRT-LLM源码**：看生产级系统怎么集成量化
6. **CUTLASS Python**：快速原型量化kernel
---
## 研究机会
1. **W2/W3高效CUDA kernel**：现在W4 kernel很快，W2/W3 kernel还有很大优化空间
2. **非均匀量化硬件友好设计**：怎么设计非均匀量化算法不需要查表/额外变换？
3. **KV Cache量化kernel**：KV量化kernel优化不如权重量化成熟
4. **投机采样+量化kernel融合**：draft和target模型的量化怎么融合加速
5. **多端统一量化kernel**：一个算法能在GPU/CPU/NPU上都高效跑
6. **PagedAttention+量化系统设计**：分页KV下的量化
7. **FP4/INT4 Hopper优化**：NVIDIA Blackwell支持FP4，这是下一个热点
---
## 阅读建议
系统方向需要动手能力，建议：
1. 先读 **Marlin** 论文+看源码，理解W4 kernel怎么写
2. 读 **QServe**，看全栈系统级工作是什么样的
3. 读 **FlashAttention-2**，学习GPU kernel优化方法论
4. 动手：在Marlin基础上改一个简单的W3 kernel，实际测速度，你会学到很多
5. 系统论文：一定要看论文里的"Evaluation"部分，看他们怎么设计实验，怎么和baseline公平对比
