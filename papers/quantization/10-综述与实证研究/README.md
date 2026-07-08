# 10 - 综述（Survey）与实证研究（Empirical Study）
> 先读这些建立对领域的完整认知，再去看具体方向的论文
---
## 一、量化综述论文
### 1. A Survey on Quantization of Large Language Models
- **作者**：多个机构
- **时间**：2024-2025
- **内容**：系统性综述LLM量化，涵盖PTQ/QAT/低bit/KV量化/硬件
- 分类体系清晰，包含到2024年中SOTA
- arXiv: 搜索 "LLM quantization survey" 有多个版本
### 2. A Survey of Model Compression for Large Language Models
- **时间**：2024
- **内容**：不仅量化，还有剪枝/蒸馏/稀疏/低秩分解
- 对比不同压缩技术优劣
### 3. PTQ vs QAT for LLMs: An Empirical Study
- 专门对比PTQ和QAT在不同bit下的 tradeoff
- 什么时候用PTQ，什么时候值得做QAT
### 4. Low-bit Quantization for Large Language Models: A Comprehensive Survey
- 重点关注4bit以下低bit量化
### 5. KV Cache Compression for LLMs: A Survey
- **时间**：2024-2025
- 专门讲KV Cache压缩：量化+稀疏+淘汰
- KV方向必读
---
## 二、重要实证研究（Empirical Study）
实证研究帮你理解"到底哪个方法好，好在什么情况下"，比单纯刷榜更有价值。
### 1. How Good is 4-bit Quantization? An Empirical Study of LLM Quantization
- **时间**：2024
- **发现**：
  - GPTQ/AWQ/QuIP#/HQQ在W4A16其实差距没有paper吹的那么大
  - W4在70B模型上精度损失极小
  - 不同下游任务差异大：闲聊几乎不掉，代码/数学掉点明显
  - 校准数据多样性比数量重要
### 2. LLM Quantization at Scale: An Empirical Analysis
- 在7B/13B/30B/70B多个规模测所有主流方法
- 验证量化方法是否随着模型增大效果变好
### 3. The Robustness of Quantized LLMs: An Empirical Study
- 量化后模型的鲁棒性：对抗样本、OOD输入、长上下文表现
- 发现：低bit模型鲁棒性显著下降
### 4. On the Calibration Data for Post-Training Quantization of LLMs
- **核心发现**：
  - 校准数据不需要"和目标域分布一致"
  - 随机采样的128条数据就够了，多了提升很小
  - 数据多样性比领域相关性重要
  - 解释了为什么很多PTQ方法用pile/random数据也能工作
### 5. Understanding Outliers in Large Language Models
- 系统性分析activation outlier现象
- outlier的来源、性质、为什么出现在特定特征
- 帮助理解为什么SmoothQuant/LLM.int8()能work
### 6. How Does Quantization Affect LLM Capabilities? A Multi-Dimensional Evaluation
- 多维度评估量化对LLM不同能力的影响：
  - 知识/记忆：几乎不掉
  - 推理（数学/CoT）：掉点明显
  - 代码生成：掉点明显
  - 工具调用/Agent：掉点比MMLU/GSM8K显示的严重
  - 长上下文：误差累积问题严重
### 7. KV Cache Quantization: An Empirical Evaluation
- 2024-2025
- 测了所有KV量化方法在不同上下文长度下的表现
- 128k/512k/1M长度的误差累积对比
### 8. A Fair Comparison of Post-Training Quantization Methods for Large Language Models
- 公平对比所有主流PTQ方法，统一校准/eval框架
- 控制变量，消除不同论文eval设置不一致带来的偏差
---
## 三、Benchmark与评测
### 通用量化评测
- **LLM-QBench**：一个专门量化LLM量化的benchmark
  - 多维度评测：PPL、下游任务、长上下文、速度、显存
  - https://github.com/（搜索LLM QBench）
- **LM-Eval-Harness**：EleutherAI标准评测框架
  - 所有量化论文都用这个测MMLU/GSM8K/HumanEval等
  - https://github.com/EleutherAI/lm-evaluation-harness
