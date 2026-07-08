# 03 - KV Cache 量化
> 长上下文场景的核心瓶颈：KV Cache占70-90%显存，不压缩KV就没法跑长上下文
> 🔥 当前最热的量化方向之一，工业界强需求
---
## 背景：为什么KV Cache量化这么重要？
长上下文推理时：
```
模型权重：70B W4 → ~40GB（固定不变）
KV Cache：128k上下文 → ~80GB（随上下文长度线性增长！）
          1M上下文 → ~600GB+
```
**权重压缩到4bit也没用，KV Cache才是长上下文显存瓶颈！**
而且KV Cache和权重不同：
- KV是**动态生成**的，无法提前校准
- 长上下文下误差会**累积**，几千步后精度崩塌
- 需要和PagedAttention/Prefix Cache兼容
- Token生成时逐token写入，读取模式特殊
---
## 经典基础工作
### 1. Efficiently Scaling Transformer Inference (PagedAttention)
- **会议**：OSDI 2023
- **作者**：Woosuk Kwon et al. (vLLM, UC Berkeley)
- **核心贡献**：
  - 提出PagedAttention，类似OS虚拟内存分页管理KV Cache
  - 解决KV Cache内存碎片化问题，显存利用率从20-40%提升到90%+
  - vLLM核心技术，后续KV量化都需要和PagedAttention兼容
  - **不是专门讲量化**，但所有KV量化工作都基于这个框架
- **链接**：https://arxiv.org/abs/2309.06180
- **代码**：https://github.com/vllm-project/vllm
### 2. SmoothQuant对KV的初步探索
- SmoothQuant最初只量化权重和激活，后来发现V缓存可以直接量化，但K缓存不行（对精度更敏感）
---
## 2024-2025 最新SOTA
### 3. KIVI: A Tuning-Free Asymmetric 2bit Quantization for KV Cache
- **会议**：ICML 2024
- **作者**：Zijian Liu et al. (MIT)
- **核心贡献**：
  - **首个2bit KV Cache量化PTQ方案**，不需要微调
  - 非对称量化：K2/V4 或者 K4/V2，因为V对精度更敏感
  - 残差量化：离群值单独存储，量化剩下的
  - 2bit KV + 4bit权重 → 70B模型在单张A100跑长上下文
  - 支持即插即用，不需要训练
- **链接**：https://arxiv.org/abs/2402.02750
- **代码**：https://github.com/jy-yuan/KIVI
### 4. KVQuant: Towards 10 Million Context Length LLM Inference with KV Cache Quantization
- **会议**：NeurIPS 2024
- **作者**：Coleman Hooper et al. (UC Berkeley)
- **核心贡献**：
  - 系统性分析KV Cache中不同通道/层/位置的量化敏感度
  - 提出**通道级混合精度**：敏感通道FP16，其他4bit
  - **稀疏感知**：不重要的KV直接量化到更低bit
  - **误差补偿**：量化误差纠正，防止长上下文累积崩塌
  - 支持10M+上下文长度，精度损失极小
  - 针对长上下文优化，解决误差累积问题
- **链接**：https://arxiv.org/abs/2401.18079
- **代码**：https://github.com/SqueezeAILab/KVQuant
### 5. Gear: Efficient KV Cache Compression with N:M Sparsity
- **会议**：Preprint 2024
- **核心贡献**：
  - 利用KV的N:M稀疏性，结合量化
  - 稀疏+量化联合压缩，压缩比更高
  - 高效GPU kernel实现
- **链接**：https://arxiv.org/abs/2402.13358
### 6. FlashDecoding++: Faster Large Language Model Inference on GPUs
- **会议**：ASPLOS 2025
- **作者**：Hongyi Wang et al.
- **核心贡献**：
  - 提出KV Cache分块量化技术
  - 融合量化和注意力计算kernel
  - 速度比FlashAttention-2快，同时支持KV量化
  - 工业级实现，部署在生产环境
- **链接**：https://arxiv.org/abs/2311.01282
### 7. SparseKV: KV Cache Compression with Dynamic Sparsity
- **会议**：ICLR 2025
- **核心贡献**：
  - 动态识别不重要的KV token，量化/稀疏化
  - 结合H2O（Heavy Hitter Oracle）思想，保留关键KV
  - 长上下文下压缩比5-10倍，精度掉点小
### 8. QCache: Query-Aware KV Cache Quantization
- **会议**：Preprint 2024
- **核心贡献**：
  - KV的重要性和query相关，逐query自适应量化精度
  - 不重要的KV用更低bit
  - 比固定精度量化效果更好
### 9. MiKV: Mixed-Precision KV Cache Quantization with Error Feedback
- **会议**：ICML 2024 Workshop
- **核心贡献**：
  - 逐层/逐头混合精度自动搜索
  - 误差反馈机制：前层的量化误差在后层补偿
  - 解决长上下文误差累积
