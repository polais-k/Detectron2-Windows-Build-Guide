# Detectron2 Windows Source Compilation and Environment Adaptation Manual
This manual guides you through the source code installation and local compilation of Detectron2 on the Windows platform, with a primary focus on resolving compatibility issues between Detectron2's officially supported environments and practical Windows development setups. Unlike the relatively straightforward installation process on Linux, compiling Detectron2 from source on Windows requires carefully coordinating version compatibility across multiple components: Python, PyTorch, the CUDA Toolkit, NVIDIA drivers, Visual Studio, the MSVC toolset, and the Windows SDK.
Based on a real-world deployment process on a Windows 11 + RTX 4090 system, this manual fully documents the entire workflow. It covers everything from configuring the Visual Studio C++ build environment, aligning the CUDA Toolkit and PyTorch CUDA versions, and selecting the correct MSVC toolset, to the successful compilation of Detectron2's C++/CUDA extensions and completing an Editable Installation. Furthermore, it provides precise troubleshooting steps and solutions for typical issues encountered during installation, such as CUDA version mismatches, unsupported MSVC versions, incorrect environment variable configurations, and `WinError 2`.

# 1. Basic Hardware and Software Environment

The installation and compilation process of Detectron2 documented in this manual is based on the following actual environment. Since Detectron2 includes C++/CUDA extensions that require local compilation, its installation outcome depends not only on Python package dependencies but is also affected by the compatibility among the GPU, NVIDIA drivers, PyTorch CUDA build version, CUDA Toolkit, and the MSVC compilation toolchain. Therefore, before starting the installation, you must first clarify the current base environment of your computer.

| Item | Current Configuration |
| --- | --- |
| Operating System | Windows 11 |
| GPU | NVIDIA GeForce RTX 4090 |
| NVIDIA Driver | 546.65 |
| `nvidia-smi` CUDA Version | 12.3 |
| Original CUDA Toolkit / `nvcc` | CUDA 12.4 |
| Python | 3.9.18 |
| PyTorch | 2.0.1 |
| PyTorch CUDA | cu118 / CUDA 11.8 |
| Conda Environment | `fas` |
| Detectron2 | Compiled from source |
| Visual Studio | Visual Studio 2022 Community |
| Initial MSVC | 14.44.35207 |
| Final Compilation MSVC | 14.36.32532 |
| Final Compilation CUDA Toolkit | CUDA 11.8 |

It is necessary to specifically distinguish between three easily confused CUDA versions:

```text
nvidia-smi         → CUDA Version 12.3
nvcc --version     → CUDA Toolkit 12.4 (Initial Environment)
torch.version.cuda → CUDA 11.8

```

Among them, the `CUDA Version 12.3` displayed by `nvidia-smi` indicates the highest CUDA runtime version supported by the current NVIDIA driver, not the CUDA Toolkit version currently used for source compilation. Detectron2 actually uses `nvcc` when compiling CUDA extensions, so you need to pay attention to `nvcc --version` and `CUDA_HOME`. Meanwhile, compiling PyTorch's CUDA extensions requires the local CUDA Toolkit to match PyTorch's CUDA build version. Since PyTorch 2.0.1 in this environment is `cu118`, we additionally installed CUDA Toolkit 11.8 and used it during the Detectron2 compilation process, while keeping the system's original CUDA 12.4 intact.

---

# 2. Windows Build Environment Preparation and Version Adaptation

Detectron2 is not a pure Python library; its `detectron2/layers/csrc` directory contains C++/CUDA extensions that require local compilation. Therefore, installing Detectron2 from source on the Windows platform means you cannot simply install Python dependencies—you must also establish a complete local compilation toolchain. During this actual installation process, the final working compilation chain was:

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

## 2.1 Installing Visual Studio 2022 Community

First, install Visual Studio 2022 Community. The purpose of installing Visual Studio is not to use its IDE to develop Detectron2, but to obtain the MSVC compiler, linker, and Windows SDK required to compile PyTorch C++/CUDA Extensions on the Windows platform.

In the Visual Studio Installer, ensure that the components related to "Desktop development with C++" are installed. After the installation is complete, you should at least be able to find the following tools:

```bat
where cl
where link
where rc
where mt

```

Where:

```text
cl.exe    C/C++ Compiler
link.exe  Linker
rc.exe    Windows Resource Compiler
mt.exe    Manifest Tool

```

