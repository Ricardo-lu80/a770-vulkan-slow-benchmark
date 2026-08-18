# Benchmark 原始测试日志

测试环境：Intel Arc A770 8GB · 驱动 32.0.101.8801 · Windows 10 22H2 · llama.cpp b10483 (Vulkan)
统一参数：`-ngl 99 --flash-attn on --cache-type-k q4_1 --cache-type-v q4_1 --parallel 1 -c 2048`

> 日志截取自 llama-server 启动后的 `print_timing` 输出，字段含义：
> `prompt eval time` = prefill（输入处理）；`eval time` = decode（生成），即上表"生成速度"来源。

---

## 1. Qwen3-4B · Q4_K_M · 2.50 GB → 51.7 t/s

```
prompt eval time =     848.89 ms /    19 tokens (   44.68 ms per token,    22.38 tokens per second)
       eval time =    3848.42 ms /   200 tokens (   19.34 ms per token,    51.71 tokens per second)
```

---

## 2. Qwen3-8B · Q3_K_S · 3.77 GB → 28.7 t/s

```
prompt eval time =     698.67 ms /    19 tokens (   36.77 ms per token,    27.19 tokens per second)
       eval time =    6931.37 ms /   200 tokens (   34.83 ms per token,    28.71 tokens per second)
```

---

## 3. Qwen3.5-9B · Q3_K_S · 4.02 GB → 29.9 t/s

```
prompt eval time =    1038.94 ms /    19 tokens (   54.68 ms per token,    18.29 tokens per second)
       eval time =    6657.27 ms /   200 tokens (   33.45 ms per token,    29.89 tokens per second)
```

---

## 4. Qwen3-8B · Q4_K_M · 5.00 GB → 8.1 t/s

```
prompt eval time =     228.20 ms /    19 tokens (   12.01 ms per token,    83.26 tokens per second)
       eval time =   24637.17 ms /   200 tokens (  123.80 ms per token,     8.08 tokens per second)
```

---

## 5. Qwen3.5-9B · Q4_K_M · 5.68 GB → 8.1 t/s

```
prompt eval time =     404.44 ms /    19 tokens (   21.29 ms per token,    46.98 tokens per second)
       eval time =   24496.61 ms /   200 tokens (  123.10 ms per token,     8.12 tokens per second)
```

---

## 附加观察

- 第 4、5 项权重超过 ~4.5GB，decode 速度骤降至 8 t/s，而 prefill 仍正常（47-83 t/s）
- 控制变量验证：上下文 32K → 2K 速度不变；FA 开/关速度不变；投机解码（ngram/MTP）中文场景无提升
