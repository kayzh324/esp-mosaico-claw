# 板级资料：JC-ESP32P4-M3-DEV（Guition / Jingcai）

> 用途：本产品 P4 目标的**板级适配依据**（mosaico `boards/` 新板定义、`sdkconfig.defaults`、分区规划）。
> 来源：板载原理图 PNG（`5-Schematic/1..8`）、模块规格、ESP32-P4 官方资料，以及**你可复现的实测结论**。
> 本地资料路径：`~/Downloads/嵌入式文档/esp32-P4-C6开发板/JC-ESP32P4-M3-DEV/`
> 关联：`docs/p4-vs-s31-feasibility.md`、`docs/mosaico-fork-analysis-and-decision.md`

---

## 0. 结论（板级可用性）

**该板把本方案需要的绝大部分能力都板载齐了**：P4 + 板载 C6（Wi-Fi）、32 MB PSRAM、16 MB NOR、MIPI-DSI（2-lane）、MIPI-CSI、ES8311 音频（含模拟麦 + NS4150 功放）、TF 卡、两路 USB-OTG、以太网、4 个用户按键、扩展 IO 排针。
**缺口只有**：无 IMU/磁力计、无电量计、无 NAND、无触摸（你的屏也不带）。

已实测（你的确认）：**音频可录可放、USB HID（2.4G 键鼠）可枚举、Wi-Fi 正常、SD 正常、JD9365 DSI 屏可显示**。

---

## 1. 核心构成

| 项 | 内容 |
|---|---|
| 主控 | **ESP32-P4**（双核 RISC-V；模块标称 360 MHz，芯片最高 400 MHz） |
| 无线 | **板载 ESP32-C6**（Wi-Fi 6 + BLE 5），经 **SDIO** 走 **ESP-Hosted** |
| 内存 | **32 MB PSRAM（封装内堆叠）** |
| 存储 | **16 MB NOR Flash** |
| 供电 | USB 5 V；**TLV62569** 降压 → 3V3；**IP5306**（带电池管理，`BAT+/-` 接 CN4，可接电池） |
| 板尺寸 | 92 × 62 mm（模块 27 × 27 × 3.4 mm） |

---

## 2. 外设与连接器（按原理图）

### 2.1 显示 / 背光 / 触摸 / 摄像
| 项 | 内容 |
|---|---|
| **DSI 屏** | 连接器 **J2**（15-pin FPC），**2-lane**：`DSI_A_DATA0/1` + `DSI_A_CLK` 差分对 + I2C + 3V3。模块标称最大 **800×1280** |
| 你的屏 | **JD9365，10.1″，无触控**（实测点亮） |
| **背光** | 独立 **CN5**（4-pin，`LEDA`/`LEDK`），**MP3202** 升压驱动，**PWM 调光**（网络 `LCD_PWM`） |
| **CSI 摄像头** | 连接器 **J3**（15-pin FPC），2-lane：`CSI_A_DATA0/1` + `CSI_A_CLK` + I2C |
| 电容触摸（预留） | **FPC1**（6-pin 0.5 mm）：`RTC_DAT/CLK`、`TOUCH_INT`、`TOUCH_RST`（你的屏无触控，不用） |

> P4 的 MIPI-DSI/CSI 使用**专用引脚**（非 GPIO 矩阵），所以"DSI 引脚"由 SoC 固定 pad 决定，不占用 GPIO。

### 2.2 音频（**已实测可录音 + 可放音**）
| 项 | 内容 |
|---|---|
| Codec | **ES8311**（U1）：I2C 控制 + I2S0 数据 |
| 麦克风 | **模拟麦 MIC1** `MSM381A3729H9CP`（差分输入 `MIC1P/N` → ES8311 ADC） |
| 功放 | **NS4150**（U2）单声道 Class-D，使能 `PA_CTRL`；输出到 **J1 扬声器座**（2-pin） |
| I2S0 信号 | `CODEC_I2S0_SCLK` / `..._LRCK` / `..._DSDIN` / `..._MCLK`；codec 数据线复用名 `ES7210_SDOUT` |

- I2S0 相关 GPIO（模块图）：`DSDIN=GPIO9`、`LRCK=GPIO10`、`SCLK=GPIO12`、`MCLK=GPIO13`、`SDOUT(ADC)=GPIO48`。
- Codec/DSI/CSI/RTC 共用 **`ES_I2C` 总线**。

### 2.3 存储 / 网络 / USB / 扩展 / 按键 / 调试
| 项 | 内容 |
|---|---|
| **TF 卡** | J1，**4-bit SDIO**（`SD_DAT0..3`、`SD_CMD`、`SD_CLK`），带电平转换 |
| **以太网** | P4 内部 EMAC + **外部 PHY**（25 MHz 晶振）+ **RJ45**（含网络变压器与指示灯）；RMII 走 `GPIO28..35 / 49..52` |
| **USB1（Type-C）** | 经 **CH340C** 做串口 → 烧录/日志（DTR/RTS 自动控制 `BOOTMODE`/`CHIP_PU`） |
| **USB2（Type-C）** | *Full-Speed* OTG（P4 FS USB） |
| **USB3（Type-C）** | *High-Speed* OTG（P4 HS USB，`ESP_USB_N/P`）→ **HID host 用这路**（你已实测） |
| **RS485** | **MAX485**（U8）+ 接线端子 |
| **扩展 IO 排针 JP1** | 26-pin（2×13）：引出 `GPIO1..5`、`GPIO20`、`GPIO32`、`GPIO33`、`GPIO45/46/47`、**`ES_I2C`（SDA/SCL）**、C6 UART、`C6_IO9`、`C6_CHIP_PU`、3V3/5V/GND → **接外部 I2C 外设的入口** |
| **用户按键** | **S1 / S2 / S3 / S4**（4 个） |
| **系统按键** | `SW1`=BOOT、`SW2`=RESET |
| **调试/烧录** | 串口排针 `CN2`（MX1.25 4P：`TXD_BOOT`/`RXD`/GND/VIN）+ CH340C |

