# ESP32-S3 上运行的 OpenOCD 调试器

本项目把 OpenOCD 移植到 ESP32-S3 上，使一块 ESP32-S3 开发板可以作为独立调试器使用，并通过 Wi-Fi 或以太网向目标芯片暴露 GDB Server。

仓库也展示了如何把 OpenOCD 这类较复杂的 C 项目集成到 ESP-IDF 工程中。

## 硬件要求

- 调试器开发板必须是 ESP32-S3。普通 ESP32 内存不足，不适合运行本应用。
- ESP32-S3 开发板必须带 PSRAM。
- ESP32-S3-N16R8 已验证可用：16 MB Flash + 8 MB Octal PSRAM，本仓库的 `sdkconfig.defaults.esp32s3` 已按该规格设置。
- 其他板卡需要在 `menuconfig` 中按实际规格调整 Flash 与 PSRAM 的 SPI 模式，例如 DIO、QIO 或 OPI。
- 被调试的目标板需要已经烧录可运行的应用。
- 调试器板和目标板需要共地。

## JTAG 连接

默认 JTAG 引脚如下：

| ESP32-S3 调试器引脚 | 功能 | 目标板引脚示例 |
| --- | --- | --- |
| GPIO38 | TCK | GPIO13 |
| GPIO39 | TMS | GPIO14 |
| GPIO40 | TDI | GPIO12 |
| GPIO41 | TDO | GPIO15 |

运行时可以通过 OpenOCD 命令修改 JTAG 引脚：

```tcl
esp_gpio_jtag_nums 38 39 40 41
```

默认引脚配置位于 OpenOCD 子模块中的 `interface/esp_gpio_jtag.cfg`。

如需显示 JTAG 收发活动，可以外接 LED，并在 `interface/esp_gpio_jtag.cfg` 中使用：

```tcl
esp_gpio_blink_num <led_pin_num>
```

## SWD 连接

项目也支持 SWD，默认连接如下：

| ESP32-S3 调试器引脚 | 目标板功能 |
| --- | --- |
| GPIO38 | SWCLK |
| GPIO39 | SWDIO |

SWD 默认配置文件为 OpenOCD 子模块中的 `interface/esp_gpio_swd.cfg`。

## 软件环境

推荐使用 ESP-IDF 5.x。当前工程已在以下环境验证通过：

- Windows
- ESP-IDF v5.5.4
- Python 3.11
- Git for Windows
- GNU Make

项目声明的最低 ESP-IDF 版本为 5.0，但新环境建议优先使用稳定的 5.x 版本。

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

如果 Windows 下 OpenOCD 子模块在生成 `startup_tcl.inc` 时失败，可以应用本仓库保留的 Windows 构建补丁：

```powershell
git -C main/openocd apply ..\..\patches\openocd-esp32-windows-startup-tcl.patch
```

## Windows 环境准备

安装 ESP-IDF v5.5.4 的一个可复现示例：

```powershell
mkdir F:\esp
git clone --branch v5.5.4 --recursive --depth 1 --shallow-submodules https://github.com/espressif/esp-idf.git F:\esp\esp-idf-v5.5.4

$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
powershell -ExecutionPolicy Bypass -File F:\esp\esp-idf-v5.5.4\install.ps1 esp32s3
```

如果当前 shell 中的 `python` 来自 PlatformIO 虚拟环境，建议把系统 Python 放到 PATH 前面再安装或构建：

```powershell
$env:PATH = 'C:\Users\Windows\AppData\Local\Programs\Python\Python311;C:\Users\Windows\AppData\Local\Programs\Python\Python311\Scripts;' + $env:PATH
```

Windows 构建 JimTcl 还需要 GNU Make。可以使用 winget 安装：

```powershell
winget install --id ezwinports.make --source winget --accept-source-agreements --accept-package-agreements
```

每次打开新终端后，先加载 ESP-IDF 环境：

```powershell
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
$env:IDF_TOOLS_PATH = 'F:\esp\.espressif'
. F:\esp\esp-idf-v5.5.4\export.ps1
```

## 构建

设置目标芯片并构建：

```powershell
idf.py set-target esp32s3
idf.py build
```

构建成功后会生成：

- `build/openocd-on-esp32.bin`
- `build/bootloader/bootloader.bin`
- `build/partition_table/partition-table.bin`
- `build/storage.bin`

当前默认分区表中应用分区大小为 5 MB，FATFS 存储分区从 `0x510000` 开始。

## 配置 Wi-Fi

烧录前可以进入配置菜单设置 Wi-Fi：

```powershell
idf.py menuconfig
```

常用配置包括：

- Wi-Fi SSID 与密码
- Flash / PSRAM 模式
- 是否启用 ESP-BOX 屏幕 UI
- OpenOCD 启动参数

## 烧录和监视

ESP32-S3-N16R8 当前枚举为 `COM5` 时，可以直接烧录并打开监视器：

```powershell
idf.py -p COM5 flash monitor
```

通用写法如下：

```powershell
idf.py -p COMx flash monitor
```

把 `COMx` 替换成实际串口号。再次只烧录应用时可以使用：

```powershell
idf.py -p COMx app-flash monitor
```

构建完成后也可以用 esptool 手动烧录：

```powershell
python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size 16MB --flash_freq 80m 0x0 build\bootloader\bootloader.bin 0x8000 build\partition_table\partition-table.bin 0x10000 build\openocd-on-esp32.bin 0x510000 build\storage.bin
```

## 通过 JTAG 烧录应用

