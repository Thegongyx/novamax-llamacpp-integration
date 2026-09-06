# 在 Windows 下编译 ciru-rocmfpx（llama.cpp Kairic Edge / ROCmFPX）引擎

本文记录在 **Windows + AMD HIP** 环境下编译 `ciru-rocmfpx`（llama.cpp 的 ROCmFPX / Kairic Edge
分支，用于跑 `Qwen3.8-27B-IU4-Kairic-Edge` 这类需要原生 IU4 / PromptForge 加速的模型）的步骤。

> 方法已在本机实测通过：`ggml-hip.dll` 正常，`llama-bench --list-devices` 能识别 AMD GPU，
> `llama-server --version` / `--kairic-edge` 可用。

文中所有路径均用**占位符**表示，请按实际环境替换（不写死绝对路径）：

| 占位符 | 含义 | 示例（仅示意） |
|---|---|---|
| `<repo_root>` | ciru-rocmfpx 源码根目录 | `/path/to/ciru-rocmfpx` |
| `<rocm_root>` | ROCm/HIP SDK 根目录（HIP 运行时 + clang） | `/opt/rocm` 或 `<sdk_dir>\rocm` |
| `<vs_root>` | Visual Studio / Build Tools 安装根目录 | `<vs_install>\BuildTools` |
| `<msvc_toolset>` | 锁定的 MSVC 工具集版本 | `14.44` |
| `<ck_root>` | Composable Kernel 检出目录（= `<repo_root>/third_party/composable_kernel`） | `<repo_root>\third_party\composable_kernel` |
| `<build_dir>` | 构建目录（= `<repo_root>build-win-kairic`） | `<repo_root>\build-win-kairic` |
| `<gpu_arch>` | GPU 架构（Strix Halo = `gfx1151`） | `gfx1151` |
| `<model_dir>` | 模型文件（`.gguf` + 3 个 `.pfs` 侧车）所在目录 | `<models>\...` |
| `<model_repo>` | 模型 HF 仓库（如 `jcbtc/Qwen3.8-27B-IU4-Kairic-Edge`） | `jcbtc/...` |

---

## 1. 目标与范围

- **仓库**：ciru-rocmfpx（llama.cpp + ROCmFPX + PromptForge + Kairic Edge 分支）。
  Kairic Edge 版把选定的 packed-U4 运算接到 AMD RDNA 3.5 原生 `IU4` matrix 指令上，
  并携带 Prompt Forge / Dual View `.pfs` 侧车运行时。
