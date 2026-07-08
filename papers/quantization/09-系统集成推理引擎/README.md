# 09 - 系统集成：推理引擎中的量化实现
> vLLM/TensorRT-LLM/SGLang/llama.cpp等主流推理引擎中的量化方案
> 了解工业界实际怎么部署量化，生产环境需要考虑什么
---
## 为什么看推理引擎？
论文里的方法很多是研究原型，真正生产部署用的是推理引擎里的实现：
- 支持continuous batching、PagedAttention、speculative decoding等系统优化
- 量化和这些系统优化深度结合
- 稳定性、正确性、性能都是生产级别的
---
## 一、vLLM
- **GitHub**：https://github.com/vllm-project/vllm
- **机构**：UC Berkeley → 社区开源 → 现在是最流行的开源推理引擎
### 支持的量化方法
| 量化 | 支持情况 | 说明 |
|------|---------|------|
| FP8 | ✅ 成熟 | Hopper/Ada支持，W8A8 FP8 E4M3，KV Cache也支持FP8 |
| W8A8 INT8 | ✅ 成熟 | SmoothQuant实现 |
| W4A16 GPTQ | ✅ 成熟 | AutoGPTQ格式支持，Marlin kernel |
| W4A16 AWQ | ✅ 成熟 | AutoAWQ，Marlin优化 |
| W4A16 Marlin | ✅ 最快 | 目前最快的W4 kernel |
| W4A16 GPTQModel | ✅ | 下一代GPTQ实现 |
| W8A8 KV Cache | ✅ | 实验性→逐渐成熟 |
| FP8 KV Cache | ✅ | 生产可用 |
| QuIP# | 🟡 实验性 | 2bit/4bit E8格点 |
| AQLM | 🟡 实验性 | 加性量化2bit |
| BitsAndBytes | ✅ | 4bit/8bit QLoRA风格，用于训练/轻量部署 |
### 量化相关核心代码
- `vllm/model_executor/layers/quantization/` - 量化实现目录
- `vllm/_custom_ops.py` - 自定义量化kernel入口
### vLLM量化特点
- 支持**PagedAttention + 量化**：KV量化和分页结合
- 支持**continuous batching**同时跑量化模型
- 支持**Prefix Caching**和量化共存（但KV量化可能破坏Prefix字节）
- 社区活跃，新量化方法很快集成
---
## 二、TensorRT-LLM
- **GitHub**：https://github.com/NVIDIA/TensorRT-LLM
- **机构**：NVIDIA官方，性能最强（NVIDIA GPU上）
### 支持的量化方法
| 量化 | 支持情况 | 说明 |
|------|---------|------|
| FP8（E4M3/E5M2） | ✅ 官方优化 | H100/H200/RTX40/Blackwell原生支持 |
| INT8 SmoothQuant | ✅ 官方优化 | W8A8生产级，延迟最低 |
| INT4 Weight-only | ✅ 官方优化 | 高度优化的INT4 kernel |
| INT4 AWQ | ✅ 支持 | AWQ算法+优化kernel |
| INT4 GPTQ | ✅ 支持 | GPTQ格式支持 |
| FP8/INT8 KV Cache | ✅ 生产级 | KV量化最成熟的实现之一 |
| W4A8 KV8 (QServe风格) | ✅ 最新 | 全栈低bit |
### 特点
- NVIDIA自家，对自家GPU优化到极致
- FP8/INT8 kernel是所有引擎里最快的
- 支持in-flight batching、paged KV、speculative decoding
- 官方做了大量benchmark，性能参考标准
- 缺点：编译复杂，NVIDIA-only，模型支持需要手动转换
---
## 三、SGLang
- **GitHub**：https://github.com/sgl-project/sglang
- **机构**：Microsoft/加州大学等，RadixAttention是核心创新
### 支持的量化方法
类似vLLM，额外特点：
- **RadixAttention**：前缀树KV缓存，跨请求自动复用KV
- KV缓存量化时前缀复用需要特殊处理（这也是Hermes Prefix Cache遇到的问题）
- FP8/INT8/W4A16 AWQ/GPTQ都支持
- 对长上下文+量化优化更好
### 核心创新：RadixAttention和量化兼容性
- 自动复用不同请求间的相同前缀KV
- 如果KV量化了，复用需要保证量化后的字节一致
- 这是现在研究的热点问题
---
## 四、llama.cpp / GGUF
- **GitHub**：https://github.com/ggerganov/llama.cpp
- **特点**：C/C++实现，跨平台（CPU/GPU/手机/Web），GGUF格式是事实标准
### GGUF量化格式
| 格式 | 位宽 | 说明 |
|------|------|------|
| F16/F32 | 16/32 | 原始精度 |
| Q8_0 | 8bit | 快速，精度损失极小，INT8 |
| Q6_K | 6bit | 高精度，接近Q8_0 |
| Q5_K_M | 5bit | 高质量，推荐 |
| Q5_K_S | 5bit | 速度快一点 |
| Q4_K_M | 4bit | **推荐**，平衡质量/大小/速度，最常用 |
| Q4_K_S | 4bit | 速度优化 |
| Q3_K_M | 3bit | 小体积，质量还可以 |
| Q3_K_S | 3bit | 极小体积 |
| Q2_K | 2bit | 极限压缩，质量损失明显 |
| IQ2_XS/IQ2_S/IQ2_M | 2bit | 改进的2bit量化，质量比Q2_K好 |
| IQ1_S/IQ1_M | 1bit | 极限压缩，用于70B在小内存上跑 |
### GGUF量化特点
- **K-quants**：混合精度量化，不同部分不同bit
- 纯CPU推理也很快，ARM NEON/Apple Metal/NVIDIA CUDA/AMD HIP/Vulkan都支持
- GGUF格式现在是开源模型的标准发布格式之一
- 不需要校准数据，量化速度快，离线工具链完善
- 端侧/CPU部署首选
---
## 五、Text Generation Inference (TGI)
- **机构**：Hugging Face
- 企业级部署用
- 支持bitsandbytes、GPTQ、AWQ、FP8、INT8
- 和HF生态集成最好
- 生产级稳定性
---
## 六、ExLlamaV2
- **GitHub**：https://github.com/turboderp/exllamav2
- 社区项目，专注于**极速W4/W3/W2推理**
- 自己写的高度优化CUDA kernel，消费级显卡（3090/4090）上速度非常快
- 支持混合位宽（同一模型不同层不同bit）
- ExLlamaV2格式量化质量高，速度快
- 很多本地推理前端（oobabooga/SillyTavern）用它做后端
---
## 七、MLC LLM
- **机构**：CMU (Apache TVM)
- 目标：**一次编译，到处运行**（GPU/CPU/手机iPhone/Android/Web）
- 基于TVM Unity编译器
- 量化+编译协同优化
- 移动端部署首选之一
- 支持INT4/INT8/FP8
---
## 八、BitsAndBytes
- **GitHub**：https://github.com/TimDettmers/bitsandbytes
- Tim Dettmers维护，QLoRA作者
- 特点：
  - NF4（NormalFloat4）QLoRA格式
  - 8bit/4bit矩阵乘法
  - 主要用于**微调/轻量推理**，训练场景用得最多
  - 集成在HuggingFace Transformers里，一行代码开启