如果 ESP-IDF 工具目录中能找到 `openocd-esp32`，CMake 会生成两个额外目标：

```powershell
cmake --build build -t jtag-flasher-full
cmake --build build -t jtag-flasher
```

`jtag-flasher-full` 会烧录应用和 FATFS 文件系统；`jtag-flasher` 只烧录应用，默认地址为 `0x10000`。

默认 OpenOCD 配置文件为 `board/esp32s3-builtin.cfg`。

## 运行现象

打开 monitor 后，设备联网成功并启动 OpenOCD 时会看到类似日志：

```text
I (...) boot.esp32s3: SPI Flash Size : 16MB
I (...) esp_psram: Found 8MB PSRAM device
I (...) esp_psram: SPI SRAM memory test OK
I (...) main: app mode (Access Point)
I (...) network-mngr: SoftAP started: ssid="esp-openocd", channel=1, auth=open, ip=192.168.4.1
Open On-Chip Debugger 0.12.0
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : esp_gpio GPIO JTAG/SWD bitbang driver
Info : Listening on port 3333 for gdb connections
```

首次启动还没有保存 Wi-Fi 或 OpenOCD 配置时，`ESP_ERR_NVS_NOT_FOUND` 和默认开启 `esp-openocd` AP 模式是正常现象。

如果 monitor 显示 `app mode (Station)`，说明 NVS 中已经保存过 Wi-Fi SSID，固件会尝试连接该路由器而不是发出热点。需要重新回到默认 AP 模式时，可以先擦除再烧录：

```powershell
idf.py -p COM5 erase-flash
idf.py -p COM5 flash monitor
```

如果 monitor 已显示 `SoftAP started` 但手机仍搜不到 `esp-openocd`，优先确认：

- 手机或电脑正在扫描 2.4 GHz Wi-Fi，ESP32-S3 不支持 5 GHz。
- 设备离开发板足够近，并换一台手机或电脑交叉扫描。
- 如果模块是 `ESP32-S3-WROOM-1U` 或开发板带 IPEX/U.FL 外接天线座，请接好 2.4 GHz 天线。
- AP 固定在 channel 1，SSID 不隐藏，默认无密码。

如果 OpenOCD 报告 JTAG chain 异常、读到 `0x00` 或 `0x1f`，优先检查：

- 调试器板和目标板是否共地
- TCK、TMS、TDI、TDO 是否接反
- 目标板是否上电
- JTAG 引脚是否被目标固件占用
- OpenOCD 中配置的引脚是否与实际接线一致

## 使用 GDB 连接目标板

记录 monitor 中打印出的 ESP32-S3 调试器 IP，然后在目标固件工程中运行：

```powershell
xtensa-esp32-elf-gdb -ex "set remotetimeout 30" -ex "target extended-remote <调试器IP>:3333" build/blink.elf
```

示例：

```powershell
xtensa-esp32-elf-gdb -ex "set remotetimeout 30" -ex "target extended-remote 192.168.0.221:3333" build/blink.elf
```

## Web 配网与配置

首次运行时，如果尚未配置 Wi-Fi，设备会创建默认开放热点：

```text
SSID: esp-openocd
IP:   192.168.4.1
```

连接该热点后，在浏览器打开 `http://192.168.4.1`，可以配置 Wi-Fi 和 OpenOCD 命令行参数。

默认只预置 Espressif 芯片相关配置文件。其他芯片配置可以通过 Web 页面上传到文件系统。

## ESP-BOX

项目可以在 ESP-BOX 开发板上启用触摸屏配置界面和配网功能。默认 ESP-BOX 与 UI 功能关闭，需要通过 `idf.py menuconfig` 启用。

## 常见问题

### `idf.py` 不存在

说明当前终端没有加载 ESP-IDF 环境。先执行：

```powershell
. F:\esp\esp-idf-v5.5.4\export.ps1
```

### ESP-IDF 安装时报 `Windows-` 平台不支持

当前 Python 环境可能无法识别 CPU 架构。安装前设置：

```powershell
$env:PROCESSOR_ARCHITECTURE = 'AMD64'
```

### ESP-IDF 不能在虚拟环境里创建虚拟环境

如果当前 `python` 来自 PlatformIO 的 `penv`，请切换到系统 Python，再运行 ESP-IDF 安装脚本。

### Windows 下 JimTcl 或 OpenOCD 子步骤找不到 shell 工具

请确认已安装 Git for Windows，并安装 GNU Make：

```powershell
winget install --id ezwinports.make --source winget
```

### ESP32-S3-N16R8 启动时报 PSRAM 模式错误

如果日志中出现 `PSRAM chip is not connected`、`wrong PSRAM line mode` 或 `Failed to init external RAM`，通常是 PSRAM 模式不匹配。ESP32-S3-N16R8 应使用 16 MB Flash 和 Octal PSRAM，确认配置中包含：

```ini
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="16MB"
CONFIG_SPIRAM_MODE_OCT=y
```

## 许可证

本仓库中 ESP-IDF 应用部分版权归 Espressif Systems (Shanghai) Co. Ltd. 所有，采用 Apache 2.0 许可证，详见 [LICENSE](LICENSE)。

OpenOCD 子模块遵循 GPL v2.0 或更新版本许可证。

## 贡献

欢迎提交问题、功能建议和 Pull Request。提交前建议安装 pre-commit：

```powershell
pip install pre-commit
pre-commit install
```

如果 pre-commit 修改了文件，请重新 `git add` 后再次提交。
