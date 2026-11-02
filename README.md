# 谱序乐鸣

**让琴键、触控、手势和歌声汇入同一台 FPGA 乐器。**

谱序乐鸣是一套基于高云 FPGA 的交互式电子乐器。FPGA 完成多声部物理建模、独立包络、效果处理与显示；RISC-V 软核管理 USB 输入；ESP32 提供手机网页；电脑视觉程序把双手动作转成带确认的演奏指令。声音最终从板端音频接口输出，手机也能预听导入曲谱。

## 获取源码

[下载完整源码压缩包](./谱序乐鸣_源码.zip)。解压到仅含英文字母、数字和下划线的目录，进入同时包含 Final7.gprj 与 README.md 的工程根，再按下文构建和配置。

## 功能与容量

| 模块 | 实现范围 |
| --- | --- |
| 合成与分配 | 7 个实际声部，2 个并行建模单元分别调度 4、3 组状态；25 键可配置映射 |
| 音色 | 3 类基础音色与自定义钢琴；上电默认三角钢琴；EP 分支在当前顶层关闭 |
| 表情与效果 | 7 路独立 ADSR、踏板、混响、合唱和颤音 |
| 录制与回放 | 最多 4096 个事件、最长 60 秒；保存与导出演奏数据 |
| 曲谱下载 | 最多 512 个按下/松开事件；网页校验后分包传输 |
| 手机演奏 | ESP32 热点、网页琴键、映射编辑、音色和参数控制 |
| 清唱转谱 | 最长 30 秒、最多 256 个音符；先识别和预听，再手动上传 |
| 双手手势 | 左手选起始音、右手选音符数量；握拳停止或触发小星星 |
| 显示与教学 | 波形、1024 点 FFT、音高轨迹、李萨如图、42 步 LED 跟弹 |

显示沿用 12 路界面布局，实际音频资源为 7 路。界面的槽位数量不能用于推断同时发声音符数。

音频同时送往两路独立的 16 位立体声串行输出，左右声道使用同一份合成 PCM：

| 输出 | 格式与引脚 |
| --- | --- |
| 板载 PT8211 耳机链路 | Japanese 格式；`hp_bck` / `hp_ws` / `hp_din` 对应 Y17 / AB17 / AA16 |
| 外接标准 I²S | WS 切换后延迟 1 个 BCK 发送 MSB；`i2s_bck` / `i2s_ws` / `i2s_sd` 对应 C17 / D17 / A19 |

两路发送器均已在当前七声部顶层接通。标准 I²S 使用 50 MHz 时钟和默认 `HALF_DIV=16`，BCK 为 1.5625 MHz，立体声帧率为 48,828.125 Hz；这组三线不另外输出 MCLK。外接 DAC 应核对 3.3 V 电平、供电与器件所需时钟；板载 PT8211 的三线时序与标准 I²S 不同。

## 系统分工

```text
USB 琴键 / 触控 ── USB Host + RISC-V ─┐
手机网页 ── ESP32 ── CH340 / USB ─────┤
双手摄像头 ── 电脑识别 ── 串口协议 ──┤
                                     ↓
                           输入仲裁 / 25 键映射
                                     ↓
                        7 声部建模 → ADSR → 效果
                                     ↓
                    板端音频 / HDMI 可视化 / LED 教学
```

FPGA 工程器件为 **GW5AST-LV138PG484AC1/I0**，顶层为 `hub_play_top`。主要时钟为输入 50 MHz、USB 48 MHz、像素 51.25 MHz 和 TMDS 256.25 MHz。引脚以 `src/play/pins.cst` 为准，时序以 `src/play/timing.sdc` 为准。

## 目录

```text
Final7.gprj             Gowin 源工程入口
Build_Final.ps1        固件、综合与完整 FPGA 构建入口
Build_ESP32.ps1        ESP32 编译入口
Start_Gesture_Host.ps1 双手手势程序入口
src/                   RTL、include、PLL、ROM 与字体资源
firmware/              RISC-V USB 固件、启动代码、链接脚本
implementation/        工程内相对布局参考
arduino/P8_WiFi_Web/    ESP32 草图、协议与嵌入网页
web/                   网页可编辑源文件与离线预览
gesture_host/          当前双手演奏程序、视觉模型、字体
host_ui/               协议兼容实现与离线测试支持
converter/             可选桌面音频转谱工具
examples/              曲谱格式示例
sim/                   HDL/C/C++ 测试与本地参考夹具
tools/                 构建、资源同步与回归工具
SOURCE_DEPENDENCIES.json 完整相对文件清单和工程设置
```

