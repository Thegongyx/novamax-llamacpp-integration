---
name: novamax-llamacpp-integration
description: 把本地编译好的自定义 llama.cpp 版本（ROCm/HIP、Vulkan、w4a4 等）注册为 NovaMax 模型引擎，并正确配置模型启动参数。当用户要将编译产物（如 ROCmFPX、LaurentZuijdwijk fork）接入 NovaMax 使用，或排查模型启动/超时/参数兼容问题时使用。
---

# novamax-llamacpp-integration

把本地编译出的 llama.cpp 可执行文件（含依赖 DLL）注册为 NovaMax 的 llamacpp 引擎，并在 NovaMax 数据层配置模型启动参数。覆盖引擎目录、`.installed` 标记、模型数据库参数、启动命令生成、常见坑。

## 关键路径

| 用途 | 路径 |
|---|---|
| NovaMax 引擎目录 | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\<variant>\` |
| 引擎列表数据 | `C:\Linglong\NovaStudio\NovaMax\data\engines.json` |
| 模型库（SQLite） | `C:\Linglong\NovaStudio\NovaMax\data\novamax.db`（表 `models` / `settings`） |
| 模型运行时日志 | `C:\Linglong\NovaStudio\NovaMax\data\logs\model_<id>_runtime.log` |
| OpenAI API 端口 | `http://localhost:15057/v1`（model-router-plugin） |
| 单引擎后端端口 | `http://localhost:<model.port>/v1`（如 1234，直接 llama-server） |

## 核心机制

NovaMax 启动模型时，从 `novamax.db` 的 `models` 表读取该模型记录的 `user_parameters` 字段，生成完整 llama-server 命令行（见运行时日志第 5 行）。因此**引擎注册**和**模型参数**是两件事，要分开做。

## 引擎发现机制（本地引擎如何被扫描）

已从源码确认（`backend/dist/index.js` 的 `getInstalledVersions`）。NovaMax 发现一个本地引擎的**真实链路**：

```
遍历 llamacpp 的 variants (rocm / vulkan / cuda / other)
  → 进入 external\llamacpp\<variant>\          # 例 external\llamacpp\rocm
  → 枚举该目录下所有子目录                      # 例 roc_official, vulkan_qwen4exp
  → 对每个子目录调 _isValidEngineDir 判断是否有效
  → 有效则读该目录的 .installed JSON，取 version 字段
  → 加入已安装引擎列表，可在模型参数中作为 engine_version 选用
```

> 当前本机引擎：`external\llamacpp\rocm\{roc_official}`、`external\llamacpp\vulkan\{vulkan_official, vulkan_qwen4exp}`。社区 fork 引擎 `rocm_w4a4` / `roc_rocmfp4` **已弃用并从 NovaMax 移除**（官方 `roc_official` 已替代，见 README）。

**结论**：本地自定义引擎**只要两件事就够**——