- **目标**：编译 **ROCm (HIP) 引擎** —— `ggml-hip` 后端 + `llama-server` 工具链。
- **只编 HIP**：`GGML_VULKAN=OFF`、`GGML_CUDA=OFF`（如需 Vulkan 双后端，把 `GGML_VULKAN` 改 `ON`）。
- 产物集中在 `<build_dir>\bin\`，核心是 **`ggml-hip.dll`** 和 **`llama-server.exe`**。

---

## 2. 前置条件（本机实测路径）

| 组件 | 说明 | 关键点 |
|---|---|---|
| **MSVC (VS Build Tools)** | 提供 MSVC 头文件/库 | **必须锁定工具集，见 §6**（14.51 会挂） |
| **ROCm / HIP SDK** | 提供 HIP 运行时 + ROCm clang | `hipcc`/`hipconfig` 在 `<rocm_root>\bin`，`clang`/`clang++` 在其下（见 §2 布局说明） |
| **cmake** | 配置 | VS Build Tools 内置或独立安装 |
| **ninja** | 构建 | VS Build Tools 内置或独立安装 |
| **git** | 拉取 Composable Kernel | — |
| **Composable Kernel** | PromptForge 依赖 | 需 pin 到指定 commit 并打 IU4 补丁（§4） |

### ROCm clang 位置（重要）

ROCm/HIP clang 的安装布局**因 SDK 而异**，编译前必须确认它在 PATH 里能找到：

- 场景 A：`<rocm_root>\bin\clang++.exe`（AMD HIP SDK 常见布局，`bin\` 下直接有 clang）
- 场景 B：`<rocm_root>\lib\llvm\bin\clang++.exe`（另行解压的 ROCm 布局，`bin\` 下只有 hipcc/hipconfig）

**两种都要覆盖**：launcher 里把 `<rocm_root>\bin` **和** `<rocm_root>\lib\llvm\bin` 都加到 PATH，
让 `clang`/`clang++` 能被 CMake 找到（用**名称** `clang`/`clang++`，不要用完整路径指向编译器）。

> ⚠️ 若本机同时存在另一套被缓存/复制的 HIP 组件（例如某些 AI 工具或其它项目自带/缓存的 ROCm 树），
> **别让它优先级排在前面、也别用它当编译器**——其行为与正式 ROCm 工具链不一致，编译必坑。
> 只把正式 `<rocm_root>` 的 `bin` 和 `lib\llvm\bin` 放进 PATH。

---

## 3. 关键构建脚本（一次性 configure + build）

HIP 编译是 **MSVC-ABI / ROCm clang**：必须先在 **vcvars（工具集 14.44）** 环境里执行，
并把 `<rocm_root>\bin` + `<rocm_root>\lib\llvm\bin` 加到 PATH。

把下面内容存成 `<repo_root>\build-win-kairic.bat`，一次性完成 configure + build：

```bat
@echo off
rem ---------------------------------------------------------------
rem 1) 激活 MSVC x64 环境，锁定工具集（关键！14.51 会挂，见 §6）
call "<vs_root>\VC\Auxiliary\Build\vcvars64.bat" -vcvars_ver=<msvc_toolset> || exit /b 1

rem 2) 把 cmake/ninja 与 ROCm/HIP 的 bin + clang 目录加到 PATH
set "PATH=<vs_root>\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;<vs_root>\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;<rocm_root>\bin;<rocm_root>\lib\llvm\bin;%PATH%"

set "SRC=<repo_root>"
set "BLD=%SRC%\build-win-kairic"
set "CK=%SRC%\third_party\composable_kernel"
set "ROCM=<rocm_root>"
rem ROCM_PATH 环境变量必需：ggml-hip/CMakeLists.txt 读取它（Windows 下不设会落到 /opt/rocm -> /usr 的错路径）
set "ROCM_PATH=<rocm_root>"

rem ---------------------------------------------------------------
rem 3) 配置：仅 HIP，目标 <gpu_arch>，kairic + PromptForge 选项
rem 注意：Windows 上 CMake 走 CXX-as-HIP 路径，CMAKE_HIP_FLAGS 会被忽略，
rem       所以两个调优宏必须放进 CMAKE_CXX_FLAGS（见 §7 说明）
cmake -S "%SRC%" -B "%BLD%" -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_C_COMPILER=clang ^
  -DCMAKE_CXX_COMPILER=clang++ ^
  -DCMAKE_CXX_FLAGS="-DGGML_ROCMFPX_RDNA35_MMID_MAX_BATCH=5 -DGGML_ROCMFPX_MOE_MMVQ_ROWS_PER_BLOCK=4" ^
  -DCMAKE_PREFIX_PATH="%ROCM%" ^
  -DBUILD_SHARED_LIBS=ON ^
  -DGGML_CPU=ON -DGGML_OPENMP=ON -DGGML_HIP=ON -DGGML_CUDA=OFF ^
  -DGGML_VULKAN=OFF -DGGML_HIP_FORCE_MMQ=ON -DGGML_HIP_GRAPHS=ON ^
  -DGGML_HIP_MMQ_MFMA=ON -DGGML_HIP_NO_VMM=ON ^
  -DGGML_HIP_ROCWMMA_FATTN=OFF -DGGML_NATIVE=ON ^
  -DAMDGPU_TARGETS=<gpu_arch> -DGPU_BUILD_TARGETS=<gpu_arch> ^
  -DCMAKE_HIP_ARCHITECTURES=<gpu_arch> -DGPU_TARGETS=<gpu_arch> ^
  -DPROMPTFORGE_CK_ROOT="%CK%" ^
  -DLLAMA_BUILD_WEBUI=OFF ^
  -DGGML_BUILD_TESTS=OFF -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_SERVER=ON || exit /b 1

