# novamax-llamacpp-integration

把本地编译好的自定义 llama.cpp 版本接入 NovaMax（AMD 平台）的完整方案与可用引擎资源包。

## 目录结构

```
novamax-llamacpp-integration\
├── README.md                            ← 本文件（包说明 / 快速上手 / 引用致谢）
├── docs\                                ← 文档
│   └── novamax-llamacpp-skill\
│       └── SKILL.md                     ← 详细技能文档（流程 + 常见坑）
└── novamax-adapters\                    ← NovaMax 适配引擎（含 .installed，可直接复用）
    ├── roc_official\                    ← 官方版：HIP/ROCm（官方 ROCmFPX 主线）
    ├── vulkan_official\                 ← 官方版：Vulkan（官方 ROCmFPX 主线）
    ├── rocm_w4a4\                       ← fork版：HIP/W4A4（charlie12345/ROCmFPX）
    ├── roc_rocmfp4\                     ← fork版：HIP/ROCmFP4（charlie12345/ROCmFPX）
    └── vulkan_qwen4exp\                 ← fork版：Vulkan/qwen4exp（LaurentZuijdwijk）
```

> 说明：`novamax-adapters\` 内每个引擎目录即完整编译产物 + `.installed` 适配标记（NovaMax 仅通过 `.installed` 识别引擎，见下文"引擎发现原理"）。原始编译产物与适配版仅差一个 `.installed` 文件，故不单独存放原件。

## 引擎一览

本包含 **5 个引擎**：2 个官方版（来自官方主线 `ROCmFPX/ROCmFPX`）+ 3 个 fork 版（来自 charlie12345/ROCmFPX 与 LaurentZuijdwijk 两个 fork）。

| 引擎目录 | 来源 | variant 归属 | llama-server.exe |
|---|---|---|---|
| `roc_official` | `ROCmFPX/ROCmFPX`（HIP，gfx1151） | rocm | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `vulkan_official` | `ROCmFPX/ROCmFPX`（Vulkan，gfx1151） | vulkan | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `rocm_w4a4` | charlie12345/ROCmFPX @ main（HIP/W4A4） | rocm | ~4.2MB（单文件） |
| `roc_rocmfp4` | charlie12345/ROCmFPX @ main（HIP/ROCmFP4） | rocm | ~4.2MB（单文件） |
| `vulkan_qwen4exp` | LaurentZuijdwijk/llama.cpp @ `vulkan/qwen4exp-rocmfpx` | vulkan | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |

> 引擎都已**实测可用**：从各自目录运行 `llama-server.exe --list-devices` 均能列出 GPU（`ROCm0` / `Vulkan0: AMD Radeon 8060S`），依赖齐全。

## 快速上手：接入 NovaMax

### 1. 拷贝引擎到 NovaMax 引擎目录

variant 归属决定目标目录：

| 引擎 | 目标目录 |
|---|---|
| `roc_official` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `vulkan_official` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\`（目录可能需新建） |
| `rocm_w4a4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `roc_rocmfp4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `vulkan_qwen4exp` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\`（目录可能需新建） |

```powershell
# 例：把 vulkan_official 接入 NovaMax
$dst = "C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\vulkan_official"
New-Item -ItemType Directory -Path $dst -Force
Copy-Item -Path ".\vulkan_official\*" -Destination $dst -Recurse -Force
```

### 2. 确保 `.installed` 存在且正确

每个引擎目录根必须有 `.installed` 文件（本包已带）：

```json
{
  "runtime": null,
  "installed_at": "2026-09-02T00:05:40.020Z",
  "version": "vulkan_official",
  "archiveSha256": "official-repo-vulkan-gfx1151",
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

### 引擎适配矩阵（5 引擎）

| 引擎目录 | 后端 | 适配模型 | HF 镜像仓库链接 | 满频实测 |
|---|---|---|---|---|
| `vulkan_official` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2 需 fork 支持） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_official` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（qwen4exp） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **官方用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`（单张量）** | ✅（无投机，见下） |
| `roc_official` | HIP | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_official` | HIP | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（qwen4exp） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **官方用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`（单张量）** | ✅（无投机，decode 慢，见下） |
| `roc_official` | HIP | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP，per-head） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **fork 用 `Qwen3.8-Flash-Next-ROCmFP4-FAST-v2-ple16.gguf`（per-head PLE）** | ✅（fork 支持 MTP） |
| `rocm_w4a4` | HIP/W4A4 | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Qwen3.8-27B-ROCmFPX-GGUF | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |

> **Flash-Next 文件选择（重要，与引擎绑定）**：`agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF` 仓库**同一权重、两种布局**，选哪份由引擎定：
> - **官方引擎（`vulkan_official`/`roc_official`）→ 用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`**（单张量 `per_layer_token_embd.weight`）——官方能加载，但**只能无投机**（MTP/ngram 均不可用，见下）。
> - **fork 引擎（`vulkan_qwen4exp`）→ 用 `Qwen3.8-Flash-Next-ROCmFP4-FAST-v2-ple16.gguf`**（per-head `ple_ngram_embd.0..15.weight`）——支持 **MTP 投机**。
> - MTP 外部草稿 `agentionai/Qwen3.8-Flash-Next-MTP-ROCmFP4-FAST-GGUF`（2.28GB）**仅 fork** 可用；官方加载该草稿报 `blk.0.hc_attn_norm.weight not found`。

> **兼容性说明**：
> - 官方版 `ROCmFPX/ROCmFPX` **不支持 `--spec-draft-adaptive`**（报 `invalid argument`），DFlash2/自适应投机需用 fixed `--spec-draft-n-max`；不支持 DFlash2 外部草稿（`--spec-type draft-dflash`）。
> - `roc_official` 与 `vulkan_official` 是同一官方源码的 HIP/Vulkan 两套构建，量化支持范围一致（Q4_0_ROCMFP4/FAST、ROCMI4、Q2-Q8_ROCMFPX 全系列）。
> - **Flash-Next（qwen4exp）在官方引擎上的适配**：官方源码 `src/models/qwen4exp.cpp` 只读取**单张量** `per_layer_token_embd.weight`；现行 HF per-head 文件 `...-v2-ple16.gguf`（张量 `ple_ngram_embd.0..15.weight`，Laurent fork 布局）官方**读不了**（`per_layer_token_embd.weight not found`）。**`v2/` 单张量** `...-imatrix-v2.gguf` 官方可加载（51.2B 参数单张 PLE 表自动溢出，无需 `--ngram-on-disk`）。**但官方引擎不支持 Flash-Next 的 MTP 外部草稿投机**：`-md Qwen3.8-Flash-Next-MTP-ROCmFP4-FAST.gguf --spec-type draft-mtp` 时草稿加载失败（`blk.0.hc_attn_norm.weight not found`，官方 qwen4exp 草稿按完整层 0..N 迭代，而草稿是 nextn-only 只有第 48 层；模型卡原文确认："plain serving of Flash-Next quants works on official, the external-drafter setup requires our fork"）。**ngram-cache 投机亦为负优化**（接受率 ~9%，decode 反降至 ~2 tok/s）。故官方引擎上 Flash-Next **只能无投机**（实测见速度测试表）；fork `vulkan_qwen4exp` 可跑 MTP（见 fork 历史数据）。
> - fork 版 `vulkan_qwen4exp` 支持 DFlash2 自适应投机；`rocm_w4a4`/`roc_rocmfp4` 分别侧重 W4A4 / ROCmFP4。
>
> 镜像域名用 hf-mirror.com（国内直连）。lemonade 拉取时需 `HF_ENDPOINT=https://hf-mirror.com` + 显式 `--source huggingface`。

## 满电源频率速度测试（官方版 + fork 版）

**测试条件**（与之前一致）：
- **电源/频率**：AMD 电源档位拉满（**整机功耗约 132W**）
- **提示词长度**：~1 万 token 长上下文（散文 10123 / 代码 9950 token，固定填充文本构造）
- **输出长度**：`max_tokens=3000`，实际连续输出约 3000 token；decode 仅在累计 ≥1000 token 后取 [1000, 末尾] 稳定段（约 2000 token）统计
- **场景**：散文/代码 × 思考模式 off/low
- **加载方式**：每次场景独立冷启动模型（spawn → 测 → kill），杜绝 KVCache 复用污染

> **方法说明**：本版测试改为**通过 lemonade 加载**（不再直接 spawn），启用模型卡推荐的 **MTP 投机解码**（官方引擎不支持 `--spec-draft-adaptive`，统一用固定 `--spec-draft-n-max`；lemonade 按模型 label 自动注入 `-md`/`--spec-type`）。因此官方版 decode 高于旧版"无投机 baseline"。PREFILL 取服务端 `timings.prompt_per_second`；DECODE 用流式逐 token 时间戳取 **[1000, 末尾] 稳定段**。每场景 `lemonade load → 测 → unload` 冷启动。

### HIP 引擎（`roc_official`）—— Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0（内嵌 MTP）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 434.8 t/s | 20.28 tok/s |
| 代码 think-off | 435.5 t/s | 33.52 tok/s |
| 散文 think-low | 435.1 t/s | 19.43 tok/s |
| 代码 think-low | 428.5 t/s | 22.45 tok/s |

### HIP 引擎（`roc_official`）—— Ornith-1.5-35B-A3B-ROCmFP4-GGUF（MoE，内嵌 MTP）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 1203.3 t/s | 59.59 tok/s * |
| 代码 think-off | 1218.4 t/s | 60.01 tok/s * |
| 散文 think-low | 1239.7 t/s | 62.43 tok/s |
| 代码 think-low | 1210.5 t/s | 101.82 tok/s |

### Vulkan 引擎（`vulkan_official`）—— Qwen3.8-27B-ROCmFP4-FAST（DFlash2 投机）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 112.1 t/s | 21.67 tok/s |
| 代码 think-off | 111.9 t/s | 33.30 tok/s |
| 散文 think-low | 108.0 t/s | 23.36 tok/s |
| 代码 think-low | 108.2 t/s | 32.33 tok/s |

### Vulkan 引擎（`vulkan_official`）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2（qwen4exp，v2 单张量，**无投机**）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 220.1 t/s | 21.53 tok/s |
| 代码 think-off | 199.1 t/s | 22.12 tok/s |
| 散文 think-low | 189.7 t/s | 21.06 tok/s |
| 代码 think-low | 197.8 t/s | 22.18 tok/s |

> 官方引擎不支持 Flash-Next 的 MTP（外部草稿加载失败）且 ngram-cache 为负优化，故官方上 Flash-Next **只能无投机**。参数：`-ngl 99 -fa on -ctk q8_0 -ctv q8_0 --no-mmap -b 2048 -ub 512`。fork `vulkan_qwen4exp` 可跑 MTP（见下方 fork 历史数据）。

### HIP 引擎（`roc_official`）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2（qwen4exp，v2 单张量，**无投机**）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 220.0 t/s | 5.09 tok/s |
| 代码 think-off | 208.1 t/s | 5.11 tok/s |
| 散文 think-low | 210.6 t/s | 5.10 tok/s |
| 代码 think-low | 206.2 t/s | 5.11 tok/s |

> **HIP decode 明显慢于 Vulkan**（~5.1 vs ~21-22 tok/s，约 4 倍）：v2 单张量的 51.2B 参数 PLE 表在 HIP 上落到 host 内存处理，吞吐瓶颈；Vulkan 自动溢出更优。故 Flash-Next 官方推荐用 **Vulkan 引擎**。同样只支持无投机（MTP 官方不可用）。

> **备注**：
> - 带 **\*** 的 Ornith think-off 两场景因模型提前停止（仅 281/448 token，未达 1000 token）未取到 [1000,end] 段，用的是服务端全段 `predicted_per_second`；其余场景均为稳定的 [1000,end] 稳定段。
> - **Qwen3.8-Flash-Next-imatrix（qwen4exp）**：官方引擎**只支持无投机**（MTP 外部草稿加载失败、ngram-cache 为负优化，见上方兼容性说明），实测见上方 Flash-Next 表；使用 `v2/` 单张量文件，参数含 `--no-mmap`。
> - **HIP prefill 显著优于 Vulkan**（~430 vs ~110 t/s，约 3.9 倍）；MoE 的 Ornith 最快（prefill ~1200 t/s，decode 59-102 tok/s）。
> - 本机电源档位拉满（整机功耗约 132W），条件与下方 fork 历史数据一致。

### fork 版引擎历史测速（旧数据，含投机解码）

> 说明：下方为 fork 版引擎的**历史实测**（lemonade 加载，含投机解码 `--spec-draft-adaptive`/DFlash2，与上方官方版"无投机 baseline"方法不同，故 decode 更高）。

**`vulkan_qwen4exp`（Laurent fork）—— Qwen3.8-27B-ROCmFP4-FAST（DFlash2）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 294.0 t/s | 17.4 tok/s |
| 代码 think-off | 295.2 t/s | 32.9 tok/s |
| 散文 think-low | 294.2 t/s | 24.2 tok/s |
| 代码 think-low | 292.5 t/s | 25.0 tok/s |

**`vulkan_qwen4exp`（Laurent fork）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 286.6 t/s | 17.4 tok/s |
| 代码 think-off | 291.0 t/s | 41.3 tok/s |
| 散文 think-low | 289.3 t/s | 18.3 tok/s |
| 代码 think-low | 289.9 t/s | 24.4 tok/s |

**`rocm_w4a4`（charlie fork）—— Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 439.6 t/s | 18.9 tok/s |
| 代码 think-off | 437.4 t/s | 36.6 tok/s |
| 散文 think-low | 434.6 t/s | 20.2 tok/s |
| 代码 think-low | 431.5 t/s | 40.6 tok/s |

**`roc_rocmfp4`（charlie fork）—— Qwen3.8-27B-ROCmFPX-GGUF**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 246.3 t/s | 13.6 tok/s |
| 代码 think-off | 251.1 t/s | 28.5 tok/s |
| 散文 think-low | 249.9 t/s | 14.9 tok/s |
| 代码 think-low | 250.0 t/s | 16.3 tok/s |

**`roc_rocmfp4`（charlie fork）—— Ornith-1.5-35B-A3B-ROCmFP4-GGUF（MoE）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | **1248.7 t/s** | **86.4 tok/s** |
| 代码 think-off | **1246.4 t/s** | **82.0 tok/s** |
| 散文 think-low | **1239.1 t/s** | **85.9 tok/s** |
| 代码 think-low | **1246.6 t/s** | **79.6 tok/s** |

> **fork 版备注**：`roc_rocmfp4` 的 Ornith-1.5-35B-A3B 是 **MoE 模型**（35B 总参 / 3B 激活），prefill ~1250 t/s、decode ~80-86 tok/s，远快于所有 dense 27B 模型（约 3-5 倍）。此数据为历史实测（含投机解码），与上方官方版"无投机 baseline"方法不同，两者不可直接数值对比。

## 引用与致谢

本包中的两个 NovaMax 引擎（`roc_official` / `vulkan_official`）均基于以下开源 GitHub 仓库编译而来，特此声明并致谢：

### 编译依赖的 llama.cpp / ROCmFPx 仓库

| 仓库 | 分支 | 用途 |
|---|---|---|
| [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | master | llama.cpp 上游官方仓库 |
| [ROCmFPX/ROCmFPX](https://github.com/ROCmFPX/ROCmFPX) | main | **官方 ROCmFPX 主线**（HEAD `2249677`，compiled for `roc_official` + `vulkan_official`）——Carlo Pasquale（charlie12345）创建，含 qwen4exp/qwen35 架构 + ROCmFPx 全量化（Q4_0_ROCMFP4/FAST、ROCMI4、Q2-Q8_ROCMFPX）+ HIP/Vulkan 双后端 |
| [charlie12345/ROCmFPX](https://github.com/charlie12345/ROCmFPX) | main | ROCmFPX 前身 fork（本次已弃用，被官方主线替代）；历史引擎来源 |
| [LaurentZuijdwijk/llama.cpp](https://github.com/LaurentZuijdwijk/llama.cpp) | `vulkan/qwen4exp-rocmfpx` | 历史 qwen4exp + Vulkan ROCmFPx fork（本次已弃用）；曾作为 vulkan_qwen4exp 引擎来源 |
| [ciru-ai/ROCmFPX](https://github.com/ciru-ai/ROCmFPX) | `ciru/upstream-rocmfpx-scale-eval` | ROCmFPx 量化内核（`rocmfp4_hip.cu`）来源调研与参考 |
| [daimonionnn/amd-rocmfpx-for-win](https://github.com/daimonionnn/amd-rocmfpx-for-win) | main | AMD Windows 平台 ROCmFPx 编译/基准参考（本机 Windows 编译参数借鉴于此） |

### 各引擎对应的构建来源

| 引擎目录 | 来源仓库/分支 | 本机编译参数 |
|---|---|---|
| `novamax-adapters\roc_official` | ROCmFPX/ROCmFPX @ main（HIP，gfx1151） | `-DGGML_HIP=ON -DGGML_VULKAN=OFF -DGGML_HIP_FORCE_MMQ=ON -DCMAKE_HIP_ARCHITECTURES=gfx1151` + rocm-7.14 clang |
| `novamax-adapters\vulkan_official` | ROCmFPX/ROCmFPX @ main（Vulkan，gfx1151） | `-DGGML_VULKAN=ON -DGGML_HIP=OFF -DGGML_CUDA=OFF` + MSVC 14.44 + VULKAN_SDK 1.4.357.0 |

### 致谢

- 感谢 **ggml-org** 维护 llama.cpp 及其社区，贡献了投机解码、prompt cache、Vulkan/ROCm 后端等核心能力。
- 感谢 **Carlo Pasquale (charlie12345)** 创建并维护 **ROCmFPX** 权重格式家族（ROCmFP3/4/6/8）及官方主线 `ROCmFPX/ROCmFPX`，使 AMD Radeon 8060S（Strix Halo）能跑通 Qwen3.8-27B-ROCmFP4-FAST 等专有量化模型。
- 感谢 **ciru-ai / LaurentZuijdwijk** 在 ROCmFPx 内核与 qwen4exp 架构上的早期贡献（已被官方主线吸收）。
- 感谢 **daimonionnn** 的 amd-rocmfpx-for-win，作为 AMD Windows 平台上的编译与性能评估参考。
- 适配目标平台 **NovaMax**（Linglong NovaStudio）与 **lemonade / LemonadeServer** 的引擎发现与模型加载机制为本次集成提供了基础。
- 本机工具链：Visual Studio 18 Build Tools (MSVC 14.44)、CMake 4.3.1、Ninja、Vulkan SDK `1.4.357.0`、ROCm 7.14 (HIP 工具链)、rocBLAS/hipBLAS，均用于上述引擎的本地编译。
