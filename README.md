<div align="center">

# Intel Arc A770 8GB · llama.cpp Vulkan 生成速度断崖现象

**实测基准数据：为什么 8B+ 模型在 A770 上只有 8 t/s，而小权重模型能到 51 t/s**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Windows%2010%2022H2-blue)](https://www.microsoft.com/windows)
[![GPU](https://img.shields.io/badge/GPU-Intel%20Arc%20A770%208GB-red)](https://www.intel.com)
[![Engine](https://img.shields.io/badge/Engine-llama.cpp%20b10483%20Vulkan-purple)](https://github.com/ggml-org/llama.cpp)

</div>

---

## 📌 概述

在使用 **Intel Arc A770 8GB** 通过 **llama.cpp Vulkan 后端** 运行本地大模型时，发现一个显著的速度断崖：

> 权重小于 **4GB** 的模型生成速度可达 **25–51 t/s**（流畅）；
> 权重超过 **4.5GB** 的模型生成速度**骤降至 8 t/s**（几乎不可用）。

本仓库记录了完整的实测数据、测试环境与复现步骤，供同配置用户参考。

---

## 🖥️ 测试环境

| 项目 | 配置 |
| :--- | :--- |
| 显卡 | Intel Arc A770 **8GB** |
| 显卡驱动 | 32.0.101.8801（最新稳定版） |
| CPU | AMD Ryzen 5 3600X（6C12T） |
| 内存 | 16GB DDR4 |
| 操作系统 | Windows 10 22H2 |
| 推理引擎 | llama.cpp b10483（Vulkan 后端） |

---

## 📊 实测数据

所有数据均为 **生成阶段（eval / decode）** 实测值，非 prefill（输入处理）速度。

| 模型 | 量化 | 权重大小 | 生成速度 | 带宽利用率* |
| :--- | :--- | :--- | :--- | :--- |
| Qwen3-4B | Q4_K_M | 2.50 GB | **51.7 t/s** | ~25% |
| Qwen3-8B | Q3_K_S | 3.77 GB | **28.7 t/s** | ~21% |
| Qwen3.5-9B | Q3_K_S | 4.02 GB | **29.9 t/s** | ~22% |
| Qwen3-8B | Q4_K_M | 5.00 GB | **8.1 t/s** | ~8% |
| Qwen3.5-9B | Q4_K_M | 5.68 GB | **8.1 t/s** | ~8% |

*\*带宽利用率 = 实测速度 ÷ 理论上限（512 GB/s ÷ 权重大小），估算值。*

### 统一测试参数

```bash
llama-server.exe -m <model.gguf> \
    -ngl 99 \
    --flash-attn on \
    --cache-type-k q4_1 \
    --cache-type-v q4_1 \
    --parallel 1 \
    -c 2048 \
    --host 127.0.0.1 \
    --port 8080
```

---

## 🔍 核心发现

### 速度断崖

```
生成速度 (t/s)
51.7 |■
     |
28.7 |■■
29.9 |■■
     |
 8.1 |■■■■■■■■■        ← 断崖
     |_____________
     2.5  3.8  4.0  5.0  5.7  权重 (GB)
```

- 权重 **< 4GB**：生成速度 25–51 t/s，流畅可用
- 权重 **> 4.5GB**：速度骤降至 **8 t/s**
- **拐点极其陡峭**，非平滑衰减

### 值得注意

- **prefill（输入处理）正常**：46–83 t/s，瓶颈仅存在于生成阶段
- 权重 3.77GB 与 4.02GB 均约 29 t/s，但 5.0GB 骤降——**拐点位于 4–4.5GB 之间**

---

## ❌ 已排除的假设

| 假设 | 验证结果 |
| :--- | :--- |
| GPU offload 配置错误 | ❌ 显卡满载、显存占用正常（4.7GB/8GB） |
| KV cache 过大导致溢出 | ❌ 上下文 32K → 2K，速度完全不变 |
| 模型架构差异（新/老架构） | ❌ Qwen3-8B（老）与 Qwen3.5-9B（新）表现一致 |
| CPU 成为瓶颈 | ❌ CPU 占用仅 ~10% |
| 内存不足 | ❌ 内存占用仅 ~30% |
| Flash Attention 开关 | ❌ 开关前后速度一致 |
| 投机解码（ngram/MTP） | ❌ 中文场景无提升；MTP 需官方带 MTP 头的模型 |

---

## ⚙️ 复现步骤

1. 下载 [llama.cpp b10483 Vulkan 版](https://github.com/ggml-org/llama.cpp/releases)（Windows）
2. 下载任意 Qwen 系 GGUF（如 [unsloth/Qwen3.5-9B-GGUF](https://modelscope.cn/models/unsloth/Qwen3.5-9B-GGUF)）
3. 按上方"统一测试参数"启动 `llama-server.exe`
4. 通过 OpenAI 兼容 API 请求，观察返回的 `timings.predicted_per_second` 字段

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.5","messages":[{"role":"user","content":"你好"}],"max_tokens":200}'
```

---

## 💡 讨论

A770（512 GB/s 带宽）的理论生成上限约为 **90 t/s**（8B 模型），实测仅 8 t/s，
带宽利用率约 **8%**。相比之下，同带宽级别下 NVIDIA CUDA 后端通常可达到 50–70% 利用率。

推测可能原因（未做源码级验证）：

1. **Vulkan 后端在 Intel 卡上缺少针对性算子优化**（llama.cpp 优化重心在 CUDA / Metal / ROCm）
2. **大权重 buffer 分配策略**：超过某阈值后 Vulkan 后端切换为分块/分页处理，每次生成需多次 GPU 提交与同步
3. **驱动层调度开销**随模型规模非线性放大

**欢迎在 Issues 中提供同配置/异配置的对比数据。**

---

## ✅ 结论

在 **A770 8GB + Windows + llama.cpp Vulkan** 环境下：

- **小权重（Q3_K_S 量化，<4GB）是能力与速度的最佳平衡点**（~29 t/s）
- 追求质量请使用 Q3_K_S 的 8B–9B 模型，而非 Q4 的大权重版本
- 若必须使用 5GB+ 权重，请考虑换用 CUDA（NVIDIA）或 SYCL（Linux + Intel oneAPI）后端

---

## 📄 License

MIT