### 10. KV-Compress: A Comprehensive Framework for KV Cache Compression
- **会议**：NeurIPS 2024
- **核心贡献**：
  - 统一框架支持量化+稀疏+淘汰
  - 自动选择每层最优压缩策略
  - 开源实现，与vLLM集成
---
## KV Cache量化的特殊挑战
| 挑战 | 说明 | 现有解决方案 |
|------|------|-------------|
| **动态生成，无法预校准** | KV在推理时才产生，不能提前用数据集校准 | 在线校准、动态范围更新、零样本量化 |
| **长上下文误差累积** | 数千次更新后误差放大，末尾生成崩塌 | 误差补偿、残差量化、周期刷新范围 |
| **K和V敏感度不同** | K一般比V更敏感，V可以压更低 | K4/V2非对称精度 |
| **通道分布异质** | 不同head/通道分布差异极大 | 通道级混合精度、per-channel缩放 |
| **和PagedAttention兼容** | KV分页存储，量化需要页对齐 | 页内独立量化、块级缩放因子 |
| **和Prefix Cache兼容** | Hermes等依赖Prefix Cache，量化不能改变prefix字节 | 前缀感知量化，保证prefix区域量化后字节一致 |
| **Streaming场景** | 逐token生成，不能做全局归一化 | 在线/流式量化算法，EMA更新范围 |
| **Key的周期性/OOB问题** | Rotary Position Embedding导致某些位置K值爆炸 | 针对RoPE的特殊处理 |
---
## 压缩技术分类
| 技术类型 | 代表方法 | 压缩比 | 精度 | 速度 |
|---------|---------|--------|------|------|
| **纯量化** | KIVI/KVQuant/W8KV | 2-8x | 高 | 快（kernel容易写） |
| **稀疏化** | H2O/Gear/StreamingLLM | 4-20x | 中 | 需要稀疏kernel |
| **Token淘汰/驱逐** | FastGen/Landmark Attention | 10-100x | 中低 | 快 |
| **低秩分解** | LoRA-KV | 4-8x | 高 | 中等 |
| **多技术结合** | KV-Compress | 8-32x | 可调 | 需要系统优化 |
---
## 系统实现：推理引擎中的KV量化
### vLLM
- v0.4+ 开始支持FP8 KV Cache
- v0.5+ 支持W8A8 KV
- 实验性支持KIVI 2bit KV
- 和PagedAttention深度集成
### SGLang
- 支持FP8 KV Cache
- RadixAttention + KV量化，前缀缓存复用
- 针对长上下文优化
### TensorRT-LLM
- NVIDIA官方W8A8/FP8 KV Cache
- 高度优化的kernel
- 支持in-flight batching + 量化KV
### llama.cpp
- Q8_0/Q4_0 KV量化，CPU/端侧
- GGUF格式支持
---
## 关键论文延伸：KV淘汰与稀疏
这些不完全是量化，但和KV量化一起用效果更好：
- **H2O: Heavy-Hitter Oracle for Efficient Generative Inference of LLMs** (NeurIPS 2023)：保留"heavy hitter"KV，淘汰不重要的
- **StreamingLLM** (ICLR 2024)：Attention sink机制，保留前几个token，滑动窗口
- **Landmark Attention** (NeurIPS 2023)：地标token标记块，随机访问
- **FastGen** (2024)：动态token驱逐，自适应窗口大小
- **InfLLM** (NeurIPS 2024)：长上下文KV分页存储，按需加载
---
## 未解决的问题与研究机会
1. **W4以下的KV量化**：W4还能保持精度，W2/A2目前掉点还比较明显
2. **长上下文（1M+）误差累积**：现有方法大多测128k，1M以上误差累积问题没解决
3. **与Prefix Cache兼容**：量化改变KV值导致缓存失效，这个Hermes遇到的问题还没人系统解决
4. **流式/实时量化**：边生成边量化，不需要预读整个序列
5. **Agent场景下的KV量化**：Agent多轮对话工具调用场景KV特征不同，专门优化
6. **硬件友好的KV量化kernel**：很多算法精度好，但kernel慢，实际部署达不到速度up
---
## 阅读建议
1. 先看**PagedAttention (OSDI 2023)**，理解KV Cache存储和管理
2. 再读**KIVI**，了解第一个2bit KV方案怎么做的
3. 读**KVQuant**，学习混合精度+误差补偿怎么解决长上下文
4. 动手：把vLLM的FP8 KV跑起来，实测长上下文下的精度变化，你会发现很多问题
5. 机会点：Prefix Cache + KV量化的兼容性问题现在几乎没人做，是很好的切入点
