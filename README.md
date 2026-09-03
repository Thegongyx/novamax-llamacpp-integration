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
| `vulkan_official` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF | ✅ |
| `roc_official` | HIP | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_official` | HIP | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF | ✅ |
| `roc_official` | HIP | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF | ✅ |
| `rocm_w4a4` | HIP/W4A4 | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Qwen3.8-27B-ROCmFPX-GGUF | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-GGUF | ✅ |
| `roc_rocmfp4` | HIP/ROCmFP4 | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |

> **兼容性说明**：
> - 官方版 `ROCmFPX/ROCmFPX` **不支持 `--spec-draft-adaptive`**（报 `invalid argument`），DFlash2/自适应投机需用 fixed `--spec-draft-n-max`；不支持 DFlash2 外部草稿（`--spec-type draft-dflash`）。
> - `roc_official` 与 `vulkan_official` 是同一官方源码的 HIP/Vulkan 两套构建，量化支持范围一致（Q4_0_ROCMFP4/FAST、ROCMI4、Q2-Q8_ROCMFPX 全系列）。
> - fork 版 `vulkan_qwen4exp` 支持 DFlash2 自适应投机；`rocm_w4a4`/`roc_rocmfp4` 分别侧重 W4A4 / ROCmFP4。
>
> 镜像域名用 hf-mirror.com（国内直连）。lemonade 拉取时需 `HF_ENDPOINT=https://hf-mirror.com` + 显式 `--source huggingface`。

## 满电源频率速度测试（官方版两引擎）

**测试条件**（与之前一致）：
- **电源/频率**：AMD 电源档位拉满（**整机功耗约 132W**）
- **提示词长度**：~1 万 token 长上下文（散文 10123 / 代码 9950 token，固定填充文本构造）
- **输出长度**：`max_tokens=3000`，实际连续输出约 3000 token；decode 仅在累计 ≥1000 token 后取 [1000, 末尾] 稳定段（约 2000 token）统计
- **场景**：散文/代码 × 思考模式 off/low
- **加载方式**：每次场景独立冷启动模型（spawn → 测 → kill），杜绝 KVCache 复用污染
- **投机**：无投机解码 baseline（`-np 2`，固定参数），保证引擎基础速度对比公平

> **方法说明**：本版用**直接 spawn 引擎**测速（官方版 Vulkan 与 lemonade 有 `--spec-draft-adaptive` 不兼容问题，故绕过 lemonade 直接驱动引擎），两个引擎方法完全一致，可公平对比。

### HIP 引擎（`roc_official`）—— Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 372.8 t/s | 13.4 tok/s |
| 代码 think-off | 382.2 t/s | 13.4 tok/s |
| 散文 think-low | 375.5 t/s | 13.4 tok/s |
| 代码 think-low | 377.1 t/s | 13.4 tok/s |

### Vulkan 引擎（`vulkan_official`）—— Qwen3.8-27B-ROCmFP4-FAST（主模型）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 114.4 t/s | 13.6 tok/s |
| 代码 think-off | 114.8 t/s | 13.5 tok/s |
| 散文 think-low | 109.6 t/s | 13.5 tok/s |
| 代码 think-low | 109.8 t/s | 13.5 tok/s |

> **测试方法**：直接 spawn 引擎（冷启动，无投机）→ 发 ~1 万 token 长上下文流式请求 → 首个 token 前计 PREFILL（prompt_tokens/首token时间）→ 累计输出 ≥1000 token 后取 [1000, 末尾] 稳定段计 DECODE → kill。每场景重新加载，保证无 KVCache 干扰。
> **备注**：
> - **HIP prefill 显著优于 Vulkan**（373-382 vs 110-115 t/s，约 3.4 倍）；decode 两者接近（13.4-13.6 tok/s）——本版测的是**无投机 baseline**（投机解码开启时 decode 更高，见历史 fork 数据）。
> - 官方两引擎是同一源码的 HIP/Vulkan 构建，性能差异主要来自后端；HIP 在 prefill 上有明显优势。

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
