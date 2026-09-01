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

## 常见坑速查

| 现象 | 根因 | 解决 |
|---|---|---|
| NovaMax 显示"安装不完整" | 缺 `.installed` 或引擎文件不全 | 补 `.installed` / 拷全 exe+dll |
| 请求被取消（`should_stop`） | 长上下文 prefill 超时 | 加 `--timeout <较大值>` |
| `thinking_effort` 传了没生效 | 模板默认覆盖了传参 | 改用模型模板覆盖 |
| 参数报 invalid argument | fork 不支持该参数（如 bin-w4a4 不支持 `--spec-draft-adaptive`） | 查 `--help` / 换支持参数的 fork |

详细流程、参数说明、6 个完整坑 → 见 `docs\novamax-llamacpp-skill\SKILL.md`。
