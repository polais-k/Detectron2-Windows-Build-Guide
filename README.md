# Detectron2 Windows 源码编译与环境适配手册
**本手册面向 Windows 平台下 Detectron2 的源码安装与本地编译，重点解决 Detectron2 官方环境支持与 Windows 实际开发环境之间的适配问题。** 与 Linux 环境下相对直接的安装方式不同，Detectron2 在 Windows 上进行源码编译时，需要同时协调 Python、PyTorch、CUDA Toolkit、NVIDIA 驱动、Visual Studio、MSVC 工具集以及 Windows SDK 等多个组件之间的版本兼容关系。本手册基于一次实际的 Windows 11 + RTX 4090 环境部署过程，完整记录从 Visual Studio C++ 编译环境配置、CUDA Toolkit 与 PyTorch CUDA 版本对齐、MSVC 工具集选择，到 Detectron2 C++/CUDA 扩展成功编译与 Editable Installation 的全过程，并针对安装过程中出现的 CUDA 版本不匹配、MSVC 版本不受支持、环境变量配置错误以及 `WinError 2` 等典型问题给出定位与解决方法。


如果此文档帮助你成功完成了 Detectron2 在 Windows 环境下的安装与编译，欢迎为本仓库点一个 ⭐ Star。希望这份基于实际安装与故障排查过程整理的记录，也能帮助更多开发者减少环境配置与版本兼容问题带来的时间成本。

# 1. 基础硬件与软件环境

本文档所记录的 Detectron2 安装与编译过程基于以下实际环境。由于 Detectron2 包含需要本地编译的 C++/CUDA 扩展，其安装结果不仅取决于 Python 包依赖，还受到 GPU、NVIDIA 驱动、PyTorch CUDA 构建版本、CUDA Toolkit 以及 MSVC 编译工具链之间兼容关系的影响。因此，在开始安装前，应首先明确当前计算机的基础环境。

|项目|当前配置|
|---|---|
|操作系统|Windows 11|
|GPU|NVIDIA GeForce RTX 4090|
|NVIDIA Driver|546.65|
|`nvidia-smi` 显示 CUDA Version|12.3|
|原有 CUDA Toolkit / `nvcc`|CUDA 12.4|
|Python|3.9.18|
|PyTorch|2.0.1|
|PyTorch CUDA|cu118 / CUDA 11.8|
|Conda 环境|`fas`|
|Detectron2|源码编译安装|
|Visual Studio|Visual Studio 2022 Community|
|初始 MSVC|14.44.35207|
|最终编译 MSVC|14.36.32532|
|最终编译 CUDA Toolkit|CUDA 11.8|

需要特别区分三个容易混淆的 CUDA 版本：

```text
nvidia-smi        → CUDA Version 12.3
nvcc --version    → CUDA Toolkit 12.4（初始环境）
torch.version.cuda → CUDA 11.8
```

其中，`nvidia-smi` 显示的 `CUDA Version 12.3` 表示当前 NVIDIA 驱动能够支持的最高 CUDA 运行时版本，并不表示当前用于源码编译的 CUDA Toolkit 版本。Detectron2 在编译 CUDA 扩展时实际使用 `nvcc`，因此需要关注 `nvcc --version` 和 `CUDA_HOME`；同时，PyTorch 的 CUDA 扩展编译要求本地 CUDA Toolkit 与 PyTorch 的 CUDA 构建版本保持一致。本环境中的 PyTorch 2.0.1 为 `cu118`，因此最终额外安装 CUDA Toolkit 11.8，并在 Detectron2 编译过程中使用 CUDA 11.8，而保留系统原有 CUDA 12.4。


# 2. Windows 编译环境准备与版本适配

Detectron2 并非纯 Python 库，其 `detectron2/layers/csrc` 目录包含需要本地编译的 C++/CUDA 扩展。因此，在 Windows 平台上从源码安装 Detectron2，不能仅完成 Python 依赖安装，还必须建立完整的本地编译工具链。本次实际安装过程中，最终可用的编译链为：

```text
Visual Studio 2022 Community
        ↓
MSVC v143 14.36.32532
        ↓
Windows SDK
        ↓
CUDA Toolkit 11.8 / nvcc
        ↓
PyTorch 2.0.1 + cu118
        ↓
Detectron2 C++/CUDA Extension
```

## 2.1 安装 Visual Studio 2022 Community

