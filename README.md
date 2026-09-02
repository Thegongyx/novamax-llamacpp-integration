# novamax-llamacpp-integration

把本地编译好的自定义 llama.cpp 版本接入 NovaMax（AMD 平台）的完整方案与可用引擎资源包。

## 目录结构

```
novamax-llamacpp-integration\
├── README.md                            ← 本文件（包说明 / 快速上手）
├── docs\
│   └── novamax-llamacpp-skill\
│       └── SKILL.md                     ← 详细技能文档（流程 + 常见坑）
├── rocm_w4a4\                           ← 引擎样本 A：HIP/W4A4 版 (rocm)
├── roc_rocmfp4\                         ← 引擎样本 B：HIP/ROCmFP4 版 (rocm)
└── vulkan_qwen4exp\                     ← 引擎样本 C：Vulkan/qwen4exp 版 (vulkan)
```

## 三个引擎一览

| 引擎目录 | 来源 | variant 归属 | 文件数 | llama-server.exe |
|---|---|---|---|---|
| `rocm_w4a4` | ROCmFPX / bin-w4a4 | rocm | 14 | ~4.2MB（单文件） |
| `roc_rocmfp4` | ROCmFPX / bin-rocmfp4 | rocm | 14 | ~4.2MB（单文件） |
| `vulkan_qwen4exp` | LaurentZuijdwijk / build-vulkan | vulkan | 12 | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |

> 三个引擎都已**实测可用**：从各自目录运行 `llama-server.exe --list-devices` 均能列出 GPU（`ROCm0` / `Vulkan0: AMD Radeon 8060S`），依赖齐全。

## 快速上手：接入 NovaMax

### 1. 拷贝引擎到 NovaMax 引擎目录

variant 归属决定目标目录：

| 引擎 | 目标目录 |
|---|---|
| `rocm_w4a4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `roc_rocmfp4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `vulkan_qwen4exp` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\`（目录可能需新建） |

```powershell
# 例：把 vulkan_qwen4exp 接入 NovaMax
$dst = "C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\vulkan_qwen4exp"
New-Item -ItemType Directory -Path $dst -Force
Copy-Item -Path ".\vulkan_qwen4exp\*" -Destination $dst -Recurse -Force
```

### 2. 确保 `.installed` 存在且正确

每个引擎目录根必须有 `.installed` 文件（本包已带）：

```json
{
  "runtime": null,
  "installed_at": "2026-09-02T00:05:40.020Z",
  "version": "vulkan_qwen4exp",
  "archiveSha256": "local-vulkan-qwen4exp-custom-build",
  "variant": "vulkan",
  "engine": "llamacpp"
}
```

- `version` 须与目录名一致
- `variant` 须与 backend 对应（rocm / vulkan）
- `archiveSha256` 用于区分本地/官方（本地自定义填 `local-*`）

### 3. 重新扫描 / 重载

引擎放好后，在 NovaMax UI 重新扫描/重载（或重启后端），即可在模型参数里把 `engine_version` 选为该引擎。

## 引擎发现原理（关键）

NovaMax 发现本地引擎**不靠 engines.json**，而是扫描磁盘：

```
遍历 llamacpp 的 variants(rocm/vulkan/cuda/other)
→ external\llamacpp\<variant>\
→ 枚举子目录
→ _isValidEngineDir 校验 → 读 .installed → 提取 version
```

只需：**正确 variant 目录 + 有效 `.installed`**。

## 模型参数放哪

模型启动参数存在 `C:\Linglong\NovaStudio\NovaMax\data\novamax.db`（SQLite）的 `models` 表 `user_parameters` 字段，NovaMax 据此生成 llama-server 命令行。

详细流程、参数说明 → 见 `docs\novamax-llamacpp-skill\SKILL.md`。

## 各引擎适配的模型（HF 镜像链接）

### 引擎适配矩阵（ROCmFPX 后端含两个引擎，需区分）

| 引擎目录 | 后端 | 适配模型 | HF 镜像仓库链接 | 满频实测 |
|---|---|---|---|---|
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST（MTP） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-GGUF | — |
| `rocm_w4a4` | HIP/W4A4 | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Qwen3.8-27B-ROCmFPX-GGUF | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |
| `rocm_w4a4` | HIP/W4A4 | Qwen3.8-27B-ROCmFPX-GGUF | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-GGUF | — |

> **两个 ROCmFPX 引擎的区别**：
> - `roc_rocmfp4`（bin-rocmfp4）= ROCmFP4 变体，适配 ROCmFPX 量化模型（Qwen3.8-27B-ROCmFPX-GGUF、Ornith-1.5-35B-A3B-ROCmFP4-GGUF）。lemonade 测试时 `rocm_bin` 指向 `bin-rocmfp4\llama-server.exe`。
> - `rocm_w4a4`（bin-w4a4）= W4A4 变体（Q4_0 权重 + 高精度激活），适配 Q4_0 权重模型（Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0）。
>
> 镜像域名用 hf-mirror.com（国内直连）。lemonade 拉取时需 `HF_ENDPOINT=https://hf-mirror.com` + 显式 `--source huggingface`。

