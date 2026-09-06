# 通用指南：在 Windows 下从源码编译 ROCmFPX 的 HIP 引擎

> 本文是 `BUILD-ROCM-ENGINE-WINDOWS.md` 的**通用化**版本：把原文档里的本机绝对路径全部抽象成
> `<占位符>` / 环境变量，并给出"如何在你自己机器上定位这些路径"的步骤。**任何一台 Windows + AMD GPU
> （Strix Halo / RDNA 等）的机器都可按本文操作**，不限于原作者的机器。
>
> 目标：编译 `ROCmFPX/ROCmFPX`（合并后的官方 fork，或你 fork 的等价仓库）里的 **ROCm (HIP) 后端**，
> 产出 `ggml-hip.dll`，并用 `llama-bench --list-devices` 认出 GPU。
>
> 参考：`amd-rocmfpx-for-win\llm-inference\Setup-ROCmFPX.ps1`（Windows 构建先例）

---

## 0. 先读懂约定：本文所有 «...» 都是你要自己填的变量

本文不像原型那样写死路径，而是用 `«变量名»`。开始前，先在头脑里（或一张纸上）为下面每个
`«变量名»` 填上**你机器上的真实值**，然后照抄命令时替换即可。

| 变量 | 含义 | 如何在本机定位 |
|---|---|---|
| `«REPO_DIR»` | ROCmFPX 源码仓库的本地路径 | 你 clone 到的目录，例如 `C:\...\ROCmFPX` |
| `«HIP_SDK»` | AMD HIP / ROCm 工具链根目录 | 见 §1 第 1 步，`hipcc.exe` 所在目录的上级 |
| `«HIP_INC»` | HIP SDK 的 include 目录 | 通常 `«HIP_SDK»\include` |
| `«CLANG_BIN»` | ROCm clang 所在目录 | `«HIP_SDK»\lib\llvm\bin`（或 `«HIP_SDK»\bin`） |
| `«VS_BUILDTOOLS»` | Visual Studio Build Tools 安装根 | 见 §1 第 2 步，`vcvars64.bat` 所在 |
| `«MSVC_VER»` | 要锁定的 MSVC 工具集版本 | **必须用 `14.44` 系**，见 §4 坑 1 |
| `«CMAKE_BIN»` | cmake.exe 所在目录 | 见 §1 第 3 步 |
| `«NINJA_BIN»` | ninja.exe 所在目录 | 见 §1 第 3 步 |
| `«GPU_ARCH»` | 你的 AMD GPU 架构 | gfx1151 / gfx1201 / gfx1100 等，见 §1 第 5 步 |
| `«BUILD_DIR»` | 构建目录（可任意） | 建议 `«REPO_DIR»\build-win-hip` |

> 通用原则：**凡是"Windows 带路径的编译器/工具"，都用名称（如 `clang`）而不是完整路径**，
> 这样 CMake 才能通过 PATH 正确解析到你想要的那套工具链。见 §4 坑 2。

---

## 1. 环境与前置条件（每台机器都要先装/先确认）

### 1.1 必备组件

| 组件 | 用途 |
|---|---|
| **AMD HIP / ROCm SDK（Windows 版）** | 提供 `hipcc`、clang、HIP 头文件与运行时 DLL（`rocblas/hipblas` 等）。**这是 HIP 引擎的核心依赖** |
| **Visual Studio Build Tools（含 C++ x64 工作负载）** | ROCm 的 clang 是 MSVC-ABI 编译器，需要 MSVC 头文件/库 |
| **cmake + ninja** | 构建工具（可单独装，或用 VS BuildTools 自带的） |
| **git** | 拉取源码 |
| （可选）**Vulkan SDK** | 仅当你要同时编 Vulkan 后端时 |

### 1.2 关键：先确定你机器上的真实路径