首先安装 Visual Studio 2022 Community。安装 Visual Studio 的目的并非使用其 IDE 开发 Detectron2，而是获得 Windows 平台编译 PyTorch C++/CUDA Extension 所需的 MSVC 编译器、链接器以及 Windows SDK。

在 Visual Studio Installer 中，应确保安装 C++ 桌面开发相关组件。安装完成后，至少需要能够找到以下工具：

```bat
where cl
where link
where rc
where mt
```

其中：

```text
cl.exe    C/C++ 编译器
link.exe  链接器
rc.exe    Windows Resource Compiler
mt.exe    Manifest Tool
```

本次安装过程中曾出现 WebView2 安装失败，但 Visual Studio Community 本体仍能够正常启动，MSVC 编译工具也能够正常使用。因此，判断 Visual Studio 是否满足 Detectron2 编译要求，不应仅依据安装器是否出现非核心组件错误，而应直接检查上述编译工具是否可用。

## 2.2 安装与 PyTorch 匹配的 CUDA Toolkit

本机原有环境为：

```text
NVIDIA Driver       546.65
原有 nvcc           CUDA Toolkit 12.4
PyTorch             2.0.1
torch.version.cuda   11.8
```

直接使用 CUDA 12.4 编译时，PyTorch 会报错：

```text
Detected CUDA version (12.4) mismatches
the version that was used to compile PyTorch (11.8)
```

因此，本次安装保留系统原有 CUDA 12.4，同时额外安装 CUDA Toolkit 11.8，使两个 Toolkit 并存。CUDA 11.8 安装程序选择：

```text
Operating System: Windows
Architecture:     x86_64
Version:          11
Installer Type:   exe (local)
```

安装时选择“自定义安装”，仅保留 CUDA Toolkit，不安装旧版 NVIDIA Display Driver，也不需要安装 PhysX：

```text
☑ CUDA
☐ Other Components / PhysX
☐ Driver Components
    ☐ Display Driver
    ☐ HD Audio Driver
```

默认安装目录为：

```text
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
```

安装完成后，不需要删除 CUDA 12.4，也不需要修改公共计算机的全局 CUDA 配置。编译 Detectron2 时只需在当前终端临时指定 CUDA 11.8：

```bat
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
```

这里必须注意环境变量的书写方式。错误写法：

```bat
set CUDA_HOME= C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
```

会在路径前引入一个不可见的空格，使 PyTorch 最终尝试执行：

```text
' C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8\bin\nvcc'
 ^
 前导空格
```

并导致：

```text
[WinError 2] 系统找不到指定的文件
```

因此，Windows CMD 中建议统一采用：

```bat
set "变量名=变量值"
```

验证配置：

```bat
where nvcc
nvcc --version

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print(torch.version.cuda); print(repr(CUDA_HOME))"
```

本次最终结果为：

```text
torch.version.cuda = 11.8
CUDA_HOME          = C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
nvcc               = CUDA 11.8
```

## 2.3 安装与 CUDA 11.8 兼容的 MSVC 工具集

Visual Studio 2022 Community 初始安装的最新 MSVC 为：

```text
MSVC 14.44.35207
cl   19.44.35207
```

虽然它属于 Visual Studio 2022，但 CUDA 11.8 发布较早，无法直接接受较新的 MSVC 14.44。编译 Detectron2 CUDA 源码时出现：

```text
fatal error C1189:
#error: -- unsupported Microsoft Visual Studio version!
```

因此，在 Visual Studio Installer 的“单个组件”中额外安装：

```text
MSVC v143 - VS 2022 C++ x64/x86 生成工具
(v14.36-17.6)
```

不需要安装：

```text
Spectre 缓解库
ARM 生成工具
ARM64 生成工具
```

安装完成后，本机存在两个 MSVC 工具集：

```text
14.36.32532
14.44.35207
```

二者可以并存，无需卸载最新版本。

## 2.4 显式激活 MSVC 14.36

仅安装旧版 MSVC 并不意味着系统会自动使用它。Visual Studio 默认开发环境通常仍会选择最新的 14.44，因此必须在编译 Detectron2 前显式指定 14.36：

```bat
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36
```

验证：

```bat
cl
where cl
```

正确结果应指向：

```text
D:\app\MicrosoftC\VC\Tools\MSVC\14.36.32532\bin\HostX64\x64\cl.exe
```