During this installation, a WebView2 installation failure occurred, but the core Visual Studio Community IDE could still launch normally, and the MSVC build tools functioned perfectly. Therefore, whether Visual Studio meets the Detectron2 compilation requirements should not be judged solely by whether the installer reports errors on non-core components, but rather by directly checking if the aforementioned compilation tools are available.

## 2.2 Installing the CUDA Toolkit Matching PyTorch

The original environment on this machine was:

```text
NVIDIA Driver       546.65
Original nvcc       CUDA Toolkit 12.4
PyTorch             2.0.1
torch.version.cuda  11.8

```

When attempting to compile directly with CUDA 12.4, PyTorch throws an error:

```text
Detected CUDA version (12.4) mismatches
the version that was used to compile PyTorch (11.8)

```

Therefore, this installation keeps the system's original CUDA 12.4 while additionally installing CUDA Toolkit 11.8, allowing both toolkits to coexist. The CUDA 11.8 installer selections are:

```text
Operating System: Windows
Architecture:     x86_64
Version:          11
Installer Type:   exe (local)

```

Choose "Custom (Advanced)" during installation, keeping only the CUDA Toolkit. Do not install the older NVIDIA Display Driver, and there is no need to install PhysX:

```text
☑ CUDA
☐ Other Components / PhysX
☐ Driver Components
    ☐ Display Driver
    ☐ HD Audio Driver

```

The default installation directory is:

```text
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8

```

After installation, there is no need to delete CUDA 12.4 or modify the global CUDA configuration on a shared computer. When compiling Detectron2, you only need to temporarily specify CUDA 11.8 in the current terminal:

```bat
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"

```

> **Important:** You must pay close attention to how environment variables are written here.

Incorrect syntax:

```bat
set CUDA_HOME= C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8

```

This will introduce an invisible space before the path, causing PyTorch to ultimately try and execute:

```text
' C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8\bin\nvcc'
 ^
 Leading space

```

Resulting in:

```text
[WinError 2] The system cannot find the file specified

```

Therefore, in Windows CMD, it is highly recommended to uniformly use:

```bat
set "VariableName=VariableValue"

```

Verify the configuration:

```bat
where nvcc
nvcc --version

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print(torch.version.cuda); print(repr(CUDA_HOME))"

```

The final results for this session were:

```text
torch.version.cuda = 11.8
CUDA_HOME          = C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
nvcc               = CUDA 11.8

```

## 2.3 Installing an MSVC Toolset Compatible with CUDA 11.8

The latest MSVC initially installed by Visual Studio 2022 Community is:

```text
MSVC 14.44.35207
cl   19.44.35207

```

Although it belongs to Visual Studio 2022, CUDA 11.8 was released earlier and cannot directly accept the newer MSVC 14.44. When compiling the Detectron2 CUDA source code, the following error occurs:

```text
fatal error C1189:
#error: -- unsupported Microsoft Visual Studio version!

```

Therefore, you must additionally install the following from the "Individual components" tab in the Visual Studio Installer:

```text
MSVC v143 - VS 2022 C++ x64/x86 build tools (v14.36-17.6)

```

You do not need to install:

```text
Spectre Mitigations
ARM build tools
ARM64 build tools

```

After installation, two MSVC toolsets will exist on the machine:

```text
14.36.32532
14.44.35207

```

Both can coexist; there is no need to uninstall the newer version.

## 2.4 Explicitly Activating MSVC 14.36

Simply installing the older MSVC does not mean the system will automatically use it. Visual Studio's default development environment typically still selects the latest 14.44. Thus, you must explicitly specify 14.36 before compiling Detectron2:

```bat
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36

```

Verify:

```bat
cl
where cl

```

The correct result should point to:

```text
D:\app\MicrosoftC\VC\Tools\MSVC\14.36.32532\bin\HostX64\x64\cl.exe

```

And display:

```text
Microsoft C/C++ Optimizing Compiler Version 19.36.32532 for x64

```

## 2.5 Final Compilation Environment Confirmation

Before officially compiling Detectron2, you should confirm:

```bat
where cl
where link
where rc
where mt
where ninja
where nvcc

```

The core environment used for the successful compilation in this instance was:

| Component | Final Version |
| --- | --- |
| Python | 3.9.18 |
| PyTorch | 2.0.1 |
| PyTorch CUDA | cu118 |
| CUDA Toolkit | 11.8 |
| NVCC | 11.8 |
| Visual Studio | 2022 Community |
| MSVC | 14.36.32532 |
| Windows SDK | 10.0.26100.0 |
| GPU | RTX 4090 |