### 长上下文评测
- **LongBench**：长上下文理解benchmark（128k以内）
- **InfiniteBench**：超长上下文（1M+）
- **Needle In A Haystack**：大海捞针测试，简单有效测长上下文召回
### Agent/工具调用评测
- **GAIA**：通用Agent助手benchmark
- **WebArena**：网页Agent
- **ToolBench**：工具调用
- **AgentBench**：多维度Agent能力评测
- 量化对Agent能力的影响需要用这些benchmark，不是PPL
---
## 四、论文写作参考
### 论文结构参考（优秀量化论文怎么写）
- **SmoothQuant**：问题→insight→方法→实验，简单清晰，典范
- **AWQ**：发现salient weights，故事讲得好
- **QuIP#**：理论+实验结合，incoherence processing讲得清楚
- **QServe**：系统顶会的写法，算法+kernel+系统集成+评测
### 常见审稿问题（提前准备）
1. "为什么不用xx方法？" → 你必须和最新SOTA对比
2. "只测了7B，70B上效果如何？" → 必须测至少一个大模型
3. "速度是多少？实际部署能加速吗？" → 不能只讲精度，必须报速度
4. "为什么xx场景掉点？" → 诚实承认局限性，做error analysis
5. "你的方法和xx有什么区别？" → related work要讲清楚
---
## 五、必读博客/技术报告
### 工业界技术博客
- **vLLM Blog**：vLLM官方博客，讲他们怎么集成量化
- **NVIDIA Developer Blog**：FP8/TensorRT-LLM量化
- **Meta AI Blog**：TorchAO/BitNet相关
- **TensorRT-LLM Documentation**：生产级量化指南
- **llama.cpp**：GGUF格式说明，k-quants原理
### 技术博客推荐
- **"Quantization in Practice"**：MLC/TVM团队的量化实践
- **"Making LLMs even more accessible with bitsandbytes, 4-bit quantization and QLoRA"**：HuggingFace
- **"AWQ: Activation-aware Weight Quantization"**：MIT Han Lab博客
- **"SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models"**：MIT Han Lab
---
## 六、代码资源
### 量化库/工具
| 工具 | 用途 | 链接 |
|------|------|------|
| **AutoGPTQ** | GPTQ量化实现 | https://github.com/PanQiWei/AutoGPTQ |
| **AutoAWQ** | AWQ量化实现 | https://github.com/casper-hansen/AutoAWQ |
| **GPTQModel** | 下一代GPTQ，支持多种格式 | https://github.com/ModelCloud/GPTQModel |
| **HQQ** | Half-Quadratic Quantization，零样本快速 | https://github.com/mobiusml/hqq |
| **TorchAO** | PyTorch官方量化库 | https://github.com/pytorch/ao |
| **bitsandbytes** | NF4/8bit QLoRA | https://github.com/TimDettmers/bitsandbytes |
| **llama.cpp** | GGUF量化+CPU/GPU推理 | https://github.com/ggerganov/llama.cpp |
| **QuIP#** | 2bit Hadamard+格点量化 | https://github.com/facebookresearch/quip-sharp |
| **AQLM** | 加性量化2bit | https://github.com/Vahe1994/AQLM |
| **Marlin** | 最快W4 kernel | https://github.com/IST-DASLab/marlin |
| **SmoothQuant** | W8A8 | https://github.com/mit-han-lab/smoothquant |
| **OmniQuant** | 可学习PTQ | https://github.com/OpenGVLab/OmniQuant |
| **vLLM** | 推理引擎，支持大多数量化 | https://github.com/vllm-project/vllm |
| **SGLang** | 推理引擎，RadixAttention | https://github.com/sgl-project/sglang |
| **TensorRT-LLM** | NVIDIA官方推理引擎 | https://github.com/NVIDIA/TensorRT-LLM |
| **MLC LLM** | 多端部署量化+编译 | https://github.com/mlc-ai/mlc-llm |
| **ExLlamaV2** | 消费卡极速W4/W3/W2 | https://github.com/turboderp/exllamav2 |
| **lm-eval-harness** | 评测框架 | https://github.com/EleutherAI/lm-evaluation-harness |
---
## 七、跟踪最新论文
### arXiv分类
- **cs.LG**：Machine Learning
- **cs.CL**：Computation and Language（NLP/LLM）
- **cs.AR**：Hardware Architecture
- **cs.DC**：Distributed/Systems
- **cs.AI**：Artificial Intelligence
### 关键词订阅（arxiv-sanity/Google Scholar Alert）
- "large language model quantization"
- "post-training quantization LLM"
- "KV cache compression/quantization"
- "low-bit quantization"
- "hardware-aware quantization LLM"
- "quantization aware training LLM"
### 会议截稿时间参考（每年）
| 会议 | 大致截稿 |
|------|---------|
| ICLR | 9-10月 |
| ICML | 1-2月 |
| NeurIPS | 5-6月 |
| ACL | 12月 |
| EMNLP | 6月 |
| MLSys | 10月/4月 |
| OSDI/NSDI | 系统顶会，看官网 |
| ISCA/MICRO | 体系结构顶会 |
| ASPLOS | 11月左右 |
| COLM | 新LLM专属会议 |
---
## 入门路径建议
如果你刚进入LLM量化方向：
### 第1-2周：建立基础
1. 读一篇好的Survey，了解领域全貌
2. 跑通bitsandbytes/AutoAWQ在Llama-2/3-7B上，量化一个模型玩玩
3. 读SmoothQuant论文，理解outlier问题
### 第3-4周：理解PTQ
1. GPTQ → AWQ → SpQR → HQQ
2. 复现一个简单W4 PTQ方法（用Hessian或者AWQ的salient权重缩放）
3. 用lm-eval测MMLU/PPL，对比baseline
### 第5-6周：深入一个方向
- 对系统/硬件感兴趣→看Marlin kernel，动手改一个简单kernel
- 对KV/长上下文感兴趣→复现KIVI/KVQuant，测大海捞针
- 对低bit感兴趣→试QuIP#/AQLM/BiLLM，看2bit效果
- 对QAT感兴趣→跑LoRA-QAT，看比PTQ提升多少
### 第7-8周：找insight，开始自己的工作
1. 做实证分析，把现有方法在你关心的场景跑一遍
2. 发现问题："为什么xx情况下所有方法都不行？"
3. 提出你的解法，做原型看有没有效果
### 持续跟进
- 每周扫一次arxiv cs.CL/cs.LG新论文
- 关注顶会accepted论文列表（NeurIPS/ICML/ICLR每年收很多量化论文）
---
## 常见误区
1. **只看PPL不看下游任务**：PPL和下游任务相关性不强，尤其是Agent/代码/数学
2. **只测小模型不测大模型**：7B上work的方法70B不一定work
3. **只讲精度不讲速度**：量化的目的是快和省显存，光有精度没有速度up是没用的
4. **不对比最新SOTA**：现在量化发展很快，每季度都有新SOTA，投稿前一定要查最新arxiv
5. **只在短对话上测**：长上下文/多轮对话/Agent场景误差累积更严重
6. **过度claim"通用"**：说"我们的方法在所有场景都最好"几乎不可能，诚实说明适用场景和局限性