**第 1 步：找 HIP SDK**
```powershell
# 在 PowerShell 里查环境变量 / 常用安装位置
Get-ChildItem C:\ -Recurse -Filter hipcc.exe -ErrorAction SilentlyContinue | Select-Object -First 1
# 或看环境变量
$env:HIP_PATH
```
`«HIP_SDK»` = `hipcc.exe` 所在目录（`...\bin`）的**上一级目录**。
确认里面有 `include\hip\hip_runtime.h`：
```powershell
Test-Path "«HIP_SDK»\include\hip\hip_runtime.h"
```
> 找不到 `hip_runtime.h` 说明 HIP SDK 不完整，先重装。

**第 2 步：找 Visual Studio Build Tools**
```powershell
Get-ChildItem "C:\Program Files (x86)\Microsoft Visual Studio" -Directory -ErrorAction SilentlyContinue
# 找包含 VC\Auxiliary\Build\vcvars64.bat 的那个
Test-Path "«VS_BUILDTOOLS»\VC\Auxiliary\Build\vcvars64.bat"
```

**第 3 步：确认 cmake / ninja**
```powershell
Get-Command cmake, ninja -ErrorAction SilentlyContinue
# 若为空，说明不在 PATH，去 VS BuildTools 里找：
Get-ChildItem "«VS_BUILDTOOLS»\Common7\IDE\CommonExtensions\Microsoft\CMake" -Recurse -Include "cmake.exe","ninja.exe" -ErrorAction SilentlyContinue | Select FullName
```

**第 4 步：确认已装的 MSVC 工具集版本**
```powershell
Get-ChildItem "«VS_BUILDTOOLS»\VC\Tools\MSVC" -Directory | Select-Object -ExpandProperty Name
```
> 你会看到类似 `14.44.35207`、`14.51.36231`。**必须存在一个 `14.44` 开头**的版本，见 §4 坑 1。

**第 5 步：确定你的 GPU 架构**
```powershell
# 用任意已装好的 llama.cpp/llama-bench 列出设备，或查 GPU 型号对照表
«你的HIP构建产物的路径»\llama-bench.exe --list-devices
```
| 常见 AMD GPU | 架构名 |
|---|---|
| Strix Halo (Ryzen AI MAX, Radeon 8060S) | `gfx1151` |
| RDNA4 (RX 9000 系) | `gfx1201` / `gfx1211` |
| RDNA3 (RX 7000 系) | `gfx1100`–`gfx1103` |
| CDNA (Instinct 计算卡) | `gfx942` / `gfx950` 等 |

---

## 2. 写一个通用 launcher 脚本（batch）

**强烈建议**把下面内容存成一个 `.bat`（如 `build-hip.bat`），一次性完成 configure + build，
不要手动分步跑（原因见 §4 坑 4 / 坑 7）。

把 `«...»` 替换成你机器上的值后保存（注意 `%PATH%` 保留原有 PATH）：

```bat
@echo on

rem ========== 1) 激活 MSVC x64 环境，并锁定工具集版本 ==========
rem   关键：必须锁 14.44 系！14.5x 会挂（见 §4 坑 1）
call "«VS_BUILDTOOLS»\VC\Auxiliary\Build\vcvars64.bat" -vcvars_ver=«MSVC_VER» || exit /b 1

rem ========== 2) 置 PATH：cmake + ninja + HIP 的 bin 与 clang 目录 ==========
rem   注意：要同时放 HIP 的 bin（供 hipcc/运行时 DLL）和 clang 目录（供 clang 被 CMake 找到）
set "PATH=«CMAKE_BIN»;«NINJA_BIN»;«HIP_SDK»\bin;«CLANG_BIN»;%PATH%"

set "SRC=«REPO_DIR»"
set "BLD=«BUILD_DIR»"

rem ========== 3) 配置：仅 HIP，指向你的 GPU ==========
cmake -S "%SRC%" -B "%BLD%" -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_C_COMPILER=clang ^
  -DCMAKE_CXX_COMPILER=clang++ ^
  -DGGML_HIP=ON ^
  -DGGML_HIP_FORCE_MMQ=ON ^
  -DGGML_HIP_ROCWMMA_FATTN=OFF ^
  -DGGML_CUDA=OFF ^
  -DGGML_VULKAN=OFF ^
  -DCMAKE_HIP_ARCHITECTURES=«GPU_ARCH» ^
  -DGPU_TARGETS=«GPU_ARCH» ^
  -DAMDGPU_TARGETS=«GPU_ARCH» ^
  "-DCMAKE_HIP_FLAGS=--rocm-path=«HIP_SDK»" || exit /b 1

rem ========== 4) 编译（一次全量，别单编） ==========
cmake --build "%BLD%" -j 32 --target llama-server llama-cli llama-bench llama-quantize test-backend-ops || exit /b 1
```