- 不是最快的，但**最易用**
---
## 九、TorchAO
- **机构**：Meta官方
- PyTorch原生量化库
- 支持W4/W8/FP8/AWQ/GPTQ/Marlin
- 和PyTorch 2.x torch.compile深度集成
- 不用额外依赖，PyTorch自带
- 发展很快，未来可能成为主流
---
## 十、生产部署选择参考
| 场景 | 推荐引擎 | 推荐量化 |
|------|---------|---------|
| **NVIDIA H100/H200云服务** | TensorRT-LLM 或 vLLM | FP8 W8A8 FP8KV |
| **RTX 4090/3090消费卡本地推理** | vLLM 或 ExLlamaV2 | W4A16 Marlin/AWQ |
| **CPU/Apple Silicon本地推理** | llama.cpp | Q4_K_M GGUF |
| **手机/边缘端部署** | MLC LLM 或 llama.cpp | 3bit/4bit 端侧优化 |
| **长上下文（128k+）服务** | SGLang | W4A16 + FP8 KV |
| **微调/研究** | Transformers + BitsAndBytes | NF4 QLoRA |
| **快速原型开发** | Transformers + TorchAO | W4/W8 开箱即用 |
---
## 量化部署常见坑
### 1. 量化和Prefix Caching冲突
- Hermes/Hermes Agent/SGLang的Prefix Caching要求前缀部分字节一致才能命中
- KV量化可能改变KV值，导致缓存命中率下降
- 解决方案：前缀部分用高精度/不量化，或者量化算法保证可重现
### 2. 量化和Speculative Decoding冲突
- 量化后draft model和target model分布不一致
- 拒绝率上升，速度up缩水
- 需要感知量化的投机采样
### 3. KV量化长上下文精度崩塌
- 128k以内没问题，1M以上误差累积严重
- 需要周期校准/误差补偿
### 4. 工具调用/Agent能力掉点
- PPL看着没问题，但工具调用JSON格式错率上升
- 生产环境必须测实际Agent任务，不能只看PPL
### 5. 长生成vs PPL不一致
- PPL是prefill的指标，长生成（decode）误差累积掉点更多
- 必须测长生成的case
---
## 系统方向论文参考（来自工业界）
| 论文 | 会议 | 引擎 | 内容 |
|------|------|------|------|
| **Efficiently Scaling Transformer Inference (PagedAttention)** | OSDI 2023 | vLLM | PagedAttention内存管理，KV Cache分页 |
| **SGLang: Efficient Execution of Structured LLM Programs** | ASPLOS 2025 | SGLang | RadixAttention前缀缓存，自动复用KV |
| **TensorRT-LLM** | 白paper | TensorRT-LLM | NVIDIA官方推理引擎 |
| **QServe** | ISCA 2025 | 研究 | W4A8KV4系统级量化 |
| **FlashAttention-2** | ICLR 2024 | 全引擎 | 快速注意力kernel，基础组件 |
| **FlashDecoding++** | ASPLOS 2025 | 全引擎 | KV分块+量化融合 |
| **SpecInfer/Speculative Decoding** | OSDI/NSDI | 全引擎 | 投机采样和量化结合 |
---
## 阅读源码建议
如果想了解工业界怎么实现量化：
1. **先看llama.cpp**：纯C，没有CUDA黑魔法，最容易读，理解GGUF量化格式
2. **再看vLLM的quantization目录**：Python实现，接口清晰，看怎么和PagedAttention/continuous batching集成
3. **最后看Marlin kernel**：学习怎么写高性能W4 CUDA kernel
4. **TensorRT-LLM不开源kernel**，只能看文档和API，参考它怎么设计量化API
