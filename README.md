# openocd-on-esp32 使用手册

本项目把 OpenOCD 移植到 ESP32-S3 上，使一块 ESP32-S3 开发板可以作为独立无线调试器使用。电脑连接 ESP32-S3 发出的 Wi-Fi 热点或同一局域网后，可以通过 TCP 连接 OpenOCD 的 GDB Server、Telnet 和 TCL 端口，对目标芯片进行调试。

仓库也展示了如何把 OpenOCD、JimTcl 这类较复杂的 C 项目集成到 ESP-IDF 工程中。

## 已验证环境

- 主机系统：Windows
- ESP-IDF：v5.5.4
- Python：3.11
- 调试器开发板：ESP32-S3-N16R8
- Flash / PSRAM：16 MB Flash + 8 MB Octal PSRAM
- 默认串口：`COM5`
- 默认热点：`esp-openocd`
- 默认 Web 地址：`http://192.168.4.1`

ESP32-S3-N16R8 的默认配置已经写入 `sdkconfig.defaults.esp32s3`：

```ini
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="16MB"
CONFIG_SPIRAM_MODE_OCT=y
```

其他 ESP32-S3 板卡需要按实际硬件调整 Flash 容量、Flash 模式和 PSRAM 模式。

## 硬件准备

调试器开发板必须是带 PSRAM 的 ESP32-S3。普通 ESP32 内存不足，不适合运行本应用。

如果使用 `ESP32-S3-WROOM-1U` 或板卡带 IPEX / U.FL 外接天线座，必须接好 2.4 GHz 天线。没有天线时串口日志可能显示 SoftAP 已启动，但手机和电脑完全搜不到热点。

调试器板和目标板必须共地。目标板需要独立供电，且已经烧录可运行固件。

## 快速开始

已安装 ESP-IDF 并且工程已经拉取完成时，常用流程如下：

```powershell
cd F:\openocd-on-esp32

Remove-Item Env:PYTHONPATH -ErrorAction SilentlyContinue
Remove-Item Env:PYTHONHOME -ErrorAction SilentlyContinue
Remove-Item Env:VIRTUAL_ENV -ErrorAction SilentlyContinue
Remove-Item Env:IDF_PYTHON_ENV_PATH -ErrorAction SilentlyContinue

$env:PYTHONNOUSERSITE = '1'
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
. F:\esp\esp-idf-v5.5.4\export.ps1

idf.py build
idf.py -p COM5 flash monitor
```

首次启动后，开发板会创建开放热点：

```text
SSID: esp-openocd
密码: 无
IP:   192.168.4.1
```

连接热点后，在浏览器打开：

```text
http://192.168.4.1
```

## 拉取源码

```powershell
git clone https://github.com/Warm-rain/openocd-on-esp32.git
cd openocd-on-esp32
git submodule update --init --recursive
```

`main/openocd` 子模块使用 Espressif 的 OpenOCD 上游仓库。如果旧版本 fork 的子模块 URL 仍指向不存在的 `../openocd-esp32.git`，可以手动修正：

```powershell
git config submodule.main/openocd.url https://github.com/espressif/openocd-esp32.git
git submodule update --init --recursive main/openocd
```

Windows 下如果 OpenOCD 子模块在生成 `startup_tcl.inc` 时失败，可以应用仓库中保留的补丁：

```powershell
git -C main/openocd apply ..\..\patches\openocd-esp32-windows-startup-tcl.patch
```

## 安装 ESP-IDF

以下路径是当前验证过的安装方式。你也可以换成自己的目录，但后续命令中的路径需要同步修改。

```powershell
mkdir F:\esp
git clone --branch v5.5.4 --recursive --depth 1 --shallow-submodules https://github.com/espressif/esp-idf.git F:\esp\esp-idf-v5.5.4

$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
powershell -ExecutionPolicy Bypass -File F:\esp\esp-idf-v5.5.4\install.ps1 esp32s3
```

如果当前 PowerShell 被 PlatformIO 虚拟环境污染，`export.ps1` 可能会检查到错误的 Python 包版本。进入 ESP-IDF 环境前建议清掉这些环境变量：