At this point, the local Windows C++/CUDA compilation environment preparation is complete. In the next section, we will proceed to **fetching the Detectron2 source code, installing Python dependencies, performing an Editable Installation, and compiling the C++/CUDA Extensions**.

# 3. Detectron2 Source Installation and Local Compilation

After completing the environment adaptation for the CUDA Toolkit, MSVC, and Windows SDK, you can begin installing Detectron2. Because we will need to modify Faster R-CNN, RPN, ROI Heads, and other detection modules based on Detectron2 later, this manual adopts the development approach of using the source directory combined with an **Editable Installation**, rather than installing Detectron2 as a standard, unmodifiable third-party package.

---

## 3.1 Activating the Python Environment

We will use an existing Conda environment for this setup:

```bash
conda activate fas

```

Verify Python and PyTorch:

```bash
python --version
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"

```

The environment for this setup should be:

* **Python:** 3.9.18
* **PyTorch:** 2.0.1
* **PyTorch CUDA:** 11.8
* **CUDA Available:** True

Note that `torch.version.cuda` represents the CUDA build version of the current PyTorch binary package. Since this environment uses `cu118`, we will use CUDA Toolkit 11.8 when compiling the Detectron2 CUDA Extension later.

## 3.2 Activating the MSVC 14.36 Build Environment

After activating the Conda environment, explicitly load MSVC 14.36:

```cmd
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36

```

Verify:

```cmd
cl
where cl

```

The compiler currently in use should be:

* MSVC 14.36.32532
* cl 19.36.32532

> **Note:** It is recommended to follow the order of "activate Conda first, then activate MSVC" to prevent subsequent Conda environment switches from altering the already configured build environment.

## 3.3 Specifying CUDA Toolkit 11.8

Execute the following in the same CMD terminal:

```cmd
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
set "DISTUTILS_USE_SDK=1"

```

Where:

* **`CUDA_HOME`**: Specifies the CUDA Toolkit used by the PyTorch CUDA Extension.
* **`PATH`**: Ensures `nvcc.exe` from CUDA 11.8 is prioritized.
* **`DISTUTILS_USE_SDK`**: Instructs the Python build system to use the currently activated MSVC SDK environment.

Verify:

```cmd
where nvcc
nvcc --version
echo [%CUDA_HOME%]

```

Further confirm from within PyTorch:

```bash
python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print('Torch CUDA:', torch.version.cuda); print('CUDA_HOME:', repr(CUDA_HOME))"

```

You should confirm the output is:

```text
Torch CUDA: 11.8
CUDA_HOME: 'C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8'

```

## 3.4 Navigating to the Detectron2 Source Directory

The source directory for this setup is:

```text
F:\owod\detectron2

```

Enter the project:

```cmd
cd /d F:\owod\detectron2

```

For subsequent algorithm development, this directory itself will be the actual Detectron2 source code in use. For example:

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

Therefore, any subsequent modifications to Python modules like `detectron2/modeling` will be changes to the actual code being executed.

## 3.5 Installing Build Dependencies

First, ensure Ninja is available:

```bash
python -m pip install ninja

```

Verify:

```bash
where ninja
ninja --version

```

Ninja is a parallel build tool used by PyTorch C++ Extensions. Missing Ninja or not having it in the current environment's `PATH` can cause local extension builds to fail.

In this environment, `setuptools==80.9.0` will produce the following message during compilation:

```text
pkg_resources is deprecated as an API

```

This is merely a deprecation warning and is not the cause of any Detectron2 compilation failure. As long as no actual exceptions follow, you should not treat this warning as an error.

## 3.6 First-Time Compilation of Detectron2 C++/CUDA Extension

Before officially executing the Editable Installation, you can directly compile Detectron2's native extensions first:

```bash
python setup.py build_ext --inplace

```

This command compiles the C++/CUDA source code in `detectron2/layers/csrc` and copies the generated `_C` extension directly into the Detectron2 source directory.

Upon success, the end of the log will show:

```text
正在生成代码 (Generating code)
已完成代码的生成 (Finished generating code)
copying build\lib.win-amd64-cpython-39\detectron2\_C.cp39-win_amd64.pyd -> detectron2

```