并显示：

```text
Microsoft C/C++ Optimizing Compiler Version 19.36.32532 for x64
```

## 2.5 最终编译环境确认

正式编译 Detectron2 前，应确认：

```bat
where cl
where link
where rc
where mt
where ninja
where nvcc
```

本次成功编译所使用的核心环境为：

|组件|最终版本|
|---|---|
|Python|3.9.18|
|PyTorch|2.0.1|
|PyTorch CUDA|cu118|
|CUDA Toolkit|11.8|
|NVCC|11.8|
|Visual Studio|2022 Community|
|MSVC|14.36.32532|
|Windows SDK|10.0.26100.0|
|GPU|RTX 4090|

至此，Windows 本地 C++/CUDA 编译环境准备完成。下一节即可进入 **Detectron2 源码获取、Python 依赖安装、Editable Installation 与 C++/CUDA Extension 编译**。

# 3. Detectron2 源码安装与本地编译

完成 CUDA Toolkit、MSVC 和 Windows SDK 的环境适配后，可以开始安装 Detectron2。由于后续需要基于 Detectron2 修改 Faster R-CNN、RPN、ROI Heads 以及其他检测模块，本手册采用**源码目录 + Editable Installation** 的开发方式，而不是将 Detectron2 作为不可修改的普通第三方包安装。

## 3.1 激活 Python 环境

本次使用已有 Conda 环境：

```bat
conda activate fas
```

确认 Python 与 PyTorch：

```bat
python --version
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
```

本次环境应为：

```text
Python 3.9.18
PyTorch 2.0.1
PyTorch CUDA 11.8
CUDA Available True
```

需要注意，`torch.version.cuda` 表示当前 PyTorch 二进制包的 CUDA 构建版本。由于本环境为 `cu118`，后续编译 Detectron2 CUDA Extension 时使用 CUDA Toolkit 11.8。

## 3.2 激活 MSVC 14.36 编译环境

在 Conda 环境激活后，再显式加载 MSVC 14.36：

```bat
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36
```

验证：

```bat
cl
where cl
```

当前使用的编译器应为：

```text
MSVC 14.36.32532
cl 19.36.32532
```

建议采用“**先激活 Conda，再激活 MSVC**”的顺序，避免后续 Conda 环境切换改变已经配置好的编译环境。

## 3.3 指定 CUDA Toolkit 11.8

在同一 CMD 终端中执行：

```bat
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
set "DISTUTILS_USE_SDK=1"
```

其中：

```text
CUDA_HOME          指定 PyTorch CUDA Extension 使用的 CUDA Toolkit
PATH               确保优先调用 CUDA 11.8 的 nvcc.exe
DISTUTILS_USE_SDK  告知 Python 构建系统使用当前已激活的 MSVC SDK 环境
```

验证：

```bat
where nvcc
nvcc --version
echo [%CUDA_HOME%]
```

进一步从 PyTorch 内部确认：

```bat
python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print('Torch CUDA:', torch.version.cuda); print('CUDA_HOME:', repr(CUDA_HOME))"
```

应确认：

```text
Torch CUDA: 11.8
CUDA_HOME: 'C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8'
```

## 3.4 进入 Detectron2 源码目录

本次源码目录为：

```text
F:\owod\detectron2
```

进入项目：

```bat
cd /d F:\owod\detectron2
```

对于后续算法开发，该目录本身就是实际使用的 Detectron2 源码。例如：

```text
F:\owod\detectron2\
├── detectron2\
│   ├── modeling\
│   ├── layers\
│   ├── data\
│   ├── engine\
│   └── ...
├── configs\
├── tools\
├── setup.py
└── pyproject.toml
```

因此，后续对 `detectron2/modeling` 等 Python 模块的修改就是对实际运行代码的修改。

## 3.5 安装编译依赖

首先确认 Ninja 可用：

```bat
python -m pip install ninja
```

验证：

```bat
where ninja
ninja --version
```

Ninja 是 PyTorch C++ Extension 使用的并行构建工具。缺少 Ninja 或 Ninja 不在当前环境的 `PATH` 中，都可能导致本地扩展构建失败。

对于本次环境，`setuptools==80.9.0` 在编译过程中会出现：

```text
pkg_resources is deprecated as an API
```

该信息属于弃用警告，并非本次 Detectron2 编译失败的原因。只要没有进一步出现实际异常，不应将该 warning 作为故障处理。