运行：`cmd /c build-hip.bat`（PowerShell 里用 `& cmd.exe /c build-hip.bat`）。

> **`-j 32` 请改成你的 CPU 核数**。用 `[Environment]::ProcessorCount` 查，或直接去掉 `-j`（让 CMake 默认）。

---

## 3. 编译参数说明（每台机器通用）

- `-DGGML_HIP=ON` —— 开启 HIP 后端（即 ROCm 引擎）。
- `-DGGML_HIP_FORCE_MMQ=ON` —— ROCmFPX 4-bit 内核**必须**开 MMQ 强制。
- `-DGGML_HIP_ROCWMMA_FATTN=OFF` —— 关闭 ROCWMMA flash-attn（RDNA35 平台防不兼容内核；若你完全不用相关内核可保持 OFF）。
- `-DGGML_CUDA=OFF` / `-DGGML_VULKAN=OFF` —— 只编 HIP，避免误开其它后端。
- `-DCMAKE_HIP_ARCHITECTURES / -DGPU_TARGETS / -DAMDGPU_TARGETS = «GPU_ARCH»` —— **三者都设**，指向你的 GPU。
- `-DCMAKE_HIP_FLAGS="--rocm-path=«HIP_SDK»"` —— 让 HIP 找到非标准位置（或默认未注册）的 ROCm 工具链；若 HIP SDK 装在标准路径且已注册，这项可留空。
- `-DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++` —— 用**名称**而非完整路径（见 §4 坑 2）。

编译成功后，`«BUILD_DIR»\bin\` 下应出现 **`ggml-hip.dll`** 和若干 `llama-*.exe`（thin launcher）。

---

## 4. 验证

**务必先把 HIP 运行时 DLL 目录加到 PATH**（`rocblas/hipblas` 等不在构建产物里，直接读 SDK）：

```powershell
$env:PATH = "«HIP_SDK»\bin;$env:PATH"
& "«BUILD_DIR»\bin\llama-bench.exe" --list-devices
```

预期输出类似（出现 `ROCm0` 即成功）：

```
ggml_cuda_init: found 1 ROCm devices (Total VRAM: ...):
  Device 0: AMD Radeon(TM) ... Graphics, gfx1151 (...), VMM: no, Wave Size: 32
Available devices:
  ROCm0: AMD Radeon(TM) ... Graphics (... MiB, ... MiB free)
