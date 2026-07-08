# 06 - 量化感知训练（QAT, Quantization-Aware Training）
> 量化时微调模型，让模型适应量化噪声，精度显著高于PTQ
> 代价是需要训练资源/数据，但精度天花板更高
---
## 背景：PTQ vs QAT
| 方法 | 是否需要训练 | 数据需求 | 精度 | 成本 | 适用场景 |
|------|------------|---------|------|------|---------|
| **PTQ**（Post-Training） | ❌ 不需要 | 少量校准数据（128-2048条） | 良好（W4/W8可用，低bit掉点） | 几分钟到几小时，单卡 | 快速部署、开源模型量化 |
| **QAT**（Quantization-Aware Training） | ✅ 需要 | 几千到几十万条训练数据 | 优秀（W2/W1都能保持高精度） | 几小时到几天，多卡 | 追求极致精度、低bit部署、生产环境 |
PTQ是"事后补救"，给训好的模型做量化；QAT是"训练时就考虑量化"，让模型适应量化，低bit下优势特别明显。
---
## 经典QAT方法（CNN时代的基础）
这些方法思路被迁移到LLM：
1. **Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference** (Google, CVPR 2018)：Fake Quantization + Straight-Through Estimator(STE)，QAT开山之作
2. **LSQ: Learned Step Size Quantization** (ICLR 2020)：量化步长（scale）可学习
3. **PACT: Parameterized Clipping Activation** (ICLR 2018)：可学习截断阈值
这些CNN时代的基础思想（Fake Quantize + STE + 可学习scale/clipping）在LLM QAT中继续沿用。
---
## LLM QAT 早期工作
### 1. LLM-QAT: Data-Free Quantization Aware Training For Large Language Models
- **会议**：ICLR 2024
- **作者**：Yuzhang Shang et al.
- **核心贡献**：
  - 首个系统性LLM QAT方法
  - **无数据QAT**：用模型自身生成数据做训练，不需要真实训练数据
  - 同时量化权重、激活、KV Cache
  - W4A4 KV4精度接近FP16
- **链接**：https://arxiv.org/abs/2305.17888
### 2. PEQA: Parameter-Efficient Quantization Adaptation
- **会议**：ICML 2024
- **核心贡献**：
  - 参数高效QAT：只优化缩放因子，不更新所有权重
  - 训练成本比全参数QAT低很多
  - 适合轻量级快速适配
- **链接**：https://arxiv.org/abs/2310.10683
### 3. QLoRA本身不是QAT，但启发了LoRA-QAT
- QLoRA量化后做LoRA微调，但目标是任务适配，不是让模型适应量化
- LoRA-QAT把LoRA思想用到QAT，只训练LoRA参数让模型适应量化
---
## 2024-2025 最新SOTA QAT
### 4. SpinQuant-LLM-QAT
- SpinQuant虽然主要讲PTQ的旋转变换，也可以结合QAT进一步提升精度
### 5. QAT4LM: Quantization-Aware Training for Language Models
- **会议**：Preprint 2024
- 系统性QAT recipe，W4A4精度超过PTQ W8A8
- 少数据QAT，只用几千条数据就能达到很好效果
### 6. BitNet b1.58
- **会议**：Preprint 2024 (Microsoft)
- **核心贡献**：
  - **从头训练三元权重{-1,0,+1} LLM**（1.58bit）
  - 不是PTQ，是从零开始训练低bit模型
  - 性能与FP16 LLaMA相当，推理快、能耗低
  - 证明了低bit训练到收敛是可行的
- **链接**：https://arxiv.org/abs/2402.17764
### 7. BitNet a4.8: 4-bit Activation 8-bit Weight
- 比b1.58进一步，激活也4bit
- 训练recipe成熟，适合边缘部署
### 8. QuIP# + QAT
- QuIP#的Hadamard+格点初始化，再做少量QAT微调，W2精度进一步提升
### 9. LQER: Low-Rank Quantization Error Reconstruction
- **会议**：ICML 2024
- **核心贡献**：
  - 低秩残差补偿量化误差
  - 量化后加低秩adapter，训练adapter补偿误差
  - 训练成本比全参数QAT低很多
  - W2/W3效果显著提升
- **链接**：https://arxiv.org/abs/2402.08039
### 10. ZipLM: Structured Pruning + Quantization Aware Training
- 剪枝+量化联合QAT，压缩比更高
- 同时做稀疏和量化，协同优化
### 11. AQLM with QAT Finetuning
- AQLM码本初始化后做少量QAT微调，W2精度再上一个台阶
- 比纯PTQ AQLM提升明显
### 12. OmniQuant-LoRA
- OmniQuant的可学习变换 + LoRA微调，轻量QAT
- 只训LoRA参数，成本低，效果好
---
## LoRA-QAT：参数高效QAT
全参数QAT成本高，LoRA-QAT只训少量LoRA参数，性价比高：
### 13. QLoRA-QAT
- 量化后用LoRA微调，间接达到QAT效果
- 简单易行，社区广泛使用
### 14. LoftQ: LoRA-Fine-Tuning-Aware Quantization
- **会议**：ICLR 2024
- **核心贡献**：
  - 专门为LoRA微调设计的量化方法
  - 量化时考虑后续LoRA微调，交替优化量化器和LoRA
  - 量化+LoRA微调后精度接近FP16