```powershell
Remove-Item Env:PYTHONPATH -ErrorAction SilentlyContinue
Remove-Item Env:PYTHONHOME -ErrorAction SilentlyContinue
Remove-Item Env:VIRTUAL_ENV -ErrorAction SilentlyContinue
Remove-Item Env:IDF_PYTHON_ENV_PATH -ErrorAction SilentlyContinue
$env:PYTHONNOUSERSITE = '1'
```

Windows 构建 JimTcl 需要 GNU Make，可以用 winget 安装：

```powershell
winget install --id ezwinports.make --source winget --accept-source-agreements --accept-package-agreements
```

如果系统找不到 `make.exe`，把 winget 安装目录加入 PATH：

```powershell
$env:PATH = 'C:\Users\Windows\AppData\Local\Microsoft\WinGet\Packages\ezwinports.make_Microsoft.Winget.Source_8wekyb3d8bbwe\bin;' + $env:PATH
```

每次打开新终端后，加载 ESP-IDF 环境：

```powershell
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
. F:\esp\esp-idf-v5.5.4\export.ps1
```

## 构建工程

首次构建建议先确认目标芯片：

```powershell
idf.py set-target esp32s3
idf.py build
```

构建成功后会生成：

- `build\openocd-on-esp32.bin`
- `build\bootloader\bootloader.bin`
- `build\partition_table\partition-table.bin`
- `build\storage.bin`

默认分区表：

| 分区 | 地址 | 大小 | 用途 |
| --- | --- | --- | --- |
| `nvs` | `0x9000` | `0x6000` | 保存 Wi-Fi 和 Web 配置 |
| `phy_init` | `0xf000` | `0x1000` | RF 校准数据 |
| `factory` | `0x10000` | `5M` | 主应用 |
| `storage` | `0x510000` | `528K` | FATFS，保存 OpenOCD 配置文件 |

## 烧录和串口监视

ESP32-S3-N16R8 枚举为 `COM5` 时：

```powershell
idf.py -p COM5 flash monitor
```

通用写法：

```powershell
idf.py -p COMx flash monitor
```

只更新应用时：

```powershell
idf.py -p COM5 app-flash monitor
```

如果要清掉已经保存的 Wi-Fi 配置和 Web 配置，先擦除整片 Flash：

```powershell
idf.py -p COM5 erase-flash
idf.py -p COM5 flash monitor
```

手动使用 esptool 烧录：

```powershell
python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size 16MB --flash_freq 80m 0x0 build\bootloader\bootloader.bin 0x8000 build\partition_table\partition-table.bin 0x10000 build\openocd-on-esp32.bin 0x510000 build\storage.bin
```

## 启动日志

正常启动时，串口会看到类似内容：

```text
I (...) boot.esp32s3: SPI Flash Size : 16MB
I (...) esp_psram: Found 8MB PSRAM device
I (...) esp_psram: SPI SRAM memory test OK
I (...) main: app mode (Access Point)
I (...) network-mngr: SoftAP started: ssid="esp-openocd", channel=1, auth=open, tx_power=20.00 dBm, ip=192.168.4.1
I (...) esp_netif_lwip: DHCP server started on interface WIFI_AP_DEF with IP: 192.168.4.1
Open On-Chip Debugger 0.12.0
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Listening on port 3333 for gdb connections
```

首次启动时还没有保存 NVS 配置，出现下面这类日志是正常的：

```text
ESP_ERR_NVS_NOT_FOUND
Default values read from menuconfig. AP mode will be activated!
```

## 连接热点和 Web 页面

首次启动默认进入 AP 模式：

```text
SSID: esp-openocd
密码: 无
频段: 2.4 GHz
信道: 1
IP:   192.168.4.1
```

电脑连接热点后，正常会拿到类似地址：

```text
IPv4:    192.168.4.2
Gateway: 192.168.4.1
```

可以用下面命令验证：

```powershell
ping 192.168.4.1
Invoke-WebRequest -Uri http://192.168.4.1/ -UseBasicParsing
```

浏览器打开：

```text
http://192.168.4.1
```

Web 页面可以配置：

- Wi-Fi SSID 和密码
- OpenOCD 目标配置文件
- JTAG / SWD 接口
- RTOS 类型
- 目标 Flash 支持
- OpenOCD 额外命令行参数

保存 Wi-Fi 后，下一次启动会进入 STA 模式，连接你配置的路由器，而不是继续创建 `esp-openocd` 热点。需要恢复默认热点时执行：