Here, `detectron2\_C.cp39-win_amd64.pyd` is the native Detectron2 C++/CUDA extension generated for the Python 3.9, Windows x64 environment. Successfully generating this file means the actual compilation chain (MSVC, CUDA Toolkit, NVCC, PyTorch C++ Extension, and Detectron2 source) has been verified.

## 3.7 Executing an Editable Installation

After completing the local extension compilation, execute the following in the Detectron2 root directory:

```bash
python -m pip install -e . --no-build-isolation

```

Where `-e` stands for **Editable Installation** (development mode). The Python environment will not use a Detectron2 copy that is completely independent of the current source directory; instead, it uses the current source project as the actual source of development code.

Therefore, when subsequently modifying Python code in:

```text
F:\owod\detectron2\detectron2\modeling\...

```

You generally do not need to re-execute `pip install`. The modified implementation can be used simply by restarting the Python process.

However, if you modify C++ or CUDA source code in:

```text
detectron2\layers\csrc\

```

You **must** recompile the native extensions:

```bash
python setup.py build_ext --inplace

```

## 3.8 Verifying the Installation Results

First, verify the actual loading location of Detectron2:

```bash
python -c "import detectron2; print(detectron2.__file__)"

```

The output should point to the current source project, for example:

```text
F:\owod\detectron2\detectron2\__init__.py

```

Verify the C++/CUDA Extension:

```bash
python -c "from detectron2 import _C; print('Detectron2 C++/CUDA extension OK')"

```

Verify the GPU:

```bash
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"

```

The expected output for this environment is:

```text
True
NVIDIA GeForce RTX 4090

```

At this point, Detectron2 has successfully completed source compilation and development mode installation on the Windows platform. Moving forward, you can directly modify Faster R-CNN, RPN, ROI Heads, Loss Functions, and custom C++/CUDA Operators based on the current source project.


# 4. Typical Errors, Causes, and Troubleshooting Methods During Installation

The primary difficulty when installing Detectron2 on the Windows platform usually lies not in the Python dependencies themselves, but in coordinating the compatibility among PyTorch, the CUDA Toolkit, MSVC, the Windows SDK, and the Python build system. During this installation process, several highly representative issues were encountered. This section documents the symptoms, troubleshooting steps, root causes, and solutions based on the actual failure chain.

## 4.1 WebView2 Installation Failure During Visual Studio Installation

When installing Visual Studio 2022 Community, the installer might prompt that WebView2-related components failed to install. Such an error does not necessarily mean the Visual Studio C++ build environment failed to install.

For Detectron2, what you genuinely need to confirm is the presence of the following tools:

DOS

```
where cl
where link
where rc
where mt
```

If you can successfully find:

- `cl.exe`
    
- `link.exe`
    
- `rc.exe`
    
- `mt.exe`
    

And if the Visual Studio Community IDE can launch normally, an isolated WebView2 installation failure will generally not prevent the compilation of Detectron2's C++/CUDA extensions.

Therefore, distinguish between:

**Visual Studio IDE add-on installation issues** ≠ **MSVC C++ build toolchain unavailability**

Do not immediately reinstall the entire Visual Studio package just because a single component like WebView2 failed.

## 4.2 PyTorch CUDA 11.8 Mismatches with Local CUDA Toolkit 12.4

**Symptom:**

The initial environment on the machine was:

- PyTorch 2.0.1 + cu118
    
- `torch.version.cuda` = 11.8
    
- `nvcc --version` = 12.4
    

Compiling Detectron2 resulted in a CUDA version inconsistency error:

Plaintext

```
Detected CUDA version (12.4) mismatches
the version that was used to compile PyTorch (11.8)
```

**Cause:**

You need to distinguish among three different concepts:

1. **`nvidia-smi` CUDA Version:** The highest CUDA runtime version supported by the NVIDIA driver.
    
2. **`nvcc --version`:** The current CUDA Toolkit compiler version.
    
3. **`torch.version.cuda`:** The CUDA build version of the current PyTorch binary package.
    

When Detectron2 compiles the CUDA Extension, it uses PyTorch's `cpp_extension` to call the local `nvcc`. Therefore, the local CUDA Toolkit must match `torch.version.cuda`.

**Solution:**

Keep the original CUDA Toolkit 12.4 intact, but additionally install CUDA Toolkit 11.8, and specify it in the current compilation terminal:

DOS

```
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
```

Verify:

DOS