## 开始使用

### 1. 准备工具

| 用途 | 工具 |
| --- | --- |
| FPGA | Gowin EDA Education 1.9.11.03，须支持上述精确器件 |
| USB 软核固件 | xPack RISC-V GCC 13.2.0-2，`riscv-none-elf-*` 工具，RV32I/ILP32 |
| RTL 仿真 | ModelSim 10.7，命令行 `vlib`、`vlog`、`vsim`；原生器件测试另需 Gowin 仿真库 |
| 构建和资源生成 | Python 3.10 或更新，基础构建只使用标准库 |
| 网页协议测试 | Node.js 18 或更新，无需安装 npm 包 |
| 手势界面与视觉 | PySide6、pyserial、OpenCV、MediaPipe、NumPy，见 `requirements.txt` |
| ESP32 | Arduino CLI、`esp32:esp32` 核心 3.3.12，AI Thinker ESP32-CAM 配置 |
| 可选转谱 | NumPy、Tkinter、FFmpeg；高级转谱依赖见 `converter/requirements-optional.txt` |

构建工具从 PATH 或以下环境变量寻找安装位置。变量填本机安装目录；不要把个人安装路径写回源文件。

| 变量 | 含义 |
| --- | --- |
| `GOWIN_BIN` | 含 `gw_sh.exe` 的 Gowin `IDE/bin` 目录 |
| `RISCV_BIN` | 含 `riscv-none-elf-gcc.exe` 的工具目录 |
| `MODELSIM_BIN` | 含 `vlib.exe`、`vlog.exe`、`vsim.exe` 的目录 |
| `ARDUINO_CLI_BIN` | 含 `arduino-cli.exe` 的目录 |
| `ARDUINO_CONFIG` | 可选 Arduino CLI 配置文件 |
| `PUXU_PYTHON` | 可选构建 Python 路径 |
| `INSTRUMENT_UI_PYTHON` | 可选手势界面 Python 路径 |
| `INSTRUMENT_VISION_PYTHON` | 可选独立视觉 Python 路径；默认与界面相同 |

手势依赖可安装到自己的虚拟环境：

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
$env:INSTRUMENT_UI_PYTHON = (Resolve-Path .\.venv\Scripts\python.exe).Path
```

### 2. 构建 FPGA

在工程根目录执行：

```powershell
.\Build_Final.ps1 -Stage firmware  # 重新编译 USB 固件并更新三个 ROM HEX
.\Build_Final.ps1 -Stage syn       # 固件 + 综合
.\Build_Final.ps1 -Stage full      # 固件 + 综合 + 布局布线
```

综合和布局布线会打开独立只读日志窗口。关闭日志窗口不终止构建。工具日志和中间文件生成到 `build/`，Gowin 输出生成到 `impl/`；完整构建的目标码流为 `impl/pnr/Final7.fs`。

工程提供已生成的 RTL、PLL 源文件和三个固件 HEX，因此打开 `Final7.gprj` 能找到其源依赖。命令行入口会先重建固件，再使用工程内的相对布局参考。`.p` 是 Gowin 保护格式的布局输入，不是可下载码流；它不依赖原电脑的其他工程。`impl/Final7_process_config.json` 在构建时生成，包含顶层、include 与布局配置。

本仓库交付源码，未附带 FPGA 码流或既有 Programmer 结果索引。完整构建后请检查当前输出、器件、时序报告和引脚，再通过本机 Programmer 临时下载到 SRAM。烧写永久 Flash 应由使用者明确选择。

### 3. 配置和编译 ESP32

热点的默认设置是公开示例。先把 `arduino/P8_WiFi_Web/config.local.example.h` 复制为同目录的 `config.local.h`，填写自己的热点名称和至少 8 位密码。该本地文件已加入忽略清单。当前默认只开 AP，不需要路由器或电脑热点。

```powershell
arduino-cli core install esp32:esp32@3.3.12
.\Build_ESP32.ps1
```

先按 Espressif Arduino 核心的安装说明配置其软件包索引。需要专用 CLI 配置时设置 `ARDUINO_CONFIG`。编译目标为 `esp32:esp32:esp32cam`，本轮输出位于 `build/esp/<本次目录>/output/`。

编译入口不会连接串口。需要烧录时，按实际 USB 串口号执行，例如：

```powershell
arduino-cli upload --fqbn esp32:esp32:esp32cam --port <你的串口> --input-dir <本次output目录> arduino/P8_WiFi_Web
```

进入下载模式、复位与供电按实际 ESP32-CAM/CH340 载板说明操作。UART0 用于二进制协议；运行时把配套 CH340 接到 FPGA USB Host，避免其他程序同时占用该连接。

网页显示的示例热点说明不保存你的本地口令。手机连接你配置的热点后，打开 `http://192.168.4.1/`，保持无互联网连接，等待设备确认后演奏。