rem ---------------------------------------------------------------
rem 4) 编译目标：先 ggml-hip（探查 HIP 编译风险），再 llama-server/llama-bench
set "JOBS=32"
cmake --build "%BLD%" --target ggml-hip -j %JOBS% || exit /b 1
cmake --build "%BLD%" --target llama-server llama-bench -j %JOBS% || exit /b 1

echo Build OK: %BLD%\bin\llama-server.exe
exit /b 0
```

运行：`& cmd.exe /c "<repo_root>\build-win-kairic.bat"`（PowerShell）或直接在 cmd 里执行。

---

## 4. 准备 Composable Kernel（PromptForge 必需）

`ggml-hip/CMakeLists.txt` **强制要求** `-DPROMPTFORGE_CK_ROOT`（未设置会 `FATAL_ERROR`），
且只编译其中的**一个** CK 实例源文件（不是整个 CK 库）。流程：

```bash
# 在 <repo_root> 下
git clone https://github.com/ROCm/composable_kernel.git third_party/composable_kernel
git -C third_party/composable_kernel checkout <CK_PINNED_COMMIT>
git -C third_party/composable_kernel apply <repo_root>/patches/composable-kernel-gfx1151-iu4.patch
```

- `<CK_PINNED_COMMIT>`：见 `<repo_root>/docs/kairic-edge-gfx1151.md` 或 `scripts/build-kairic-edge-gfx1151.sh`
  （当前为 `fdf4bb7fcc984811cef48ce817d89aac064b984a`）。
- 补丁路径：`patches/composable-kernel-gfx1151-iu4.patch`；文档里给了 SHA-256 校验值，
  **请自行校验**——若本地补丁哈希与文档不一致，说明分支/补丁可能被改动过，
  但只要 CK 检出已在 pin commit 上打好补丁（`git -C ... diff --check` 为空、缺少数文件已修改），即可继续。

---

## 5. 关键 CMake 开关说明

| 开关 | 值 | 作用 |
|---|---|---|
| `GGML_HIP` | `ON` | 开启 HIP 后端（ROCm 引擎） |
| `GGML_HIP_FORCE_MMQ` | `ON` | ROCmFPX/ROCMFP4 内核要求的 MMQ 强制开关，**必须开** |
| `GGML_HIP_MMQ_MFMA` | `ON` | 指定 MMQ 走 MFMA 内核路径 |
| `GGML_HIP_GRAPHS` | `ON` | 启用 HIP graph（该分支所需） |
| `GGML_HIP_NO_VMM` | `ON` | 关闭 VMM（该分支所需） |
| `GGML_HIP_ROCWMMA_FATTN` | `OFF` | RDNA 3.5 平台关闭 ROCWMMA flash-attn（防不兼容内核） |
| `GGML_CUDA` / `GGML_VULKAN` | `OFF` | 只编 HIP，避免误开其它后端 |
| `AMDGPU_TARGETS` / `GPU_BUILD_TARGETS` / `CMAKE_HIP_ARCHITECTURES` / `GPU_TARGETS` | `<gpu_arch>` | 指向目标 GPU |
| `PROMPTFORGE_CK_ROOT` | `<ck_root>` | PromptForge 的 CK 实例源（**必填**） |
| `BUILD_SHARED_LIBS` | `ON` | 共享库（`ggml-hip` 不支持静态，`GGML_STATIC` 会 FATAL） |
| `LLAMA_BUILD_SERVER` / `GGML_BUILD_TESTS` / `LLAMA_BUILD_TESTS` | `ON`/`OFF`/`OFF` | 只编 server，减编译量 |

### 两个调优宏（必 须 生效）

`GGML_ROCMFPX_RDNA35_MMID_MAX_BATCH` 与 `GGML_ROCMFPX_MOE_MMVQ_ROWS_PER_BLOCK` 控制核模板实例化
（有默认值，不传也能编译，但按认证值 `5` / `4` 更符合 Kairic Edge 达标配置）。

**Windows 关键差异**：因为 CMake 在 Windows 上走 `CXX_IS_HIPCC` 路径（`enable_language(HIP)`
被跳过），`.cu` 按 CXX 编译，导致 **`CMAKE_HIP_FLAGS` 会被 CMake 忽略**（configure 会打印
"variables were not used ... CMAKE_HIP_FLAGS" 警告）。所以**这两个宏必须放进 `CMAKE_CXX_FLAGS`**
（如 launcher 所示），否则不生效。

---

## 6. Windows 专属移植（Linux 分支缺的适配）

Kairic Edge 是在 **Linux/GCC** 上认证的，AI 生成的 `PromptForge` 相关代码用了大量 **POSIX/Linux
专属 API**，在 Windows 上需要做最小适配。以下三处是**实测必须**的（均加 `_WIN32` 保护，不影响 Linux）：

### 6.1 PromptForge `.pfs` 侧车加载的 POSIX I/O

`ggml/src/ggml-cuda/promptforge.cu`、`promptforge_output_k8.cu` 用 `sys/mman.h` / `pread` /
`ssize_t` / `madvise` / `posix_fadvise` 等（Linux-only，Windows 无对应头文件）。

**做法**：新增 `<repo_root>/ggml/src/ggml-cuda/promptforge_win_compat.h`，在 `_WIN32` 下提供 shim：

- `open→_open`、`close→_close`、`fstat→_fstat64`、`read→_read`、`stat→_stat64`（MSVC CRT）
- `mmap` = 定位读取整段到 `malloc` 缓冲、`munmap` = `free`（避免把 `<windows.h>` 引入 `-x hip`
  设备翻译单元——宏会污染设备代码）
- `pread`（按 offset 读，不改文件读指针）
- `madvise`/`posix_fadvise` 为**空实现**
- 定义 `ssize_t` / `PROT_READ` / `MAP_PRIVATE` / `MAP_FAILED` / `MADV_DONTNEED` / `O_CLOEXEC` 等

然后在两个 `.cu` 的包含区改为：

```c
#if defined(_WIN32)
#include "promptforge_win_compat.h"
#else
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#endif
```

### 6.2 `setenv`（Linux-only）

`common/common.cpp`、`common/arg.cpp` 用了 `setenv`（Kairic ngram/M65 环境变量 + `GPU_MAX_HW_QUEUES`）。
Windows 用 `_putenv_s`。在文件顶部加：

```c
#ifdef _WIN32
#  ifndef setenv
#    define setenv(_name_, _value_, _overwrite_) _putenv_s((_name_), (_value_))
#  endif
#endif
```

### 6.3 MTP 符号跨 DLL 导出

`common/speculative.cpp`（llama-common）会调用 `src/llama-graph.cpp` 里的
`llm_graph_set_mtp_speculative_step` / `llm_graph_get_mtp_speculative_step`。
**共享库 + Windows 必须显式 `__declspec(dllexport)`**（Linux 默认可见性自动导出，所以上游在 Linux
测过能过、Windows 链接却报 `undefined symbol`）。在 `src/llama-graph.cpp` 的定义处：

```c
#ifdef _WIN32
#define LLM_GRAPH_MTP_EXPORT __declspec(dllexport)
#else
#define LLM_GRAPH_MTP_EXPORT
#endif