## 满电源频率速度测试（lemonade 重新实测）

**测试条件**：
- **电源/频率**：AMD 电源档位拉满（**整机功耗约 132W**）
- **提示词长度**：~1 万 token 长上下文（散文 10158 / 代码 9979 token，固定填充文本构造）
- **输出长度**：`max_tokens=3000`，实际连续输出约 2772-3000 token；decode 仅在累计 ≥1000 token 后取 [1000, 末尾] 稳定段（约 2000 token）统计
- **场景**：散文/代码 × 思考模式 off/low
- **加载方式**：通过 lemonade 加载模型，每次场景独立冷加载（load → 测 → unload），杜绝 KVCache 复用污染

> **方法改进（重要）**：旧数据中 think-low 的 PREFILL 虚高（如 17087 t/s）是 KV cache 复用假象。本版**每场景独立冷加载**，彻底消除 cache 污染，PREFILL 均为真实冷启动速度。

### Vulkan 后端（LaurentZuijdwijk build-vulkan）

**Qwen3.8-27B-ROCmFP4-FAST（DFlash2）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 294.0 t/s | 17.4 tok/s |
| 代码 think-off | 295.2 t/s | 32.9 tok/s |
| 散文 think-low | 294.2 t/s | 24.2 tok/s |
| 代码 think-low | 292.5 t/s | 25.0 tok/s |

**Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 286.6 t/s | 17.4 tok/s |
| 代码 think-off | 291.0 t/s | 41.3 tok/s |
| 散文 think-low | 289.3 t/s | 18.3 tok/s |
| 代码 think-low | 289.9 t/s | 24.4 tok/s |

### ROCmFPX 后端（charlie12345）—— 两个引擎分别归属

> `rocm_w4a4`（bin-w4a4）= W4A4 变体，适配 Q4_0 权重模型（ROCmI4）；`roc_rocmfp4`（bin-rocmfp4）= ROCmFP4 变体，适配 ROCmFPX 量化模型。测试中按模型正确切换 `rocm_bin`。

**Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0 —— `rocm_w4a4` 引擎**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 439.6 t/s | 18.9 tok/s |
| 代码 think-off | 437.4 t/s | 36.6 tok/s |
| 散文 think-low | 434.6 t/s | 20.2 tok/s |
| 代码 think-low | 431.5 t/s | 40.6 tok/s |

**Qwen3.8-27B-ROCmFPX-GGUF —— `roc_rocmfp4` 引擎**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 246.3 t/s | 13.6 tok/s |
| 代码 think-off | 251.1 t/s | 28.5 tok/s |
| 散文 think-low | 249.9 t/s | 14.9 tok/s |
| 代码 think-low | 250.0 t/s | 16.3 tok/s |

**Ornith-1.5-35B-A3B-ROCmFP4-GGUF —— `roc_rocmfp4` 引擎（MoE）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | **1248.7 t/s** | **86.4 tok/s** |
| 代码 think-off | **1246.4 t/s** | **82.0 tok/s** |
| 散文 think-low | **1239.1 t/s** | **85.9 tok/s** |
| 代码 think-low | **1246.6 t/s** | **79.6 tok/s** |

> **测试方法**：lemonade 加载模型（冷启动）→ 发 ~1 万 token 长上下文流式请求 → 首个 token 前计 PREFILL（prompt_tokens/首token时间）→ 累计输出 ≥1000 token 后取 [1000, 末尾] 稳定段计 DECODE → unload。每场景重新加载，保证无 KVCache 干扰。
> **备注**：
> - rocm 后端两引擎的 PREFILL 明显优于 Vulkan（430-440 vs 290 t/s），decode 代码场景最快（ROCmI4 代码 36.6-40.6 tok/s）；ROCmFPX-GGUF 整体最慢。
> - **Ornith-1.5-35B-A3B 是 MoE 模型**（35B 总参 / 3B 激活），prefill ~1250 t/s、decode ~80-86 tok/s，**远快于所有 dense 27B 模型**（约 3-5 倍）。这是本套引擎+硬件上的最快配置。