- **链接**：https://arxiv.org/abs/2310.08659
### 15. QA-LoRA: Quantization-Aware Low-Rank Adaptation
- **会议**：ICLR 2024
- **核心贡献**：
  - 4bit双重量化 + LoRA
  - 训练时去量化，存储时量化
  - 单卡可训练65B模型
- **链接**：https://arxiv.org/abs/2309.14717
---
## 关键技术：Fake Quantization（伪量化）
QAT训练的核心技术是Fake Quantization：
```python
# 伪量化：前向传播模拟量化误差，反向传播用STE让梯度流过
class FakeQuantize(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x, scale, zero_point, bits=4):
        # 前向：模拟量化（dequant）
        x_int = torch.clamp(torch.round(x / scale) + zero_point, 0, 2**bits - 1)
        x_dequant = (x_int - zero_point) * scale
        return x_dequant  # 前向传播用"伪量化"的值，模拟量化噪声
    
    @staticmethod
    def backward(ctx, grad_output):
        # 反向：Straight-Through Estimator (STE)
        # 假装量化是恒等变换，梯度直接流过
        return grad_output, None, None, None
```
训练时前向模拟量化误差，反向梯度直接传（STE），模型权重还是FP16/FP32，训练完后真正量化。
---
## QAT关键技巧
| 技巧 | 作用 |
|------|------|
| **Straight-Through Estimator (STE)** | 让梯度能通过不可导的量化算子 |
| **可学习scale/zero_point** | LSQ/PACT思想，量化参数也训练 |
| **权重衰减/正则化** | 让权重分布更适合量化 |
| **冻结第一层/最后一层** | 这两层对量化特别敏感，保持FP16 |
| **学习率预热** | 初期学习率小，稳定训练 |
| **逐层QAT/Block-wise QAT** | 一次只训练一层，降低显存需求 |
| **蒸馏辅助** | 用FP16模型作为老师，引导QAT学生 |
| **KV Cache量化QAT** | KV也加入QAT，长上下文更准 |
---
## 训练成本对比
| 方法 | 7B模型训练成本 | 数据量 | 精度 |
|------|--------------|--------|------|
| PTQ | ~10分钟，单卡 | 128-2048条 | W4好，W2差 |
| LoRA-QAT | ~1-5小时，单卡 | 10k-100k条 | W4/W2很好 |
| 全参数QAT | ~10-50小时，8卡 | 100k-1M条 | W2/W1优秀 |
| 从头训练BitNet | ~类似LLaMA2训练成本，数百卡 | 万亿token | 1.58bit媲美FP16 |
---
## 未解决问题/研究机会
1. **少数据QAT**：能不能用几千条甚至几百条数据做QAT，达到全数据QAT效果？
2. **无数据QAT**：LLM-QAT开始了，能不能做到完全无数据（纯自生成）QAT效果和有数据媲美？
3. **QAT后的模型是否通用**：在通用数据上QAT后，下游任务/Agent/工具调用能力会不会掉？
4. **跨模型QAT**：能不能把一个模型QAT的知识迁移到另一个模型？
5. **KV Cache QAT**：现在KV量化PTQ为主，QAT能不能让2bit KV真正work？
6. **动态QAT**：训练时就学习混合精度策略，运行时自适应bit-width
7. **极低bit（1/1.58bit）QAT/从头训**：BitNet是从头训，能不能基于现有LLM QAT到1.58bit而不从头训？
---
## 开源工具
| 工具/仓库 | 支持QAT |
|----------|--------|
| **Transformers** + **PEFT** | 支持LoRA-QAT |
| **TorchAO** | Meta官方，支持各种QAT recipe |
| **TensorRT-LLM** | 支持QAT模型部署 |
| **llama.cpp** | 支持导入QAT模型 |
| **bitsandbytes** | QLora支持 |
| **GPTQModel** | 支持GPTQ+QAT微调 |
---
## 阅读建议
```
入门基础：
  先搞懂Fake Quantize + STE（看Google 2018那篇开山之作）
  然后看LSQ/PACT了解可学习量化参数
LLM QAT：
  LLM-QAT → PEQA → LoftQ/QA-LoRA
低bit从头训：
  BitNet b1.58（这是2024年影响力很大的工作）
进阶方向：
  LQER（低秩补偿）→ 结合混合精度+QAT
```
如果你的研究方向是**部署/端侧**，QAT比PTQ更有研究价值——PTQ已经很卷了，QAT尤其是低bit/少数据QAT还有很多机会。