```powershell
idf.py -p COM5 erase-flash
idf.py -p COM5 flash monitor
```

## JTAG 接线

默认 JTAG 引脚：

| ESP32-S3 调试器引脚 | 功能 | 目标板示例引脚 |
| --- | --- | --- |
| GPIO38 | TCK | GPIO13 |
| GPIO39 | TMS | GPIO14 |
| GPIO40 | TDI | GPIO12 |
| GPIO41 | TDO | GPIO15 |
| GND | GND | GND |

运行时可以通过 OpenOCD 命令修改 JTAG 引脚：

```tcl
esp_gpio_jtag_nums 38 39 40 41
```

默认 JTAG 配置文件位于：

```text
main/openocd/tcl/interface/esp_gpio_jtag.cfg
```

如果要用 LED 显示 JTAG 活动，可以在接口配置中设置：

```tcl
esp_gpio_blink_num <led_pin_num>
```

## SWD 接线

默认 SWD 引脚：

| ESP32-S3 调试器引脚 | 目标板功能 |
| --- | --- |
| GPIO38 | SWCLK |
| GPIO39 | SWDIO |
| GND | GND |

默认 SWD 配置文件位于：

```text
main/openocd/tcl/interface/esp_gpio_swd.cfg
```

## 使用 GDB 调试目标板

OpenOCD 启动后会监听：

| 端口 | 用途 |
| --- | --- |
| `3333` | GDB Server |
| `4444` | Telnet 控制台 |
| `6666` | TCL 端口 |

如果电脑连接的是 `esp-openocd` 热点，调试器 IP 通常是：

```text
192.168.4.1
```

如果 ESP32-S3 调试器连接到路由器，请以串口 monitor 或路由器后台显示的 IP 为准。

ESP32 目标示例：

```powershell
xtensa-esp32-elf-gdb -ex "set remotetimeout 30" -ex "target extended-remote 192.168.4.1:3333" build\blink.elf
```

ESP32-S3 目标示例：

```powershell
xtensa-esp32s3-elf-gdb -ex "set remotetimeout 30" -ex "target extended-remote 192.168.4.1:3333" build\blink.elf
```

RISC-V ESP32 目标请使用对应工具链的 `riscv32-esp-elf-gdb`。

## 通过 JTAG 烧录目标应用

如果 ESP-IDF 工具目录中能找到 `openocd-esp32`，CMake 会生成两个额外目标：

```powershell
cmake --build build -t jtag-flasher-full
cmake --build build -t jtag-flasher
```

`jtag-flasher-full` 会烧录应用和 FATFS 文件系统；`jtag-flasher` 只烧录应用，默认应用地址为 `0x10000`。

默认 OpenOCD 配置文件为：

```text
board/esp32s3-builtin.cfg
```

## 修改默认配置

进入配置菜单：

```powershell
idf.py menuconfig
```

常用菜单：

- `OpenOCD-on-ESP32 Configuration`
- `OpenOCD target file`
- `OpenOCD interface`
- `OpenOCD target flash size`
- `OpenOCD rtos type`
- `WiFi SSID`
- `WiFi Password`
- `Serial flasher config`
- `Component config -> ESP PSRAM`

ESP32-S3-N16R8 建议保持：

```ini
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_SPIRAM_MODE_OCT=y
```

## ESP-BOX UI

项目支持在 ESP-BOX 类开发板上启用触摸屏 UI 和配网功能。默认关闭，需要在 `idf.py menuconfig` 中启用：

```text
OpenOCD-on-ESP32 Configuration -> Run on ESP-BOX
```

普通 ESP32-S3-N16R8 开发板不需要启用该选项。

## 常见问题

### `idf.py` 不存在

当前终端没有加载 ESP-IDF 环境。执行：

```powershell
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
. F:\esp\esp-idf-v5.5.4\export.ps1
```

### ESP-IDF 安装时报 `Windows-` 平台不支持

当前 Python 环境可能没有正确识别 CPU 架构。安装或导出环境前设置：

```powershell
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
```

### Python 依赖检查读到了 PlatformIO 的包

现象示例：

```text
Requirement 'click<8.2,>=7.0' was not met. Installed version: 8.3.2
Requirement 'pyparsing<3.3,>=3.1.0' was not met. Installed version: 3.3.2
```

处理方式：