```
where nvcc
nvcc --version

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print(torch.version.cuda); print(repr(CUDA_HOME))"
```

The final output should satisfy:

Plaintext

```
torch.version.cuda = 11.8
CUDA_HOME          = CUDA\v11.8
nvcc               = CUDA 11.8
```

## 4.3 [WinError 2] The system cannot find the file specified

**Symptom:**

When executing:

Bash

```
python -m pip install -e . --no-build-isolation
```

Or:

Bash

```
python setup.py build_ext --inplace
```

You only get:

Plaintext

```
running build_ext
error: [WinError 2] The system cannot find the file specified.
```

This error does not explicitly state which file is missing, making it easy to misdiagnose as a missing installation of MSVC, Windows SDK, Git, Ninja, or the CUDA Toolkit.

**Initial Troubleshooting:**

Verify the following sequentially:

DOS

```
where cl
where link
where rc
where mt
where ninja
where nvcc
where git
```

In this environment, all the above programs were present and normal, meaning the issue was not missing tools.

Further verification:

Bash

```
python -c "from torch.utils.cpp_extension import CUDA_HOME; import torch; print('torch CUDA=',torch.version.cuda); print('CUDA_HOME=',CUDA_HOME)"
```

On the surface, it outputted:

Plaintext

```
torch CUDA= 11.8
CUDA_HOME=  C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
```

However, there was actually a leading space before the `CUDA_HOME` path.

**Precise Diagnosis:**

By tracing the Python `subprocess.Popen` call, it was discovered that PyTorch was actually executing:

Python

```
[' C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8\\bin\\nvcc',
 '--version']
```

Notice:

Plaintext

```
' C:\Program Files\...
 ^
 Leading space before the path
```

Because of this, Windows treated the entire string (including the space) as the executable path and returned: `[WinError 2] The system cannot find the file specified`.

**Root Cause:**

An incorrect way of setting environment variables:

DOS

```
set CUDA_HOME= C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8
              ^
              Space after the equal sign
```

**Solution:**

Reset the variable correctly:

DOS

```
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
```

When verifying, it is highly recommended to use `repr()` because a standard `print()` makes it difficult to spot leading spaces in paths:

Bash

```
python -c "from torch.utils.cpp_extension import CUDA_HOME; print(repr(CUDA_HOME))"
```

Correct output:

Plaintext

```
'C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v11.8'
```

This issue highlights that during Windows environment variable troubleshooting, "the path exists" does not mean "the path string passed to subprocess is correct." For unexplained `WinError 2` occurrences, always check for invisible spaces, missing quotes, and `PATH` execution order.

## 4.4 CUDA 11.8 Does Not Support Current MSVC 14.44

**Symptom:**

After fixing `CUDA_HOME`, NVCC successfully started and began compiling Detectron2's CUDA source files, such as:

- `deform_conv_cuda_kernel.cu`
    
- `ROIAlignRotated_cuda.cu`
    

However, the compilation failed with:

Plaintext

```
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8\
include\crt\host_config.h(153):

fatal error C1189:
#error: -- unsupported Microsoft Visual Studio version!
```

**Cause:**

The current Visual Studio 2022 defaults to using:

- MSVC 14.44.35207
    
- cl 19.44.35207
    

CUDA 11.8 was released much earlier and cannot recognize or officially support the subsequently released MSVC 14.44. Although both belong to the Visual Studio 2022 ecosystem, "Visual Studio 2022" does **not** equate to "all MSVC toolsets released during the VS 2022 lifecycle are supported by CUDA 11.8."

**Solution:**

Open the Visual Studio Installer and navigate to:

> Visual Studio 2022 Community → Modify → Individual components

Additionally install:

- `MSVC v143 - VS 2022 C++ x64/x86 build tools (v14.36-17.6)`
    

You do not need to install:

- Spectre Mitigations
    
- ARM build tools
    
- ARM64 build tools
    

After installation:

DOS

```
dir /b "D:\app\MicrosoftC\VC\Tools\MSVC"
```

The environment showed:

Plaintext

```
14.36.32532
14.44.35207
```

Both MSVC versions can coexist; there is no need to uninstall 14.44.

## 4.5 Still Using Latest MSVC After Installing Older Version

Simply installing MSVC 14.36 will not automatically force NVCC to use it. The default Visual Studio environment may still load `MSVC 14.44`.

Therefore, you must explicitly activate it:

DOS

```
call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36
```

Verify:

DOS

