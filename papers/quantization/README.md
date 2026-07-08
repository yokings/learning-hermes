# 大模型量化论文集（2024-2026）
> 近两年顶会（NeurIPS/ICML/ICLR/ACL/EMNLP/OSDI/NSDI/MLSys/ISCA/MICRO）大模型量化论文分类整理
> 创建日期：2026-07-07

---

## 分类目录

| 目录 | 主题 | 论文数 |
|------|------|--------|
| [01-Weight-only量化](01-Weight-only量化/) | 仅权重量化（W4A16/W8A16等），activation保持FP16 | 经典+最新SOTA |
| [02-W8A8联合量化](02-W8A8联合量化/) | Weight和Activation都量化到8bit，全INT8推理 | |
| [03-KV-Cache量化](03-KV-Cache量化/) | KV Cache压缩，长上下文场景关键 | 🔥 当前热点 |
| [04-Sub4bit低比特量化](04-Sub4bit低比特量化/) | 3bit/2bit/1bit/三元/二值化极限压缩 | |
| [05-混合精度自动化量化](05-混合精度自动化量化/) | AutoML搜索、混合精度、非均匀量化 | |
| [06-量化感知训练QAT](06-量化感知训练QAT/) | Quantization-Aware Training，精度更高 | |
| [07-场景专用量化](07-场景专用量化/) | Agent/多模态/LoRA/长上下文/端侧/投机采样专用 | |
| [08-硬件算法协同](08-硬件算法协同/) | CUDA核、硬件友好量化、编译器协同 | 系统方向 |
| [09-系统集成推理引擎](09-系统集成推理引擎/) | vLLM/SGLang/TensorRT-LLM等引擎中的量化实现 | |
| [10-综述与实证研究](10-综述与实证研究/) | Survey、Empirical Study、Benchmark | |

---

## 快速开始阅读顺序建议

1. 先看 **[10-综述与实证研究](10-综述与实证研究/)** 建立整体认知
2. 再看经典基础：GPTQ → AWQ → SmoothQuant → QLoRA
3. 然后看最新SOTA：QuIP# → AQLM → Atom → VS-Quant
4. 当前热点方向：**[03-KV-Cache量化](03-KV-Cache量化/)** 和 **[07-场景专用量化](07-场景专用量化/)** 机会最多
5. 系统方向：**[08-硬件算法协同](08-硬件算法协同/)** 和 **[09-系统集成](09-系统集成推理引擎/)** 含金量高