LLM_GRAPH_MTP_EXPORT void llm_graph_set_mtp_speculative_step(int32_t step) { ... }
LLM_GRAPH_MTP_EXPORT int32_t llm_graph_get_mtp_speculative_step() { ... }
```

---

## 7. 验证

构建完成后，把 ROCm/HIP 运行时 DLL 目录（`<rocm_root>\bin`）加到 PATH，再探测：

```powershell
$env:PATH = "<rocm_root>\bin;$env:PATH"
& "<build_dir>\bin\llama-server.exe" --version
& "<build_dir>\bin\llama-server.exe" --help | Select-String 'kairic-edge'
& "<build_dir>\bin\llama-bench.exe" --list-devices
```

预期：

```
version: 81 (205a3e5f)
built with Clang 23.0.0 for Windows AMD64
--kairic-edge    enable the qualified KAIRIC EDGE serving path
ggml_rocm_init: found 1 ROCm devices (Total VRAM: ...):
  Device 0: AMD Radeon ..., <gpu_arch> ...
Available devices:
  ROCm0: AMD Radeon ... (...)
```

出现 `ROCm0` 即表示 ROCm 引擎编译并加载成功。

---

## 8. 运行某个 Kairic Edge 模型

模型通常需要 4 个文件（主 `.gguf` + 3 个 `.pfs` 侧车，从模型卡下载）：

```
<model>.gguf
...-FFN.pfs
...-GDN.pfs
...-GDN-Output.pfs
```

运行前设置一系列 `PROMPTFORGE_*` 环境变量（参考 `<repo_root>/scripts/run-kairic-edge-gfx1151.sh`
移植到 Windows），再启动：

```powershell
$env:PATH = "<rocm_root>\bin;$env:PATH"
$env:ROCM_PATH = "<rocm_root>"
$env:HIP_VISIBLE_DEVICES = "0"
$env:HSA_OVERRIDE_GFX_VERSION = "11.5.1"
$env:GGML_CUDA_GRAPH_OPT = "0"
$env:LLAMA_TARGET_GREEDY_ARGMAX_FASTPATH = "1"
$env:LLAMA_MTP_CPU_ARGMAX_FASTPATH = "1"
$env:PROMPTFORGE_MODE = "iu4_ffn"
$env:PROMPTFORGE_IU4_HADAMARD = "1"
$env:PROMPTFORGE_ENABLE_GDN = "1"
$env:PROMPTFORGE_ENABLE_GDN_OUTPUT = "1"
$env:PROMPTFORGE_GDN_OUTPUT_KEEPERS = "v3_lateq6"
$env:PROMPTFORGE_ENABLE_SMALLM_IU4 = "1"
$env:PROMPTFORGE_ENABLE_SMALLM_GDN_IU4 = "1"
# 三个 sidecar 路径（换成模型实际存放目录）
$env:PROMPTFORGE_IU4_SIDECAR   = "<model_dir>\...-FFN.pfs"
$env:PROMPTFORGE_SIDECAR       = "<model_dir>\...-FFN.pfs"
$env:PROMPTFORGE_GDN_SIDECAR_OVERRIDE = "<model_dir>\...-GDN.pfs"
$env:PROMPTFORGE_GDN_SIDECAR   = "<model_dir>\...-GDN.pfs"
$env:PROMPTFORGE_GDN_OUTPUT_SIDECAR   = "<model_dir>\...-GDN-Output.pfs"