## 3.6 首次编译 Detectron2 C++/CUDA Extension

正式执行 Editable Installation 前，可以先直接编译 Detectron2 的原生扩展：

```bat
python setup.py build_ext --inplace
```

该命令会编译 `detectron2/layers/csrc` 中的 C++/CUDA 源码，并将生成的 `_C` 扩展直接复制到 Detectron2 源码目录。

成功时，日志末尾出现：

```text
正在生成代码
已完成代码的生成
copying build\lib.win-amd64-cpython-39\detectron2\_C.cp39-win_amd64.pyd -> detectron2
```

其中：

```text
detectron2\_C.cp39-win_amd64.pyd
```

是 Python 3.9、Windows x64 环境下生成的 Detectron2 原生 C++/CUDA 扩展。该文件成功生成意味着 MSVC、CUDA Toolkit、NVCC、PyTorch C++ Extension 和 Detectron2 源码已经完成实际编译链验证。

## 3.7 执行 Editable Installation

完成本地扩展编译后，在 Detectron2 根目录执行：

```bat
python -m pip install -e . --no-build-isolation
```

其中：

```text
-e
```

表示 Editable Installation，即开发模式安装。Python 环境不会使用一份与当前源码目录完全独立的 Detectron2 副本，而是将当前源码工程作为实际开发代码来源。

因此，后续修改：

```text
F:\owod\detectron2\detectron2\modeling\...
```

中的 Python 代码，通常不需要重新执行 `pip install`，重新启动 Python 进程后即可使用修改后的实现。

但如果修改：

```text
detectron2\layers\csrc\
```

中的 C++ 或 CUDA 源码，则必须重新编译原生扩展：

```bat
python setup.py build_ext --inplace
```

## 3.8 安装结果验证

首先验证 Detectron2 的实际加载位置：

```bat
python -c "import detectron2; print(detectron2.__file__)"
```

输出应指向当前源码工程，例如：

```text
F:\owod\detectron2\detectron2\__init__.py
```

验证 C++/CUDA Extension：

```bat
python -c "from detectron2 import _C; print('Detectron2 C++/CUDA extension OK')"
```

验证 GPU：

```bat
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

本次环境预期为：

```text
True
NVIDIA GeForce RTX 4090
```

至此，Detectron2 已完成 Windows 平台下的源码编译与开发模式安装。后续可以直接基于当前源码工程修改 Faster R-CNN、RPN、ROI Heads、Loss Function 以及自定义 C++/CUDA Operator。

# 4. 安装过程中的典型错误、原因与排查方法

Windows 平台下安装 Detectron2 的主要困难通常不在 Python 依赖本身，而在于 **PyTorch、CUDA Toolkit、MSVC、Windows SDK 与 Python 构建系统之间的协同关系**。本次安装过程中实际遇到了多个具有代表性的问题。本节按照实际故障链路记录问题现象、定位过程、根本原因及解决方法。

## 4.1 Visual Studio 安装过程中 WebView2 安装失败

安装 Visual Studio 2022 Community 时，安装程序可能提示 WebView2 相关组件安装失败。此类错误不一定意味着 Visual Studio 的 C++ 编译环境安装失败。

对于 Detectron2 而言，真正需要确认的是以下工具是否存在：

```bat
where cl
where link
where rc
where mt
```

若能够正常找到：

```text
cl.exe
link.exe
rc.exe
mt.exe
```

且 Visual Studio Community 可以正常启动，则 WebView2 的单独安装失败通常不会阻止 Detectron2 的 C++/CUDA 扩展编译。

因此，应区分：

```text
Visual Studio IDE 附加组件安装问题
            ≠
MSVC C++ 编译工具链不可用
```

不要因为 WebView2 单一组件失败而立即重新安装整个 Visual Studio。

---

## 4.2 PyTorch CUDA 11.8 与本地 CUDA Toolkit 12.4 不匹配

### 问题现象

本机最初的环境为：

```text
PyTorch 2.0.1 + cu118
torch.version.cuda = 11.8
nvcc --version     = 12.4
```

编译 Detectron2 时出现 CUDA 版本不一致错误：

```text
Detected CUDA version (12.4) mismatches
the version that was used to compile PyTorch (11.8)
```

### 原因

需要区分三个不同概念：

```text
nvidia-smi 中的 CUDA Version
        ↓