```powershell
Remove-Item Env:PYTHONPATH -ErrorAction SilentlyContinue
Remove-Item Env:PYTHONHOME -ErrorAction SilentlyContinue
Remove-Item Env:VIRTUAL_ENV -ErrorAction SilentlyContinue
Remove-Item Env:IDF_PYTHON_ENV_PATH -ErrorAction SilentlyContinue
$env:PYTHONNOUSERSITE = '1'
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
. F:\esp\esp-idf-v5.5.4\export.ps1
```

### Windows 下 JimTcl 或 OpenOCD 构建失败

确认已安装 Git for Windows 和 GNU Make：

```powershell
winget install --id ezwinports.make --source winget
```

如果 OpenOCD 子模块在 Windows 下生成 `startup_tcl.inc` 失败，应用补丁：

```powershell
git -C main/openocd apply ..\..\patches\openocd-esp32-windows-startup-tcl.patch
```

### ESP32-S3-N16R8 启动时报 PSRAM 错误

错误示例：

```text
PSRAM chip is not connected
wrong PSRAM line mode
Failed to init external RAM
```

ESP32-S3-N16R8 应使用 Octal PSRAM。确认配置中包含：

```ini
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="16MB"
CONFIG_SPIRAM_MODE_OCT=y
```

修改后重新构建烧录：

```powershell
idf.py build
idf.py -p COM5 flash monitor
```

### 串口被占用

如果烧录时报 `Could not open COM5`，通常是串口工具或 monitor 还占着端口。关闭 SSCOM、串口助手、`idf.py monitor` 或其他上位机程序后重试。

可以查看串口：

```powershell
[System.IO.Ports.SerialPort]::getportnames()
mode COM5
```

### 搜不到 `esp-openocd` 热点

先看串口是否已经启动 AP：

```text
SoftAP started: ssid="esp-openocd", channel=1, auth=open, tx_power=20.00 dBm, ip=192.168.4.1
```

如果这行没有出现，说明 Wi-Fi 初始化未完成，需要看前面的错误日志。

如果这行已经出现但仍搜不到：

- 确认手机或电脑正在扫描 2.4 GHz Wi-Fi。
- ESP32-S3 不支持 5 GHz。
- 确认外接天线已经接好，尤其是 `ESP32-S3-WROOM-1U`。
- 让手机或电脑靠近开发板重新扫描。
- 换一台手机或电脑交叉验证。

Windows 可以用命令扫描：

```powershell
netsh wlan show networks mode=bssid
```

正常应看到：

```text
SSID: esp-openocd
BSSID: fc:01:2c:d1:fa:e5
Band: 2.4 GHz
Channel: 1
Authentication: Open
```

### OpenOCD 报 JTAG 全 0

错误示例：

```text
Error: JTAG scan chain interrogation failed: all zeroes
Error: esp32.cpu0: IR capture error; saw 0x00 not 0x01
```

这说明 OpenOCD 已经运行，但没有正确读到目标芯片 JTAG 链路。优先检查：

- 调试器板和目标板是否共地。
- 目标板是否上电。
- TCK、TMS、TDI、TDO 是否接反。
- 目标芯片 JTAG 引脚是否被固件占用。
- OpenOCD 里的 JTAG 引脚配置是否和实际接线一致。
- 线太长时先降低 JTAG 速度或缩短线。

### Web 页面打不开

先确认电脑已经连接 `esp-openocd` 热点，并且拿到 `192.168.4.x` 地址：

```powershell
ipconfig
ping 192.168.4.1
```

如果 `ping` 正常但浏览器打不开，尝试：

```powershell
Invoke-WebRequest -Uri http://192.168.4.1/ -UseBasicParsing
```

## 维护建议

提交前建议至少跑一次：

```powershell
idf.py build
```

父仓库提交只会记录父仓库文件。`main/openocd` 是 Git 子模块，如果直接修改子模块内容，需要单独提交到子模块对应仓库，或者像当前 Windows 构建修复一样把补丁文件保存在父仓库中。

## 许可证

本仓库中 ESP-IDF 应用部分版权归 Espressif Systems (Shanghai) Co. Ltd. 所有，采用 Apache 2.0 许可证，详见 [LICENSE](LICENSE)。

OpenOCD 子模块遵循 GPL v2.0 或更新版本许可证。
