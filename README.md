# Multilingual-Subtitle-Generation-Tool
# wsub

基于Whisper用纯 C 实现的多语言字幕生成器：把任意音频或视频转成 `.srt` 字幕，**完全在本地运行**，不上传任何数据。
包含一个命令行程序 `wsub.exe` 和一个 Win32 图形界面 `wsub_gui.exe`。

> A small, dependency-light subtitle generator written in C. It decodes audio with ffmpeg, transcribes it locally with [whisper.cpp](https://github.com/ggml-org/whisper.cpp), and writes SRT files. Includes a native Win32 GUI. Windows only for now.

## 特性

- 支持 ffmpeg 能解码的任何音视频格式，输出标准 SRT 字幕
- 本地离线识别，基于 whisper.cpp
- 可选 **Silero VAD**：跳过静音和音乐，明显减少"无声处乱出字幕"和"结尾句子反复重复"
- 语言自动检测，也可手动指定
- 默认束搜索（beam search，宽度 5），比贪心解码更准
- 兜底去重：连续出现完全相同的字幕时只保留一条
- 图形界面：拖拽文件、进度条、一键取消、自动保存上次设置
- 纯 C，界面只用 Win32 API，没有第三方界面库

## 工作原理

```
音视频 ──ffmpeg（管道，16kHz 单声道 float32）──▶ 内存中的 PCM
                                                  │
                                    whisper.cpp（可选 VAD）
                                                  ▼
                                  带时间戳的分段 ──▶ .srt

wsub_gui.exe ──CreateProcess + 匿名管道──▶ wsub.exe（识别在独立进程里，界面不会卡死）
```

## 环境要求（Windows 10 / 11）

- [MinGW-W64](https://www.mingw-w64.org/)：需要 `gcc`、`windres`、`mingw32-make`
- [CMake](https://cmake.org/download/)
- [Git](https://git-scm.com/download/win)
- [ffmpeg](https://ffmpeg.org/download.html)，并确保命令行里 `ffmpeg -version` 能运行（本项目不打包 ffmpeg）

## 编译

**1. 克隆**

```powershell
git clone --recursive https://github.com/COM10101/Multilingual-Subtitle-Generation-Tool.git
cd Multilingual-Subtitle-Generation-Tool
```

**2. 编译 wsub**

```powershell
.\build.ps1
```

PowerShell 提示"禁止运行脚本"时：`powershell -ExecutionPolicy Bypass -File .\build.ps1`。

成功后在 `dist\` 下得到 `wsub.exe` 和 `wsub_gui.exe`，两个文件放在同一个文件夹里使用。

## 下载模型

模型自行下载，放在任意位置：

```powershell
# 识别模型：small 较快，large-v3-turbo 量化版更准、也更慢，可根据需要选择其他模型
Invoke-WebRequest https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin -OutFile ggml-small.bin
Invoke-WebRequest https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo-q5_0.bin -OutFile ggml-large-v3-turbo-q5_0.bin
...

# VAD 模型（很小，强烈建议下载）
Invoke-WebRequest https://huggingface.co/ggml-org/whisper-vad/resolve/main/ggml-silero-v5.1.2.bin -OutFile ggml-silero-v5.1.2.bin
```

## 使用

### 图形界面

运行 `dist\wsub_gui.exe`，选择（或拖入）音视频文件和模型，点"开始识别"。字幕保存在输入文件旁边，同名 `.srt`。设置会保存在 `wsub_gui.ini` 中。

### 命令行

```powershell
.\wsub.exe <输入音视频> <模型.bin> [选项]
```

| 选项 | 说明 | 默认 |
|---|---|---|
| `-l <语言>` | `auto` / `zh` / `en` / `ja` … | `auto` |
| `-o <输出.srt>` | 输出路径 | 输入文件同名 `.srt` |
| `--vad <模型.bin>` | 启用语音活动检测 | 关闭 |
| `--beam <N>` | 束搜索宽度，`0` 为贪心（更快但更容易出错） | `5` |
| `-t <N>` | CPU 线程数 | `4` |

示例：

```powershell
.\wsub.exe talk.mp4 ggml-large-v3-turbo-q5_0.bin --vad ggml-silero-v5.1.2.bin -l en
```

## 使用建议

- **语言不确定就用 `auto`。** 强制指定错误的语言会让模型编出不相干的文字。
- **有片头片尾、背景音乐或长时间停顿的视频，务必开 VAD。**
- 追求准确率：用 `large-v3-turbo` 量化版或更大的模型；追求速度：用 `small` 或 `base`。
- 控制台中文输出乱码时先执行 `chcp 65001`。

## GPU 加速（实验性）

默认编译出的是纯 CPU 版本。whisper.cpp 支持 CUDA、Vulkan 等后端，这是**编译 whisper.cpp 时决定的**，不是运行时开关。

在 Windows 上，CUDA 的 `nvcc` 需要搭配 MSVC，不能搭配 MinGW：

1. 安装 Visual Studio Build Tools 和 CUDA Toolkit；
2. 在 *x64 Native Tools Command Prompt* 里用 `-DGGML_CUDA=ON -DBUILD_SHARED_LIBS=ON` 编译 whisper.cpp；
3. 用 `cl` 编译 `src\wsub.c` 并链接得到的 `whisper.lib`，把相关 DLL 放在 `wsub.exe` 旁边；
4. 直接用新的 `wsub.exe` 即可。

运行时日志里出现 `ggml_cuda_init: found 1 CUDA devices` 即说明用上了 GPU。这部分尚未在本仓库中完整验证，欢迎反馈。

## 已知限制

- 传给 `wsub.exe` 的路径会经过系统 ANSI 代码页，含生僻字符的路径可能失败，建议使用英文路径。
- 目前只输出 SRT。
- 仅支持 Windows。
- 源码为 UTF-8（无 BOM）。使用 MSVC 编译时需要加 `/utf-8`。

## 计划

- [ ] 批量处理多个文件
- [ ] 输出 VTT / ASS
- [ ] 翻译为英文（`translate` 选项）
- [ ] 路径改用宽字符接口，彻底支持中文路径

## 项目结构

```
src/wsub.c            命令行程序：ffmpeg 解码 + whisper.cpp 识别 + 写 SRT
gui/wsub_gui.c        Win32 图形界面
gui/wsub_gui.rc       资源脚本（嵌入 manifest）
gui/wsub_gui.manifest 启用现代控件外观与 DPI 感知
build.ps1             一键编译脚本
third_party/          whisper.cpp
```

## 致谢与许可

本项目使用 [MIT 许可证](LICENSE)。

- [whisper.cpp](https://github.com/ggml-org/whisper.cpp)（MIT）
- [OpenAI Whisper](https://github.com/openai/whisper)（MIT）
- [Silero VAD](https://github.com/snakers4/silero-vad)（MIT）
- [FFmpeg](https://ffmpeg.org/)：本项目不分发，请自行安装，使用时请遵守其许可证