```
cl
where cl
```

You must confirm that the current active call points to:

Plaintext

```
...\MSVC\14.36.32532\bin\HostX64\x64\cl.exe
```

And not:

Plaintext

```
...\MSVC\14.44.35207\bin\HostX64\x64\cl.exe
```

Following this, clear out old build files:

DOS

```
rmdir /s /q build
```

Then re-execute:

Bash

```
python setup.py build_ext --inplace
```

This should successfully generate: `detectron2\_C.cp39-win_amd64.pyd`.

## 4.6 `pkg_resources is deprecated` Warning

**Symptom:**

During compilation, the following appears:

Plaintext

```
UserWarning: pkg_resources is deprecated as an API.
The pkg_resources package is slated for removal...
```

Along with a prompt to: `pin to Setuptools<81`.

**Cause:**

In this environment, Setuptools was `80.9.0`. This message is simply a deprecation warning triggered by old code inside PyTorch 2.0.1 utilizing `pkg_resources`. It is **not** the direct cause of the Detectron2 compilation failure.

**Solution:**

Warning ≠ Build Error.

As long as the real exception occurs elsewhere, you should not waste time continuously tweaking Setuptools over this warning. When troubleshooting compilation issues, prioritize looking for:

- `fatal error`
    
- `error Cxxxx`
    
- `nvcc fatal`
    
- `FAILED:`
    
- `Traceback`
    

## 4.7 `ninja: build stopped: subcommand failed` is Not the Root Error

**Symptom:**

When compilation fails, the end of the log might show:

Plaintext

```
ninja: build stopped: subcommand failed.
```

Followed immediately by:

Plaintext

```
subprocess.CalledProcessError
RuntimeError: Error compiling objects for extension
```

**Cause:**

These are all secondary exceptions resulting from an upstream compilation task failure. They are not the root cause.

**Solution:**

The correct troubleshooting approach is to scroll up and look for the first actual compilation error near a `FAILED:` tag. For example, the real root cause in this case was:

Plaintext

```
fatal error C1189:
unsupported Microsoft Visual Studio version
```

Therefore, when reading large compilation logs, follow this bottom-up hierarchy:

Plaintext

```
Final RuntimeError
        ↑
Ninja build failure
        ↑
A specific .cu/.cpp task FAILED
        ↑
The first fatal error / compiler error
        ↑
The True Root Cause
```

# 5. Recommended Complete Startup and Compilation Workflow

After completing the initial installation, if you need to recompile Detectron2 in the future, it is highly recommended to initialize the environment in a fixed sequence every time you open a new CMD:

DOS

```
conda activate fas

call "D:\app\MicrosoftC\VC\Auxiliary\Build\vcvarsall.bat" x64 -vcvars_ver=14.36

set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8"
set "PATH=%CUDA_HOME%\bin;%PATH%"
set "DISTUTILS_USE_SDK=1"

cd /d F:\owod\detectron2
```

Subsequently verify:

DOS

```
where cl
where nvcc

python -c "import torch; from torch.utils.cpp_extension import CUDA_HOME; print('PyTorch:', torch.__version__); print('Torch CUDA:', torch.version.cuda); print('CUDA_HOME:', repr(CUDA_HOME))"
```

If you only modify Detectron2's Python code, you do not need to reinstall or recompile. However, if you modify C++/CUDA code under `detectron2/layers/csrc`, you must execute:

DOS

```
rmdir /s /q build
python setup.py build_ext --inplace
```

For this specific environment, the final stable compilation dependency chain is:

Plaintext

```
RTX 4090
    ↓
NVIDIA Driver 546.65
    ↓
PyTorch 2.0.1 + cu118
    ↕ Version Alignment
CUDA Toolkit 11.8
    ↓
NVCC 11.8
    ↕ Compiler Compatibility
MSVC v143 14.36.32532
    ↓
Windows SDK 10.0.26100.0
    ↓
Detectron2 C++/CUDA Extension
    ↓
_C.cp39-win_amd64.pyd
```

The core principle behind this configuration is not "use the latest version for all components," but rather to establish a mutually compatible CUDA and MSVC build toolchain centered around the existing PyTorch version. For scenarios where you must preserve the existing software environment of a shared computer, you should prioritize the coexistence of multiple CUDA Toolkit versions, the coexistence of multiple MSVC toolsets, and local terminal environment variable configurations, rather than uninstalling or replacing the system's existing components.