1. 目录放在正确的 variant 下（`rocm_*` → `rocm\`，`vulkan_*` → `vulkan\`，名字前缀须与 backend 对应）
2. 目录内有**有效的 `.installed` 文件**（`engine: llamacpp`、`variant`、`version` 齐全）

**不需要**注册进 `engines.json`（`engines.json` 只定义配方/官方发布版，`getInstalledVersions` 扫描的是磁盘目录 + `.installed`，与它无关）。

- 引擎放好后，需 NovaMax **重新扫描/重载**后才可见（UI 刷新或重启后端）。
- `variant_mode: "single"`：一个引擎 = 一个 variant 目录下的一个子目录。

## A. 引擎扫描时会被识别的文件结构

NovaMax 扫描 `external\llamacpp\rocm\<variant>\` 目录，需要的文件（以已装好的 rocm_w4a4 为例，14 个文件）：

```
.installed         # 标记文件（JSON），缺失导致 NovaMax 显示"安装不完整"
amdhip64_7.dll
ggml-base.dll
ggml-cpu.dll
ggml-hip.dll
ggml.dll
llama-bench.exe
llama-cli.exe
llama-common.dll
llama-perplexity.exe
llama-quantize.exe
llama-server.exe   # 必须包含
llama.dll
mtmd.dll
```

> 不同编译产物依赖的 DLL 不同：HIP 版带 `ggml-hip.dll`/`amdhip64_7.dll`，Vulkan 版带 `ggml-vulkan.dll`（62MB）等。**把编译目录里的全部 exe/dll 一起拷入，缺一个 NovaMax 会报引擎不完整或启动失败。**

> **不同后端文件数与打包结构不同**：
> - **rocm / HIP 版（如 rocm_w4a4, roc_rocmfp4）**：`llama-server.exe` ~4.2MB（逻辑内嵌单文件），共 **14 文件**。
> - **vulkan 版（如 vulkan_qwen4exp）**：`llama-server.exe` 仅 **10KB（启动壳）**，核心逻辑在同目录 `llama-server-impl.dll`，共 **12 文件**。
>
> 两者都是新版 llama.cpp 的两种打包方式——vulkan 的 exe 只是转发器，**只要同目录有对应 `*-impl.dll` 即可独立运行**。验证可用：`llama-server.exe --list-devices` 能列出 `Vulkan0` / `ROCm0` 设备即依赖齐全。

### `.installed` 标记文件

放在引擎目录根，JSON 格式（字段与 NovaMax 预期一致）：

```json
{
  "runtime": null,
  "installed_at": "2026-09-02T00:05:40.020Z",
  "version": "rocm_w4a4",
  "archiveSha256": "local-bin-w4a4-custom-build",
  "variant": "rocm",
  "engine": "llamacpp"
}
```

`archiveSha256` 用于区分本地/官方来源；`version` 须与引擎目录名一致。`.installed` 缺失是 NovaMax 后台显示"安装不完整"的最常见原因。

## B. 模型参数配置（novamax.db）

模型记录存在 `novamax.db` 的 `models` 表，`data` 字段是一段 JSON，关键子字段：

```
id: "custom_Qwen3_8-27B-ROCmI4-MTP-GGUF"   # 模型标识
user_parameters: { ... }                    # 实际生效的启动参数
parameters: { ... }                         # 默认参数（模型内置预设，见下方端口坑）
user_parameters_version: "1.0.0"
engine_version: "roc_official"              # 使用的引擎 variant（roc_official / vulkan_official / vulkan_qwen4exp 等）
deleted_parameters: ["ub","b","spec-draft-adaptive"]  # NovaMax 禁止/忽略的参数
```

`user_parameters` 里的键会被拼成 llama-server 命令行参数。例如 `--n-gpu-layers 100 --flash-attn on --jinja --parallel 3 --reasoning on --spec-type draft-mtp ... --kv-unified`。

> ⚠️ **手动/自定义引擎模型的 `parameters`（默认预设）必须和默认引擎保持一致，否则会出问题**：
> - **端口锁死 1234 的坑**：端口取值是 `user_parameters.port || Ue["chat"]`（`Ue={chat:1234, embedding:1278, reranker:1245}`）。若把 `port: 1234` **写进 `parameters`**（模型级预设），`saveUserParameters` 会把"与模型默认值相同的改动"当无变化而**重置/忽略** → webui 改端口不生效、模型永远锁在 1234。
> - **修复**：`parameters` 里**不要放 `port`**（保持 None，同默认引擎），端口完全交给 `user_parameters.port` 决定，回退 `Ue["chat"]=1234`。
> - **对齐默认引擎的预设**：默认 llm 引擎 `parameters` 为 `{"context_length":0,"n-gpu-layers":100,"no-mmap":true,"parallel":1,"reasoning":"off","repeat_penalty":1.1,"temperature":0.7,"top_k":40,"top_p":0.9,"version":"1.0.0"}`。**不要写 `load-mode:"nommap"`**（默认引擎用 `no-mmap:true`）、不要写 `port`，temperature 用 0.7。

### 查询/修改工具

```powershell
# 查该模型的完整配置
python -c "import sqlite3;c=sqlite3.connect(r'C:\Linglong\NovaStudio\NovaMax\data\novamax.db');rows=list(c.execute(\"select id,data from models where id like '%名称%'\"));print([r[1][:2500] for r in rows])"
```

## C. 常见坑

### 1. 超时导致请求被取消（should_stop）
日志反复出现：
```
W srv next: stopping wait for next result due to should_stop condition (adjust the --timeout argument if needed)
W srv stop: cancel task, id_task = ...
```
**根因**：超长上下文（如 23929 token）prefill 需 86-108 秒，超出默认超时被取消。若模型 `user_parameters` 没显式设置 `timeout`，会走默认值偏小。需在模型参数里加 `--timeout <较大值>`（如 600+），或在 novaairouter 配置调大超时。

### 2. thinking_effort 传了但没生效
命令行里 `--chat-template-kwargs {"thinking_effort":"low"}`，但日志模板显示 `Reasoning effort is set to xhigh`。说明模板默认值覆盖了传参，或 kwargs 传递没被模板采用。需确认模型内置模板对 thinking_effort 的处理，或用模型侧模板覆盖。

### 3. 参数兼容性（不同 fork 支持参数不同）
- **bin-w4a4（charlie12345/ROCmFPX）**：`--kv-unified -kvu`、`--spec-draft-device ROCm0`、`--reasoning on/off`、`--reasoning-budget N` ✓；**不支持** `--spec-draft-adaptive`（报 invalid argument）、`--reasoning-effort`。
- **LaurentZuijdwijk fork**：支持 `--spec-draft-adaptive`、`--reasoning-effort`。
- **MTP 模型**：`--spec-draft-adaptive` 不可用，需手动 `--spec-draft-n-max`。`chat_template_kwargs` 会被 NovaMax 存为字符串。
- **`--spec-mtp-strict-qwen`**：若日志出现 warning 可考虑加，保证投机解码与无投机输出一致。
- 不确定参数是否被支持时，用 `llama-server.exe --help` 或跑一次看日志。

### 4. MTP 投机解码接受率低
日志 `#acc rate/pos = (0.850,0.600,0.400,0.100,0.100)`：第一个 draft 接受率高，之后衰减快，decode 只有 5-7 tok/s。可降低 `--spec-draft-n-max`（6→3）减少无谓生成。