& "<build_dir>\bin\llama-server.exe" -m "<model_dir>\<model>.gguf" --alias main \
  --host 127.0.0.1 --port 8080 --jinja -dev ROCm0 -ngl 999 \
  -c 262144 -b 2048 -ub 512 -fa on -ctk f16 -ctv f16 \
  -t 16 -tb 32 -np 1 -ctxcp 32 --cache-ram 8192 \
  --cache-prompt --cache-idle-slots --metrics \
  --kairic-edge \
  --spec-type draft-mtp --spec-draft-device ROCm0 --spec-draft-ngl 999 \
  --spec-draft-type-k f16 --spec-draft-type-v f16 \
  --spec-draft-n-max 4 --spec-draft-n-min 0 \
  --spec-draft-p-min 0.0 --spec-draft-p-split 0.10 --spec-draft-backend-sampling \
  --temp 0 --top-p 1.0 --top-k 0 --min-p 0.0 \
  --reasoning off --reasoning-format none --reasoning-budget -1
```

启动后日志应出现：`Kairic Edge: enabled (ngram=24/64/64, M65 verifier=strict-compact)`。

---

## 9. 坑与注意事项（本机实测踩坑）

1. **MSVC 工具集必须锁 `14.44`**。默认 `vcvars64.bat` 会拿最新工具集（如 `14.51`），其 `<cmath>`
   新增 `_CLANG_BUILTIN` 块，导致 ROCm clang 把 `isgreater/isless/...` 重声明为 `__device__`，
   每个 `.cu` 都报
   `"__device__ function 'isgreater' cannot overload __host__ __device__ function 'isgreater'"`。
   → 必须 `-vcvars_ver=14.44`（安装对应工具集）。

2. **别用缓存/复制的另一套 HIP 组件当编译器**（例如某些 AI 工具或其它项目自带/缓存的 ROCm 树）。只用正式
   `<rocm_root>`，且 `PATH` 别让其它 HIP 组件排前面。

3. **clang 找不到**：ROCm clang 不一定在 `<rocm_root>\bin`。把 `<rocm_root>\bin` **和**
   `<rocm_root>\lib\llvm\bin` 都加到 PATH（launcher 已含）。不然 CMake 报
   `Tell CMake where to find the compiler ... clang`。

4. **`ROCM_PATH` 环境变量必须设**：`ggml-hip/CMakeLists.txt` 读取 `$ENV{ROCM_PATH}`，Windows 下没设
   会 `NOT EXISTS /opt/rocm` → fallback 到 `/usr`（错误）。launcher 里用 `set "ROCM_PATH=<rocm_root>"`。

5. **`CMAKE_HIP_FLAGS` 在 Windows 上不生效**（configure 打印 unused 警告）。两个调优宏放
   `CMAKE_CXX_FLAGS`，否则不生效。

6. **别去改 ROCm 工具链头文件 / MSVC 的 `<cmath>`**。根因是 MSVC 版本，改头文件是饮鸩止渴。
   回到 14.44 干净解决。

7. **别反复删 build 目录 + 散落命令**。一次性写好 launcher bat（§3），固定 `-vcvars_ver=14.44`、
   固定 PATH、固定 configure，保持环境一致是 HIP 构建成功的前提。

8. **这是"可移植重建"**：本地编译产物**不会**逐字节匹配官方冻结的认证二进制（文档自己也声明
   本地重建记录发布 commit + 本地编译器身份）。功能等效，但哈希/性能可能略有差异。
   认证版本用 TheRock 7.15 + GCC 13.3 + CMake 4.4；本地可用其它兼容 ROCm 版本替代。

9. **`BUILD_SHARED_LIBS=ON` 是必须的**：`ggml-hip` 不支持静态（`GGML_STATIC` 会 `FATAL_ERROR`）。

---

## 10. 一句话总结

Windows 下编译成功的充要条件 = **MSVC 14.44** + **可靠 ROCm/HIP 工具链（PATH 含 `<rocm_root>\bin` 与
`<rocm_root>\lib\llvm\bin`，不碰其它 HIP 组件）** + **名称编译器 `clang`/`clang++`** +
**设置 `ROCM_PATH` + `PROMPTFORGE_CK_ROOT` + 调优宏放 `CMAKE_CXX_FLAGS`** +
**修好 §6 的三处 Windows 移植（POSIX I/O shim / setenv / MTP dllexport）** +
**一次全量 `cmake --build`**。

其余任何"改头文件 / 拆开单编 / 用 14.51 / 用其它 HIP 组件 / 改 MSVC STL"的尝试，都会踩坑失败。