> 说明：具体 GPIO 与网络的逐一对应，请在写新板 YAML 时以 `5-Schematic/4_ESP32P4.png`（模块引脚表）与各功能页**逐条转录**为准。

---

## 3. 板载参考工程与镜像（本机目录内）

| 资源 | 路径（相对资料根） | 价值 |
|---|---|---|
| **完整 P4 工程** | `1-Demo/IDF-DEMO/xiaozhi-esp32/` | **P4 + C6(SDIO/esp_hosted) + DSI + ES8311 + LVGL** 的可跑参考；含 `managed_components/espressif__esp_hosted` 等 |
| **C6 固件** | `8-Burn operation/Burn files/JC1060P470_C6.bin` | 板载 C6 的出厂固件 → **esp_hosted 主从版本要对齐** |
| **整板测试固件** | `8-Burn operation/Burn files/JC-ESP32P4-M3-DEV-TEST-V1.3.bin` | 出厂全功能固件 |
| 烧录说明 | `8-Burn operation/烧录说明：Burning instructions.txt` + `flash_download_tool_3.9.7` | 出厂烧录流程 |
| 规格书/手册 | `2-Specification/JC-ESP32P4-M3-DEV Specifications-EN.pdf`、`6-User_Manual/Getting started ...pdf` | 规格与上手 |
| 结构图 | `3-Structure_Diagram/5.jpg` | 尺寸/接口位置 |

> **重要**：esp_hosted 需要 C6 侧跑**匹配版本的 slave 固件**。该目录自带 `JC1060P470_C6.bin`，可作为基线；升级 `esp_hosted` 时需同步 C6 固件。

---

## 4. 与 mosaico 原 S31 板的差异（板级视角）

| 能力 | mosaico S31 板（`esp_mosaico`） | JC-ESP32P4-M3-DEV | 适配动作 |
|---|---|---|---|
| 显示 | CO5300 **QSPI** 480×480 圆屏 | **MIPI-DSI** 800×1280（JD9365） | 新板 `display_lcd sub_type: dsi` + JD9365 面板驱动 |
| 触摸 | CST9217（I2C） | **无**（可用 FPC 预留） | 不需要；输入走 GPIO 按键 + USB HID |
| 音频 codec | ES8311 | ES8311（**相同**） | 直接复用 |
| 功放 | 有 | NS4150 | 复用 |
| 存储 | SPI NAND + NOR | NOR 16 MB + TF 卡 | 无 NAND（Vibe Mode/Ideas 才用 NAND） |
| 网络 | Wi-Fi（S31 内置） | Wi-Fi（**C6 + esp_hosted**）+ **以太网** | `wifi_manager` 已含 P4 分支 |
| USB | 1× OTG | **2× OTG（HS+FS）** | HID host 用 HS 口 |
| 传感器 | IMU + 磁力计 + 电量计 | **无** | 裁剪 `app_claw_hw_bridge` 资源表；外部 I2C 可选加 |
| 相机 | DVP | **MIPI-CSI** | 若用相机，改 CSI 传感器 |
| 按键 | AI 键等 | S1..S4 + BOOT/RESET | 映射到输入层 |

---

## 5. 板级适配待办（写新板 YAML 时）

1. **逐条转录引脚**（`4_ESP32P4.png`）：`ES_I2C` SCL/SDA、I2S0 五线、`PA_CTRL`、`LCD_PWM`、SD、RMII、`S1..S4` 实际 GPIO、C6 复位/启动脚。
2. **DSI 面板**：JD9365 初始化序列（复用你已验证的工程/驱动），时序与 2-lane 参数。
3. **背光**：`LCD_PWM` 走 LEDC，映射到显示背光控制。
4. **音频**：ES8311 I2C 地址与 I2S 格式；`PA_CTRL` 作为功放使能 GPIO。
5. **SD**：SDMMC 4-bit 引脚（与 P4 SDMMC 槽位对齐）。
6. **以太网（可选）**：RMII 引脚 + PHY 复位 `PHY_RSTN`。
7. **`sdkconfig.defaults`**：32 MB PSRAM（模式/速率）、ESP-Hosted、USB host、CSI（若用）。
8. **分区表**：16 MB NOR 在"去 Vibe Mode"时的 `main_app + ui_apps + system` 规划。
9. **C6 固件**：确认与所选 `esp_hosted` 版本匹配（可参考目录内 `JC1060P470_C6.bin`）。

---

*本文档为板级资料汇总。引脚级细节以板载原理图为最终依据；能力结论已由实测与原理图交叉验证。*