```

---

## 5. 失败教训：这些事千万别做（本机实测踩坑）

> **核心原则：只按 §2 的 launcher 一次性跑完 configure + build，别拆开、别改环境、别打补丁、别碰系统头文件。**

### ❌ 坑 1：用默认 `vcvars64.bat`（会拿到最新的 MSVC 14.5x）

不带 `-vcvars_ver` 时，VC 默认挑**最新**工具集。若你装了 `14.51`，`<cmath>` 的新 `_CLANG_BUILTIN` 块会把
`isgreater/isless/isunordered...` 声明为 `constexpr`，ROCm clang 再重声明成 `__device__` →
每个 `.cu` 报 `"__device__ function 'isgreater' cannot overload __host__ __device__ function 'isgreater'"`。
**必须 `-vcvars_ver=14.44`**（或你机器上 `14.44` 开头的版本）。

### ❌ 坑 2：用完整编译器路径，或 PATH 里混入了错误的 ROCm 工具链

- **用 `-DCMAKE_C_COMPILER=clang`（名称）**，让 CMake 经 PATH 解析到目标 clang。
- 若你机器上有**多套** ROCm（比如某些工具缓存目录），**别**把非目标那套加到 PATH 前面，否则解析到错误的组件。
- 目标：`PATH` 里放 `«HIP_SDK»\bin` 和 `«CLANG_BIN»`。

### ❌ 坑 3：PATH 漏掉 clang 所在的目录

`clang.exe`/`clang++.exe` 通常不在 `«HIP_SDK»\bin`（那里只有 `hipcc/hipconfig`），而在 `«HIP_SDK»\lib\llvm\bin`。
漏了这个目录，CMake 报 `Tell CMake where to find the compiler ... clang`。
有些预装运行时目录（如某些 cache 目录）是**运行时 DLL**，不含 clang，帮不上忙。

### ❌ 坑 4：孤立单编某个 `.cu` 来"隔离问题"

单编时环境往往是脏的（历史 PATH/变量残留），一个 `.cu` 单编失败不等同于全量失败。正确做法是
**一次 `cmake --build --target ... -j 32` 全量**，在干净的 launcher bat 里跑。

### ❌ 坑 5：改 clang 工具链的头文件（如 `__clang_hip_runtime_wrapper.h`）

网上流传的 LLVM PR #201563 "reorder forward-declares 到 `<cmath>` 前"补丁**只对 device-only 编译有效**，
对 ggml 的 unified host+device 编译**无效**（实测仍报 `isgreater` 冲突，还引入新的 `fabsf` 报错）。
去改 `lib\clang\23\include\__clang_hip_runtime_wrapper.h` 属饮鸩止渴：无效、污染工具链、且一旦
工具链/VS 更新就被覆盖。**根因是 MSVC 版本，不是头文件**——改回 14.44 就干净解决。

### ❌ 坑 6：改 MSVC STL 的 `<cmath>`（无权限，且不该改）

别去修改 `...\VC\Tools\MSVC\14.51.36231\include\cmath`。它在 `Program Files` 下**无写权限**（`Copy-Item` 会
`Access denied`），且改 VS 组件等于自找麻烦。正解还是回 14.44。

### ❌ 坑 7：反复 `Remove-Item` 删 build 目录 + 散落零碎命令

每次删目录、换不同环境拼长命令，环境会越走越偏。正确姿势：一次写好 launcher（§2），固定
`-vcvars_ver` + 固定 PATH + 固定 configure；要重建就删目录后**再跑同一条 bat**。

---

## 6. 一句话总结

HIP 构建成功的充要条件 =
**MSVC 工具集 `14.44` 系** + **正确的 HIP SDK（PATH 含其 `bin` 与 clang 目录，别混入其它 ROCm）** +
**名称编译器 `clang`/`clang++`** + **一次全量 `cmake --build`**。
其余任何"改头文件 / 拆开单编 / 用 14.5x / 混用多套 ROCm / 改 MSVC STL"的尝试，都会踩坑失败。

---

## 7. 参考资料

- 本机实测版：`BUILD-ROCM-ENGINE-WINDOWS.md`（含绝对路径，供本机对照）
- 参考脚本：`amd-rocmfpx-for-win\llm-inference\Setup-ROCmFPX.ps1`（Windows 构建 ROCmFPX 先例）
- 上游构建脚本：`«REPO_DIR»\scripts\build-strix-rocmfp4-mtp.sh`（Linux 等价步骤）
- 官方 HIP 说明：`ROCmFPX` 仓库 README → `Supported backends` / `HIP`