### 4. 修改网页

编辑 `web/` 内的 HTML/CSS/JS 后执行：

```powershell
python -B tools/build_wifi_web_assets.py
python -B tools/verify_web_assets.py
```

生成器同步更新 `web/preview.html`、资源清单和 `arduino/P8_WiFi_Web/web_assets.h`。修改网页后须重新编译和烧录 ESP32，设备才会使用新页面。直接打开 `web/preview.html` 可查看界面与本地曲谱预听；连接设备请使用 ESP32 的 HTTP 页面。

### 5. 双手手势演奏

运行 `Start_Gesture_Host.ps1` 或双击 `启动手势演奏.cmd`。选择电脑摄像头和板卡串口；串口使用 115200、8N1。程序收到匹配的身份、序号和校验应答后才显示已连接。

左手 1–7 选择起始音，右手 1–7 选择数量。例如左手 6、右手 4，按顺序演奏 6、7、1、2。保持同一手势只演奏一次，数字变化后重新触发。默认间隔 300 ms，可在界面调整为 100–1000 ms。

左拳立即停止当前手势音序、延音和回放。右拳单次触发小星星，保持握拳不会反复重启；松拳后继续播放，重新确认右手数字后允许下一次右拳触发。摄像头丢失会停止当前手势音序。板端应处于允许手势单音的状态，并关闭图案模式。

模型随项目提供于 `gesture_host/models/gesture_recognizer.task`。离线检查可用 `Start_Gesture_Host.ps1 -Check`；真实手势效果仍须在所用光照、摄像头和板卡上确认。

## 验证与仿真

无需板卡的网页和协议检查：

```powershell
node --test tools/test_wifi_web.js tools/test_score_download.js tools/test_performance_audio.js tools/test_transport_recovery.js tools/test_singing.js
node tools/test_poly25_keymap.js
node tools/test_singing_upload_snapshot.js
python -B tools/verify_web_assets.py
python -B tools/generate_poly_models.py --verify
```

设置 `MODELSIM_BIN` 后执行：

```powershell
python -B tools/test_final_wifi.py
python -B tools/test_final7_audio.py --tone mixed
python -B tools/final_run_sim.py tb_final_record_capacity src/final/final_recorder.sv sim/final_record/tb_final_record_capacity.sv
```

仿真结果写入 `build/sim/`。`test_final_wifi.py` 产生新 RTL 串口应答夹具后，`tools/test_dual_host.py` 可继续检查双手程序及协议兼容。仿真、编译与板上实际音频、摄像头识别属于不同检查步骤。

## 可选桌面音频转谱

运行 `python converter/app.py`，选择音频、预听并导出曲谱。基础流程使用单旋律检测；安装可选依赖后可使用 Basic Pitch。兼容导出最多 512 事件，桌面工具默认最多 4 个同时音符；这是其导出策略，不改变 FPGA 的 7 声部能力。复杂歌曲需先检查和修订识别结果。

## 第三方组件

USB Host 相关源文件保留原作者与许可证声明；Gowin PLL、器件原语及保护格式布局输入遵循其工具授权。随附字体的 SIL Open Font License 位于相应 `assets/fonts/OFL.txt`。手势模型来源入口保留于 `gesture_host/vision_worker.py`。本项目未为这些第三方组件重新指定许可证。

ModelSim 10.7 对中文工程路径的兼容性有限，执行该工具的仿真时请将源码解压到仅含英文字母、数字和下划线的目录。

