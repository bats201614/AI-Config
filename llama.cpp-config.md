# llama.cpp 部署与配置

## 1. 下载 llama.cpp
- 从 GitHub 官方仓库下载 llama.cpp 源码：{https://github.com/ggerganov/llama.cpp}
  - 下载方式一：使用 `git clone` 克隆仓库（适合想获得极致的性能，或者使用特定的加速技术（如 Vulkan、OpenCL），需自行编译）：
    ```bash
    git clone https://github.com/ggerganov/llama.cpp.git
    ```
  - 下载方式二：下载预编译版本，在 llama.cpp GitHub 的 Releases 页面找到 Assets 列表，根据你的操作系统选择对应的压缩包：
    - Windows (Nvidia 显卡): 选择带有 `cuda` 字样的文件，例如 `llama-bXXXX-bin-win-cuda-cu12.4-x64.zip`（需匹配你的 CUDA 版本）
    - Windows (无显卡/仅 CPU): 选择 `llama-bXXXX-bin-win-avx2-x64.zip`
    - macOS: 苹果芯片 (M1/M2/M3) 选择 `llama-bXXXX-bin-macos-arm64.zip`
    - 下载后解压，即可看到 `llama-cli`（命令行对话）或 `llama-server`（API 服务）等可执行文件
  - 下载方式三：使用包管理器：
    ```bash
    # macOS / Linux
    brew install llama.cpp

    # Windows
    winget install llama.cpp
    ```

## 2. 安装编译依赖
- llama.cpp 主要使用 CMake 进行构建，需要以下依赖：

- **Windows (拥有 Nvidia 显卡)**：
  - Git：[git-scm](https://git-scm.com/download/win)
  - CMake (根据显卡选择版本)：[cmake](https://cmake.org/download/)
  - CUDA Toolkit (根据显卡选择版本)：[CUDA Toolkit](https://developer.nvidia.com/cuda-downloads)
  - C++ 编译器：推荐安装 [Visual Studio 2022 社区版](https://visualstudio.microsoft.com/downloads/)，并勾选 "使用 C++ 的桌面开发"

- **Windows (仅 CPU)**：
  - Git：[git-scm](https://git-scm.com/download/win)
  - CMake (3.14 或更高版本)：[cmake](https://cmake.org/download/)
  - C++ 编译器：推荐安装 [Visual Studio 2022 社区版](https://visualstudio.microsoft.com/downloads/)，并勾选 "使用 C++ 的桌面开发"

- **Ubuntu/Debian**：
  ```bash
  sudo apt update
  sudo apt install -y build-essential cmake git
  ```

- **CentOS/RHEL**：
  ```bash
  sudo yum groupinstall -y "Development Tools"
  sudo yum install -y cmake git
  ```

- **macOS**：
  ```bash
  xcode-select --install
  brew install cmake git
  ```

## 3. 编译 llama.cpp
- 进入 llama.cpp 目录
  ```bash
  cd llama.cpp
  ```

- 使用 CMake 配置构建：
- Windows 命令：
  ```PowerShell
  cmake -B build -DGGML_CUDA=ON
  # 针对 NVIDIA 显卡，需要指定 GGML_CUDA=ON
  ```
- Linux 命令：
  ```bash
  cmake -B build -DGGML_CUDA=ON
  # 针对 NVIDIA 显卡，需要指定 GGML_CUDA=ON
  ```
- 编译（可根据 CPU 核心数调整 `-j` 参数）：
  ```PowerShell
  cmake --build build --config Release -j
  ```

- 编译完成后，可执行文件位于 `build/bin/` 目录下（进入 `build/bin/Release` (Windows) 或 `build/bin` (Linux) 目录：）：
  ```bash
  # Linux
  ls build/bin/
  # Windows
  cd build\bin\Release
  ```
- 编译检查：
- ```powershell
  ./llama-cli --version
  ```
  如果看到输出中包含 CUDA 字样，说明编译成功。
- 额外说明：如果需要重新编译命令，需运行`if (Test-Path build) { rm -r -force build }`来删掉之前创建过的 build 文件夹，清理之前的缓存，否则它找不到新装的 CUDA

## 4. 下载模型
- llama.cpp 支持 GGUF 格式的模型，可从以下来源获取：
  - Hugging Face：[GGUF 模型列表](https://huggingface.co/models?other=gguf)
  - The Bloke's GGUF 量化模型：[TheBloke](https://huggingface.co/TheBloke)
- 例如下载量化模型：
  ```bash
  # 创建模型目录
  mkdir -p models

  # 使用 wget 下载（以 llama-2-7b-chat.Q4_0.gguf 为例，也可以手动下载解压）
  wget https://huggingface.co/TheBloke/Llama-2-7B-Chat-GGUF/resolve/main/llama-2-7b-chat.Q4_0.gguf -O models/
  ```
- 根据显存大小选择模型，粗略原则最好模型不要超过显存大小的一半，MOE模型可以适当放宽限制
- 量化保证通用性选择`K-Quants`压缩如`Q{N}_K_M`，如显卡较新可选择`Microscaling Formats (MX)`压缩如`MXFP{N}`，可以在以往省显存的基础上翻倍提速，并且由于使用了浮点型量化精度更高

## 5. 运行 llama.cpp
- 使用 `llama-cli` 运行模型：
  ```bash
  # 示例
  ./build/bin/llama-cli \
    -m 你的模型.gguf \
    -n 512 \
    -fa \
    -c 8192 \
    -b 1024
    -ngl 99\
    -t 8
    -np 1
    -p "### User: Hello, who are you?\n### Assistant:"
  ```

- 常用参数说明：
  - `-m`：模型文件路径
  - `-n`：生成的最大 token 数量
  - `-fa`：闪烁注意力机制（需显卡支持，开启后减少显存占用并提升生成速度）
  - `-c`：上下文长度
  - `-b`：批处理大小，影响模型理解你输入问题的速度
  - `-t`：线程数（默认 CPU 核心数）
  - `-ngl`（重要）：GPU 加速的层数（需编译时启用 CUDA/Metal 等）
  - `-np`：并行处理数（建议个人本地使用设为 `1` 即可）
  - `-p`：输入提示词
- 如果想看完整的参数列表，可以在终端输入：
  ```PowerShell
  .\llama-cli.exe -h
  # 或使用server
  .\llama-server.exe -h
  ```

## 6. 使用 llama-server（可选）
- llama.cpp 提供 HTTP 服务器功能，方便通过 API 调用：
  ```bash
  # 示例
  ./build/bin/llama-server \
    -m models/llama-2-7b-chat.Q4_0.gguf \
    -ngl 40 \
    -c 8192 \
    --host 0.0.0.0 \
    --port 8080
  ```

- 访问 `http://localhost:8080` 查看 Web UI

## 7. 验证
- 检查编译产物：
  ```bash
  ls -la build/bin/llama-cli
  ls -la build/bin/llama-server
  ```

- 运行简单测试：
  ```bash
  ./build/bin/llama-cli -m models/llama-2-7b-chat.Q4_0.gguf -p "Hi" -n 10
  ```

- 查看支持的模型格式：
  ```bash
  ./build/bin/llama-cli --help
  ```

## 额外说明：CUDA Toolkit 与 Visual Studio 下载安装详解

## CUDA Toolkit
### 查看显卡支持的 CUDA 版本
- 打开 NVIDIA 控制面板（右键桌面 → NVIDIA 控制面板）
- 点击右下角 "系统信息"
- 在 "组件" 中查看 "驱动程序版本" 后面显示的数字版本，例如 `581.80`
- 记下这个版本号，下载的 CUDA Toolkit 版本不能超过该版本，可以去对应的CUDA版本文档中查看兼容性

### 下载 CUDA Toolkit
- 访问 [NVIDIA CUDA 下载页面](https://developer.nvidia.com/cuda-downloads)
- 选择你的操作系统：
  - **Windows**：选择 `Windows → x86_64 → 11（选择Windows版本，11或10） → exe(local)`
- 运行下载的安装程序，安装选项建议：
  - 选择 “自定义（高级）”
  - 勾选全部 "CUDA"（核心组件）
  - 勾选 "CUDA Documentation"（如有的话可选）
  - 取消勾选 "CUDA Visual Studio Integration"（如果已单独安装 VS）
  - 安装位置建议使用默认路径
- 安装完成后，验证：
  ```bash
  nvcc --version
  ```
- 设置环境变量（`安装程序通常自动设置，若未设置手动添加`）：
  ```powershell
  # 在系统环境变量中添加
  CUDA_PATH = C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4
  # 将以下路径添加到 PATH
  C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4\bin
  C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4\libnvvp
  ```

### cuDNN 下载安装（可选，用于加速）
- cuDNN 是 CUDA 的深度神经网络加速库，llama.cpp 可选使用
- 访问 [NVIDIA cuDNN 下载页面](https://developer.nvidia.com/cudnn)
- 需要登录 NVIDIA 账号并填写问卷后下载
- 选择与 CUDA 版本匹配的 cuDNN 版本：
  - 下载 `cudnn-windows-x86_64-cudaXX-YY.zip`（XX 为 CUDA 版本，YY 为 cuDNN 版本）
- 解压后将以下文件复制到 CUDA Toolkit 目录：
  - `bin/cudnn*.dll` → `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4\bin\`
  - `include/cudnn*.h` → `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4\include\`
  - `lib/cudnn*.lib` → `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4\lib\`

### 常见问题
- **编译时找不到 CUDA**：确保 CUDA Toolkit 已正确安装且环境变量已设置，重启终端或编辑器
- **CUDA 版本不兼容**：确保编译时指定的 `GGML_CUDA=ON` 与显卡驱动支持的 CUDA 版本匹配
- **显存不足**：减小模型量化等级（如使用 Q2_K 而非 Q4_0），或减少 `-ngl` 值

## Visual Studio 下载安装

### 下载 Visual Studio
- 访问 [Visual Studio 下载页面](https://visualstudio.microsoft.com/downloads/)
- 下载 **Community（社区版）**（免费版本，个人开发者使用）
- 运行安装程序

### 安装选项
- 路径选项：
  - 安装路径分为三个部分，核心产品 (第一行)是 VS 的主程序，可以更改路径；下载缓存(第二行)是安装包的临时存放地，可以更改路径避免空间占用，共享组件、工具和 SDK (第三行)不建议更改（或无法更改）路径，核心的 C++ 编译器和 Windows SDK 必须安装在系统默认路径才能让 CUDA 或 CMake 找到
  - 建议取消勾选“安装后保留下载缓存”，这个缓存只是 VS 的安装包镜像，除非打算以后在没网的情况下“修改”或“修复” VS，否则留着它只会占用好几个 GB 的空间。
- 首次运行时，安装程序会要求选择工作负载，勾选：
  - **"使用 C++ 的桌面开发"**（必须）
  - （可选）".NET 桌面开发"
  - （可选）"Python 开发"
- 右侧 "安装详细信息" 中确保勾选：
  - MSVC v143 - VS 版本 C++ x64/x86 生成工具
  - Windows 11 SDK（或 Windows 10 SDK，视系统而定）
- 安装位置建议使用默认路径

### 验证安装
- 打开 Visual Studio Installer，确认 Visual Studio 显示 "已安装"
- 打开 Visual Studio ，创建一个空项目，尝试编译一个简单的 C++ 程序验证工具链正常

### 额外说明
- 如果已经安装过 Visual Studio但没有勾选 C++ 开发组件，可以打开 Visual Studio Installer，点击 "修改"，勾选 "使用 C++ 的桌面开发" 即可
- CMake 编译 llama.cpp 时会自动检测 Visual Studio，无需额外配置
- 如果无法检测（如CUDA版本较新时，Visual Studio没能识别），可以在编译时指定路径，完整命令如：
  ```powershell
  # 先彻底删除旧的 build 文件夹
    if (Test-Path build) { rm -r -force build }
  # （可选）手动指定-G 参数明确说明用 VS 2026，并通过 -A x64 指定 64 位架构、-DCMAKE_CUDA_ARCHITECTURES架构写死（100）
  #（必选）指定 -DCMAKE_CUDA_COMPILER 定义 CUDA 编译器路径并再次尝试
  # 如依然报错，配置-DCMAKE_CUDA_FLAGS 绕过版本检查限制
  cmake -B build -G "Visual Studio 18 2026" -A x64 `
  -DGGML_CUDA=ON `
  -DCMAKE_CUDA_ARCHITECTURES=100 `
  -DCMAKE_CUDA_FLAGS="-allow-unsupported-compiler" `
  -DCMAKE_CUDA_COMPILER="C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.0/bin/nvcc.exe"
  ```