NVIDIA 驱动能够支持的 CUDA 版本

nvcc --version
        ↓
当前 CUDA Toolkit 编译器版本

torch.version.cuda
        ↓
当前 PyTorch 二进制包的 CUDA 构建版本
```

Detectron2 编译 CUDA Extension 时，需要通过 PyTorch 的 `cpp_extension` 调用本地 `nvcc`。因此，本地 CUDA Toolkit 应与 `torch.version.cuda` 保持匹配。

### 解决方法

保留原有 CUDA Toolkit 12.4，同时安装 CUDA Toolkit 11.8，并在当前编译终端中指定：

```bat
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
```

验证：

```bat
where nvcc
nvcc --version

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print(torch.version.cuda); print(repr(CUDA_HOME))"
```

最终应满足：

```text
torch.version.cuda = 11.8
CUDA_HOME          = CUDA\v11.8
nvcc               = CUDA 11.8
```

---

## 4.3 `[WinError 2] 系统找不到指定的文件`

### 问题现象

执行：

```bat
python -m pip install -e . --no-build-isolation
```

或者：

```bat
python setup.py build_ext --inplace
```

时，仅得到：

```text
running build_ext
error: [WinError 2] 系统找不到指定的文件。
```

此错误没有直接指出缺失的具体文件，因此容易被误判为 MSVC、Windows SDK、Git、Ninja 或 CUDA Toolkit 未安装。

### 初步排查

依次确认：

```bat
where cl
where link
where rc
where mt
where ninja
where nvcc
where git
```

本次环境中上述程序均正常存在，因此问题并不是工具本身缺失。

进一步确认：

```bat
python -c "from torch.utils.cpp_extension import CUDA_HOME; import torch; print('torch CUDA=',torch.version.cuda); print('CUDA_HOME=',CUDA_HOME)"
```

表面上输出：

```text
torch CUDA= 11.8
CUDA_HOME=  C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
```

但实际上 `CUDA_HOME` 路径前存在一个前导空格。

### 精确定位

通过跟踪 Python `subprocess.Popen` 调用，最终发现 PyTorch 实际执行的是：

```text
[' C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8\\bin\\nvcc',
 '--version']
```

注意：

```text
' C:\Program Files\...
 ^
 路径前存在空格
```

Windows 因此将整个带前导空格的字符串视为可执行文件路径，并返回：

```text
[WinError 2] 系统找不到指定的文件
```

### 根本原因

错误的环境变量设置方式：

```bat
set CUDA_HOME= C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
              ^
              等号后存在空格
```

### 解决方法

重新设置：

```bat
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
```

验证时建议使用 `repr()`，因为普通 `print()` 不容易观察路径中的前导空格：

```bat
python -c "from torch.utils.cpp_extension import CUDA_HOME; print(repr(CUDA_HOME))"
```

正确输出：

```text
'C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8'
```

该问题说明，在 Windows 环境变量排查中，**“路径存在”并不意味着“传递给 subprocess 的路径字符串正确”**。对于无法解释的 `WinError 2`，应检查不可见空格、引号和 PATH 顺序。

---

## 4.4 CUDA 11.8 不支持当前 MSVC 14.44

### 问题现象

修复 `CUDA_HOME` 后，NVCC 已经能够真正启动，并开始编译 Detectron2 的 CUDA 源码，例如：

```text
deform_conv_cuda_kernel.cu
ROIAlignRotated_cuda.cu
```

但编译失败：

```text
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8\
include\crt\host_config.h(153):