### 5. prompt cache 复用失败（spec-boundary-mismatch）
长上下文 + MTP 时，KV cache 因投机解码边界不匹配反复失效，每次全量重新 prefill（慢）。这是 MTP 特性，非错误。

### 6. 模型文件必须放对位置（MTP 挂载）
MTP 草稿模型要能被 lemonade/NovaMax 识别，主模型（含 mmproj/draft）须放 HF 缓存布局 `C:\Users\xu352\.cache\huggingface\hub\models--<org>--<repo>\snapshots\<commit>\`，用 `user.*` 注册跨仓库 draft。放进 `extra_models_dir` 不会被识别为 draft。

### 7. 手动/自定义引擎端口锁死 1234（改端口无效 → 模型不可调用）
**症状**：默认引擎的模型可在 webui 改端口、跑不同端口；但手动加入的引擎（自定义 `engine_version`）模型**永远锁在 1234**，一旦改端口模型就调用不到。

**根因**（`backend/dist/index.js` + `novamax.db` 实测）：
- 端口来自 `有效参数.port || Ue["chat"]`（Ue={chat:1234,…}），默认 1234。
- `getEffectiveParameters` 合并 `{默认参数(Fe,port:1234), 模型 parameters, 用户 user_parameters}`。
- **若把 `port:1234` 写进模型记录 `parameters`**（手动引擎模型常把整套默认参数写死），`saveUserParameters` 对"与模型默认值相同的改动"作**无变化处理**（重置/忽略）→ webui 改端口不保存，端口永远 1234。

**修复**：把该模型 `novamax.db` 里 `data.parameters.port` 删掉（置 None），并让 `parameters` 与默认引擎预设一致（见上方 B 节）：`no-mmap:true`（不要 `load-mode:"nommap"`）、temperature `0.7`、不写 `port`。改后需**重启 NovaMax 后端**（或重新扫描）生效。

> 备份 db 再改：`C:\Linglong\NovaStudio\NovaMax\data\novamax.db`。

### 8. Flash-Next（qwen4exp）引擎兼容性
- **官方引擎**（`roc_official`/`vulkan_official`，ROCmFPX/ROCmFPX）：能加载 Flash-Next **v2 单张量**文件，但**只能无投机**（MTP 外部草稿加载失败 `blk.0.hc_attn_norm.weight not found`；ngram-cache 负优化）。
- **fork 引擎**（`vulkan_qwen4exp`，LaurentZuijdwijk）：支持 per-head 布局 + **MTP 投机**；REAP-288 剪枝版也用它。

## D. 验证

1. **引擎加载**：`llama-server.exe --list-devices` 能列出 GPU（如 `ROCm0: AMD Radeon 8060S`）。
2. **HTTP 启动**：`GET http://localhost:<port>/v1/models` 返回 JSON 模型列表。
3. **聊天**：`POST http://localhost:<port>/v1/chat/completions` 返回 `choices[].message`（带 `reasoning_content` 表示思考开启）。
4. **OpenAI 兼容**：`http://localhost:15057/v1/models` 返回 `{"data":[{"id":...}]}`，`/v1/chat/completions` 可用。
5. **NovaMax 后台**：引擎不再显示"安装不完整"。

## 端口参考（novaairouter 全家桶）

> ⚠️ **当前可用的 OpenAI 网关**：`http://localhost:15050/v1`（novaairouter，OpenAI 兼容，`/v1/models` 返回 JSON）——实测可用，**用这个**。**默认列举的 3001 端口不可用**（原因未知），勿用。

| 端口 | 进程 | 用途 |
|---|---|---|
| 1234 | llama-server | 单模型原生 OpenAI API |
| 15048 | node (nova launcher) | Web UI |
| 15049/15051 | novaairouter | router（/v1/models 404） |
| **15050** | novaairouter | **OpenAI 兼容 API（JSON models）——当前可用网关** |
| 15057 | model-router-plugin | 标准 OpenAI API（/v1/models + /v1/chat/completions）* |
| 19825 | node | NovaMax 后端 API（需 X-OpenCLI header） |
| 3001 | node | NovaMax Web UI（HTML，非 API）——⚠️ **不可用** |