fatal error C1189:
#error: -- unsupported Microsoft Visual Studio version!
```

### 原因

当前 Visual Studio 2022 默认使用：

```text
MSVC 14.44.35207
cl 19.44.35207
```

CUDA 11.8 发布较早，无法识别和正式支持后续发布的 MSVC 14.44。虽然二者都属于 Visual Studio 2022 生态，但：

```text
Visual Studio 2022
```

并不等价于：

```text
所有 VS 2022 生命周期内发布的 MSVC 工具集
均受到 CUDA 11.8 支持
```

### 解决方法

在 Visual Studio Installer 中进入：

```text
Visual Studio 2022 Community
→ 修改
→ 单个组件
```

额外安装：

```text
MSVC v143 - VS 2022 C++ x64/x86 生成工具
(v14.36-17.6)
```

无需安装：

```text
Spectre 缓解库
ARM 生成工具
ARM64 生成工具
```

安装完成后：

```bat
dir /b "D:\app\MicrosoftC\VC\Tools\MSVC"
```

本次环境显示：

```text
14.36.32532
14.44.35207
```

两个 MSVC 版本可以并存，不需要卸载 14.44。

---

## 4.5 安装旧版 MSVC 后仍然使用最新版本

仅安装 MSVC 14.36 并不会自动让 NVCC 使用该版本。Visual Studio 默认环境仍可能加载：

```text
MSVC 14.44
```

因此必须显式激活：

```bat
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36
```

验证：

```bat
cl
where cl
```

必须确认当前实际调用：

```text
...\MSVC\14.36.32532\bin\HostX64\x64\cl.exe
```

而不是：

```text
...\MSVC\14.44.35207\bin\HostX64\x64\cl.exe
```

随后重新清理旧构建文件：

```bat
rmdir /s /q build
```

再执行：

```bat
python setup.py build_ext --inplace
```

最终成功生成：

```text
detectron2\_C.cp39-win_amd64.pyd
```

---

## 4.6 `pkg_resources is deprecated` 警告

编译过程中出现：

```text
UserWarning: pkg_resources is deprecated as an API.
The pkg_resources package is slated for removal...
```

并提示：

```text
pin to Setuptools<81
```

本次环境中的 Setuptools 为：

```text
80.9.0
```

该信息属于 PyTorch 2.0.1 中旧代码使用 `pkg_resources` 所产生的弃用警告，不是导致 Detectron2 编译失败的直接原因。

因此：

```text
Warning ≠ Build Error
```

只要真正的异常发生在其他位置，就不应围绕该 warning 反复调整 Setuptools。排查编译问题时，应优先寻找：

```text
fatal error
error Cxxxx
nvcc fatal
FAILED:
Traceback
```

---

## 4.7 `ninja: build stopped: subcommand failed` 不是根本错误

编译失败时，日志末尾可能出现：

```text
ninja: build stopped: subcommand failed.
```

随后是：

```text
subprocess.CalledProcessError
RuntimeError: Error compiling objects for extension
```

这些都是**上游编译任务失败后的二次异常**，不是根本原因。

正确的排查方向是向上查找：

```text
FAILED:
```

附近的第一条实际编译错误。例如本次真正的根因是：

```text
fatal error C1189:
unsupported Microsoft Visual Studio version
```

因此，阅读大型编译日志时，应遵循：

```text
最终 RuntimeError
        ↑
Ninja 构建失败
        ↑
某个 .cu/.cpp 编译任务 FAILED
        ↑
第一条 fatal error / compiler error
        ↑
真正根因
```

---

# 5. 推荐的完整启动与编译流程

完成首次安装后，如果后续需要重新编译 Detectron2，建议每次新建 CMD 后按照固定顺序初始化环境：

```bat
conda activate fas

call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36

set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
set "DISTUTILS_USE_SDK=1"

cd /d F:\owod\detectron2
```

随后验证：

```bat
where cl
where nvcc

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print('PyTorch:', torch.__version__); print('Torch CUDA:', torch.version.cuda); print('CUDA_HOME:', repr(CUDA_HOME))"
```

如果只修改 Detectron2 的 Python 代码，无需重新安装或编译；如果修改 `detectron2/layers/csrc` 下的 C++/CUDA 代码，则执行：

```bat
rmdir /s /q build
python setup.py build_ext --inplace
```

对于本次环境，最终稳定的编译关系为：

```text
RTX 4090
    ↓
NVIDIA Driver 546.65
    ↓
PyTorch 2.0.1 + cu118
    ↕ 版本对齐
CUDA Toolkit 11.8
    ↓
NVCC 11.8
    ↕ 编译器兼容
MSVC v143 14.36.32532
    ↓
Windows SDK 10.0.26100.0
    ↓
Detectron2 C++/CUDA Extension
    ↓
_C.cp39-win_amd64.pyd
```

这套配置的核心原则不是“所有组件都使用最新版本”，而是**围绕既有 PyTorch 版本建立相互兼容的 CUDA 与 MSVC 编译工具链**。对于需要保留公共计算机现有软件环境的场景，应优先采用 CUDA Toolkit 多版本并存、MSVC 工具集多版本并存以及当前终端局部环境变量配置，而不是卸载或替换系统已有组件。


