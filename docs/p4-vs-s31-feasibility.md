# ESP32-P4 vs ESP32-S31：本方案的可移植性评估

> 状态：技术评估（基于官方资料 + 本仓库代码勘察）
> 评估问题：**"服务层 + 自研响应式 GSP UI"方案原本面向 ESP32-S31，换到 ESP32-P4 是否成立？有无硬性不足？**
> 关联：`docs/mosaico-fork-analysis-and-decision.md`、`docs/vibe-mode-and-ideas-analysis.md`、`docs/board-jc-esp32p4-m3-dev.md`（板级资料）

---

## 0. 结论先行

**没有硬性不足打断核心方案；而且对"P4 + 10.1″ MIPI-DSI 大屏"这个具体目标，P4 比 S31 更合适。**

- **PPA / GSP / 2D-DMA**：P4 全部具备，`espressif/esp-gsp` **官方目标列表含 `esp32p4`**，且有 **ESP32-P4 + MIPI-DSI 的官方 benchmark 例程**（RGB565/RGB888）。你"自写 GSP 场景层 + PPA 加速 + 编译期响应式"的路线在 P4 上直接成立。
- **MIPI-DSI**：P4 **原生支持**；**S31 不支持 MIPI-DSI**（只有并行 RGB / I8080 / MOTO6800，摄像头是 DVP 而非 CSI）。你的 10.1″ DSI 屏在 **P4 上自然对接，在 S31 上反而要加桥接**。
- **USB HID host**：P4 有 **USB 2.0 HS OTG + FS OTG 两个控制器**，官方提供 **HID host（键盘/鼠标）例程** → 你要的 USB HID 键鼠直接可用。
- **"无线"不构成缺口**：P4 硬件确实无内置 Wi-Fi/BT（S31 内置 Wi-Fi6+BT5.4+802.15.4），需伴生芯片走 **ESP-Hosted / esp_wifi_remote**；**但产品层 `wifi_manager` 早已内置 P4 支持**（见 §2.7），你的 P4 板也板载 C6。
- **上层封装已大幅为 P4 准备**：显示(D)、触摸、Wi-Fi、能力桥等**服务层大体与芯片解耦**；S31 锁定集中在**板级 + 启动 + Vibe Mode + 打包目标**（见 §2.7）。

**一句话**：没有架构级硬缺口；无线由伴生芯片 + 产品层已内置的 `esp_wifi_remote`/`esp_hosted` 解决。剩余为**板级 + 启动 + 打包目标**的移植工作量，以及你本就要重写的 UI 层。

---

## 1. 两芯片关键差异对照

| 维度 | **ESP32-S31**（现行方案） | **ESP32-P4**（你的目标） | 对本方案 |
|---|---|---|---|
| 无线 | Wi-Fi 6 + BT 5.4(LE+Classic) + 802.15.4 **集成** | **无任何无线** | ⚠️ 硬件需伴生芯片，**但产品层已内置 P4 支持** |
| CPU | 双核 RISC-V @ 320 MHz（一核 128-bit SIMD） | 双核 RISC-V @ 更高（另含 LP 核） | ✅ |
| PPA | 有 | 有 | ✅ 关键 |
| 2D-DMA | 有 | 有 | ✅ |
| JPEG | 有 | 有（另有 ISP、**H.264 硬件编码**） | ✅ |
| **MIPI-DSI** | **无**（并行 RGB/I8080/MOTO6800） | **原生** | ✅ **P4 更适合** |
| MIPI-CSI | 无（DVP camera） | 原生 | 视需求 |
| USB | USB 2.0 HS OTG | **HS OTG + FS OTG 双控制器** | ✅ HID host 官方例程 |
| PSRAM | 8-bit DDR @ 250 MHz（封装内） | **最高 32 MB** | ✅ 大屏多缓冲更从容 |
| 以太网 | **1000 Mbps** MAC | 10/100 EMAC | 视需求（S31 更强） |
| GPIO | 60 | 较多 | ✅ |
| 电容触摸 | 最多 14 路集成 | 需外置触摸 IC | 你当前无触控，无影响 |
| SoC 成熟度 | 数据手册 v0.5 **PRELIMINARY** | 已量产/多开发板 | ✅ P4 更成熟 |

---

## 2. 逐项硬性检查

### 2.1 PPA / GSP —— ✅ 支持，且官方在 P4+DSI 上有基准例程
- ESP-IDF 明确提供 **ESP32-P4 的 PPA（`esp_driver_ppa`）**：旋转/缩放/镜像/混合。
- `espressif/esp-gsp` 的**目标列表**：`esp32c3 / esp32s3 / **esp32p4** / esp32s31 / esp32c5 / esp32c6`。
- ESP-GSP 的 benchmark 例程**直接以 ESP32-P4 + MIPI-DSI（RGB565 与 RGB888）**为构建目标——证明"GSP 在 P4 的 MIPI-DSI 上跑"是官方验证过的组合。
- ESP-GSP 自述的一个限制恰好**印证你"编译期响应式"的决策**：*"A bundle uses one logical scene resolution and one output pixel format."* —— 即**每个 bundle 固定一种逻辑分辨率与像素格式**，响应式必须在**生成/编译期**按目标形态产出多个 bundle。这与你的方案一致。
- 其他 ESP-GSP 限制（需知）：不支持 RTL 布局与复杂文字整形（阿拉伯文等）；捏合手势不自动缩放控件。

### 2.2 大屏 / 分辨率 —— ⚠️ 需实测，但硬件可行
- 10.1″ 常见 1280×800（≈1.02 MP）。RGB565 单帧 ≈2 MB；双/三缓冲 ≈4–6 MB + GSP 资源。
- **P4 最高 32 MB PSRAM**，你的板正是 **32 MB**，容量充足；带宽是主要变量（PPA 合成 + LCD 扫描并发）。
- 你的屏是 **JD9365 10.1″（800×1280）**，P4 DSI 为 **2-lane × 1.5 Gbps**，带宽可覆盖。
- 结论：**容量不是问题，帧率/带宽仍需实测**（既有 Spike 项，见主文档 §7）。

### 2.3 USB（GPIO 按钮之外的 HID 输入）—— ✅ 支持
- P4 有 **USB 2.0 HS OTG（DWC_OTG）**，官方 **USB Host HID 例程**（键盘/鼠标）就在 esp32p4 文档内。
- 你的"输入归一化层"（触控/GPIO/USB HID → pointer/key/action）在 P4 上有现成底座。
- ✅ 板级已验证：JC 板有 **HS + FS 两路 OTG**，你已实测 **2.4G 无线键鼠**可枚举。

### 2.4 无线 —— ⚠️ 硬件需伴生芯片，产品层已内置解决
- P4 **无内置无线**。官方指引：用任意 ESP32-C/S 系列作伴生芯片，经 **SPI/SDIO/UART** 走 **ESP-Hosted / esp_wifi_remote**（或有 flash 的 **esp-extconn**）。
- 影响面（本产品**深度联网**）：
  - 产品服务层：`wifi_manager`、云 LLM/ASR、IM、MCP、天气、更新检查、微信绑定…
  - Vibe Mode：HTTPS Bridge、Download Ideas、在线更新检查、系统更新
  - 蓝牙：**BLE 可用**；**蓝牙音频（A2DP / Classic BR/EDR）不可用** —— 见 §7.2。
- 现实：你的 **JC-ESP32P4-M3-DEV 板载 ESP32-C6-MINI**（SDIO + ESP-Hosted）。软件侧走 `esp_wifi_remote`/`esp_hosted`，标准 `esp_wifi` API 不变。
- 这是**架构性新增**（BOM + 布线 + 伴生固件烧录/配网 + 电源管理差异），但不是不可行——**是成熟方案**。
- **关键**：本仓库 `wifi_manager` 已按 `target in [esp32p4]` 条件引入 `espressif/esp_wifi_remote` + `espressif/esp_hosted`，并在 `.c` 中用 `esp_hosted_misc.h` 取 AP MAC（见 §2.7）。**即无线支持在产品层已写好**，P4 上走标准 `esp_wifi` API。

### 2.5 S31 专属的低层实现 —— ⚠️ 需为 P4 重做/复核
| 机制 | 位置 | P4 情况 |
|---|---|---|
| 启动前清 USB DMA（`usb_utmi_ll_*`） | Vibe Mode bootloader | P4 也有 USB HS OTG/UTMI，但**须逐符号复核** |
| 屏幕交接寄存器 `LP_STORE15` | Vibe Mode bootloader + `mosaico_boot_handoff.h` | 须复核 P4 是否同名可用 |
| eFuse USER_DATA 低 16 位读板型 | Vibe Mode bootloader | 须复核 P4 映射 |
| 启动 Logo（CO5300/QSPI，SPI2 40 MHz 半双工） | Vibe Mode bootloader | **与屏强绑定**，DSI 屏必须重写 |
| bootloader 24 KiB 容量上限 | Vibe Mode 构建检查 | P4 需重新核容量预算 |

### 2.6 组件与目标支持 —— ⚠️ 待逐项确认
- 打包脚本 `tools/package_firmware.py` 中 **`TARGET = "esp32s31"` 写死**。
- 需逐项确认 P4 构建支持：**esp-claw** 内核、`esp-gsp`（✅ 已确认含 P4）、`esp_vision_core`（✅ esp-vision 支持 P4 板）、`esp_bt_audio`、`esp_display_present`/`esp_lvgl_present`、`esp-mosaico-bsp`、`esp_idf_patch.cmake` 打补丁的 IDF 源。
- Vibe Mode 包的 `compatibility.chip_target` 硬校验为 `esp32s31` → **P4 必须先重建整个 Vibe Mode 工程**才能通过校验。

### 2.7 上层封装（mosaico 产品层）在 P4 上的就绪度 —— **已大幅准备**

> 关键结论：**mosaico 的"服务层/封装层"大体与芯片解耦，部分关键件已内置 P4 支持**。S31 锁定集中在**板级 + 启动 + Vibe Mode + 打包目标**，**不在服务层**。

**已就绪证据（本仓库代码）：**

| 组件 | P4 就绪证据 |
|---|---|
| `wifi_manager` | `idf_component.yml` 按 `target in [esp32p4]` 引入 `espressif/esp_wifi_remote` + `espressif/esp_hosted`；`wifi_manager.c` 有 `#ifdef CONFIG_IDF_TARGET_ESP32P4` 分支，用 `esp_hosted_misc.h` 取 AP MAC。**无线缺口在产品层已被内置解决。** |
| `display_service` | 有 `DISPLAY_SERVICE_PANEL_INTERFACE_DSI` 枚举；`display_service_lvgl.c` 对 DSI/RGB/RGB_3WIRE_SPI 有**真实处理**（旋转、buffer、PSRAM），由板级能力宏 `CONFIG_ESP_BOARD_DEV_DISPLAY_LCD_SUB_DSI_SUPPORT` 门控 |
| `esp_lvgl_present` | 提供 `ESP_LV_ADAPTER_DISPLAY_MIPI_DEFAULT_CONFIG(...)`（MIPI=DSI）宏；依赖 `esp_display_present ^1.0.2` |
| `esp_lcd_touch_mux` | 测试 app 编译矩阵含 **ESP32-P4 Passed** |
| `app_claw` 硬件桥 | `app_claw_hw_bridge.c` 用 `SOC_*` 能力宏生成资源表 → 天然可移植 |
| `cap_system` | `chip_model` 取 `CONFIG_IDF_TARGET` → 可移植 |
| `mosaico_camera_service` | 未见 DVP/MIPI/芯片硬编码 → 接口由**板级**决定，服务层解耦 |
| `esp-gsp` | 目标列表含 `esp32p4`，且官方有 P4 + MIPI-DSI benchmark |
| IDF 补丁 (`tools/cmake/esp_idf_patch.cmake`) | 改的是 `esp_lcd/parl`、`usb_types_ch9.h`、`esp_http_client`，**与 target 无关**（同一 IDF 6.x） |

**仍需为 P4 重做（S31 锁定点，全部在板级/启动/打包）：**

| 项 | 位置 | 处理 |
|---|---|---|
| 板级定义 | `boards/esp_mosaico/`（`chip: esp32s31`，CO5300 QSPI 圆屏） | **新建 P4 板**：`board_info`(chip `esp32p4`) + 设备 YAML（DSI 面板 `sub_type: dsi`）+ `setup_*.c` 面板工厂入口 + 用 `idf.py bmgr` 重生成代码 |
| 启动 splash | `mosaico_boot_splash`（CO5300/QSPI + `LP_STORE15` 交接） | 去除，或为新屏重写 |
| USB console | `boards/esp_mosaico/components/usb_hs_console`（`usb_utmi_ll`） | P4 亦有 HS OTG/UTMI，提供/复用即可 |
| Vibe Mode / 分区表 | `retained_recovery.cmake`、`partitions_16MB.csv`（S31 ABI） | 去除，或为 P4 重建 |
| 打包目标 | `tools/package_firmware.py` 的 `TARGET="esp32s31"` | 改 P4 + 相应 `sdkconfig.defaults` |
| 相机（若使用） | 板级 DVP → P4 MIPI-CSI | 板级替换 |
| `sdkconfig.defaults` | 含 PSRAM/lwip/audio/vision 等 | 多数可移植；`SPIRAM_XIP_FROM_PSRAM`、mbedtls 硬件加速等需按 P4 复核 |

**判断**：既然 esp-claw 内核已在 P4 上单烧跑通，且产品层服务件（Wi-Fi / 显示 / 触摸 / 能力桥）**大体已 P4-ready**，则 **mosaico 级上层封装在 P4 上可行**。剩余工作是**板级 + 启动 + 打包目标**适配，以及你本就要重写的 UI 层；**不涉及服务层的架构性改动**。

---

## 3. 硬性缺口清单（结论）

| # | 缺口 | 硬度 | 性质 |
|---|---|---|---|
| 1 | **P4 无 Wi-Fi/BT** | ~~硬~~ → **产品层已解决** | 硬件需伴生芯片，但 `wifi_manager` 已含 `esp_wifi_remote`/`esp_hosted` 的 P4 分支（§2.7） |
| 2 | 预编译 Vibe Mode 包是 **S31 专属二进制** | 硬（产物不可移植） | 必须为 P4 **重建整个 Vibe Mode 工程**（ABI/机制可移植，产物不可） |
| 3 | boot splash / 屏幕交接为 **S31+CO5300/QSPI 专属** | 中 | 换 P4+DSI 必须重写 |
| 4 | S31 专属低层代码（UTMI 清理、LP_STORE15、eFuse 板型） | 中 | 逐项为 P4 复核 |
| 5 | 组件/打包目标的 P4 支持未逐一验证 | 中 | 逐项确认 + 改 `TARGET` |
| 6 | **蓝牙音频 A2DP（Classic BR/EDR）** | **硬**（C6 无 BR/EDR 射频） | 板载 C6 **仅 BLE**；ESP-Hosted 官方：Classic 仅 ESP32 协处理器支持。`mosaic_ui/apps/bluetooth` 是 A2DP → P4+C6 **无法运行**（见 §7.2） |

**没有任何一条是"方案在 P4 上根本做不了"**；Wi-Fi/显示/内存等均已就绪，缺口落在**板级 + 启动 + 打包目标**（不在服务层，§2.7）。**唯一的硬功能损失是蓝牙音频 A2DP（C6 无 Classic BR/EDR 射频，见 §7.2）**。

---

## 4. 反而对 P4 有利的点

1. **PPA + GSP + MIPI-DSI** 是官方**已基准**的组合 → 你"自写 GSP 场景层用 PPA"的核心路线在 P4 上更顺。
2. **MIPI-DSI 原生**：你的 10.1″ DSI 屏在 P4 上是"正路"；在 S31 上反而要加 RGB/DSI 桥接。
3. **PSRAM 最高 32 MB**：大分辨率多缓冲更从容。
4. **USB HS+FS 双 OTG + 官方 HID host**：多输入方案底座齐备。
5. **P4 更成熟**：已量产、开发板丰富；S31 数据手册仍是 PRELIMINARY。

---

## 5. 对主方案的影响

- **主方案（服务层跟随 + 自研响应式 GSP UI）不变**，且在 P4 上更自然。
- **无线**：**不是新增架构**，而是启用产品层**已就绪**的 P4 通路（`esp_wifi_remote` + `esp_hosted` + 板上 C6）。只需确认伴生固件版本与配网/电源策略；`wifi_manager` 无需重写。
- **Vibe Mode 决策与芯片强耦合**：若保留 Vibe Mode，则必须为 P4 重建其工程（含 bootloader/驱动/网络）；若去除，则 §3 的缺口 2–5 里与 Vibe Mode 相关的部分一并消失。**建议把"Vibe Mode 去留"与"是否要工厂级刷不坏维护固件"一起决策**（见主文档 §8）。

---

## 6. 建议的验证动作（追加到 Spike）

1. **无线通路**：直接用本仓库 `wifi_manager` 的 P4 分支（`target in [esp32p4]`）构建，确认与板上 C6 的 `esp_hosted` 版本握手成功、标准 `esp_wifi` API 联网。
2. **GSP on P4 + DSI**：直接用 `espressif/esp-gsp` 的 P4-MIPI-DSI 例程跑一版，测目标分辨率帧率/内存。
3. **USB HID host**：跑 IDF 的 `peripherals/usb/host/hid` 例程，确认键鼠枚举与输入上报。
4. **组件目标盘点**：对 `components/` 与 esp-claw 逐个确认 P4 构建支持，输出"需改清单"（预期集中在板级/启动/打包，见 §2.7）。
5. **Vibe Mode（若保留）**：在 P4 上重建 utils 工程（含 bootloader 适配新屏 + 24 KiB 容量校验），跑 `prepare_recovery.py` 通过校验。
6. **蓝牙**：确认放弃 A2DP（C6 无 BR/EDR，已决定）；如需 BLE 用途，验证 `esp_hosted` 的 BLE 通路。
7. **板级转录**：按原理图 `4_ESP32P4.png` 逐条落位 `ES_I2C`/I2S0/`PA_CTRL`/`LCD_PWM`/SD/RMII/`S1..S4`/C6 复位脚（清单见 `docs/board-jc-esp32p4-m3-dev.md` §5）。

---

## 7. 目标硬件盘点：JC-ESP32P4-M3-DEV + 蓝牙硬约束

### 7.1 板载资源（Guition / Jingcai JC-ESP32P4-M3-DEV）

| 项 | 规格 | 对本方案 |
|---|---|---|
| 主控 | ESP32-P4 双核 RISC-V（模块标称 360 MHz，芯片最高 400 MHz） | ✅ |
| 无线协处理器 | **板载 ESP32-C6**（Wi-Fi 6 + Bluetooth 5 / BLE），经 **SDIO** 走 ESP-Hosted | ✅ Wi-Fi；⚠️ BT 仅 BLE（见 §7.2） |
| 内存 | **32 MB PSRAM（封装内堆叠）** | ✅ 大屏多缓冲 |
| Flash | **16 MB NOR** | ✅ 与 `partitions_16MB.csv` 容量一致 |
| 显示 | **MIPI-DSI 2-lane**（J2，模块标称最大 800×1280）；背光 MP3202 + `LCD_PWM` | ✅ JD9365 10.1″ 无触控（实测） |
| 摄像 | **MIPI-CSI 2-lane**（J3，含 ISP） | ✅（若用相机） |
| 音频 | **ES8311 + 模拟麦 + NS4150 功放 + 扬声器座** | ✅ 实测可录可放 |
| USB | **HS OTG + FS OTG 两路 Type-C**（另有 CH340C 烧录口） | ✅ HID host 已实测 |
| 存储扩展 | **TF 卡（4-bit SDIO）** | ✅ |
| 网络 | Wi-Fi（C6）+ **以太网（P4 EMAC + PHY + RJ45）** | ✅ |
| 扩展 | **JP1 26-pin**（含 `ES_I2C`、GPIO1..5/20/32/33/45/46/47） | ✅ 外接 I2C 外设 |
| 按键 | S1–S4 用户键 + BOOT/RESET | ✅ |
| 传感器 | **无** IMU/磁力计/电量计 | ⚠️ 裁剪资源表（可选外接） |
| 参考 | 原理图 PNG：`5-Schematic/1..8`；规格书、手册、**P4+C6 DSI 参考工程（xiaozhi-esp32）**、**C6 固件** | **详见 `docs/board-jc-esp32p4-m3-dev.md`** |

### 7.2 蓝牙：**Classic（A2DP）是硬缺失**

- 官方 ESP-Hosted 明确：**"Classic Bluetooth (BR/EDR) is supported only on an ESP32 co-processor. The C/S/H-series chips are BLE-only and will fail at controller init."** 协处理器支持表：ESP32 = Yes；C2/C3/C5/**C6**/C61/H2/S3 = No。
- 你的板载是 **C6** → **只有 BLE，没有 Classic BR/EDR 射频**。
- 而本产品蓝牙音频 App **正是 A2DP（Classic）**：`mosaic_ui/apps/bluetooth/bluetooth_audio_runtime.c` 引 `esp_bt_audio_classic.h`、`app.cmake` 编 `bluetooth_a2dp_lifecycle.c`、依赖 `espressif/esp_bt_audio ^1.1.0`。
- **结论**：`mosaic_ui/apps/bluetooth`（蓝牙音箱 / A2DP Sink）在 **P4 + C6 上无法运行**。LE Audio 亦不可用（C6 无 ISO；且 ESP-BLE-AUDIO 官方标注为 preview）。
- **处理建议**：
  - 首选 **去除该 App**（它是独立 App，且你本就要重写 UI）。
  - 若蓝牙音频是硬需求 → 需把射频协处理器换成 **ESP32（原版，带 BR/EDR）**，代价是失去 Wi-Fi 6/802.15.4 且需重做射频，成本高——**先确认是否真的需要**。
- 影响**仅限"蓝牙音频"**：**Wi-Fi（C6）与 BLE（如 BLE 配网）不受影响**。

### 7.3 板级确认结果（已核对原理图 + 实测）

> 详细清单见 `docs/board-jc-esp32p4-m3-dev.md`。

| 项 | 结论 | 依据 |
|---|---|---|
| **音频** | ✅ **ES8311 + 模拟麦（MSM381A3729H9CP）+ NS4150 功放 + 扬声器座**；已实测**可录音可放音** | 原理图 `1_PWR&SPEAKER` / `3_8311&TFCARD` / `6_MIPI_DSI&MIPI_CSI` + 实测 |
| **USB HID** | ✅ 有 **HS + FS 两路 OTG**；已实测 **2.4G 无线键鼠**可枚举 | 原理图 `5_USB&485` + 实测 |
| **显示屏** | ✅ **JD9365 10.1″ 无触控**，DSI **2-lane**（J2），背光 **MP3202 + `LCD_PWM` 调光**（CN5） | 原理图 + 实测点亮 |
| **存储** | ✅ **16 MB NOR** + **TF 卡（4-bit SDIO）**；无 NAND | 原理图 + 实测 SD |
| **无线** | ✅ **板载 C6（SDIO + ESP-Hosted）**，Wi-Fi 已实测正常 | 原理图 + 实测 |
| **以太网** | ✅ P4 内部 EMAC + 外部 PHY + RJ45（**额外能力**） | 原理图 `7_RJ45` |
| **传感器** | ❌ 无 IMU / 磁力计 / 电量计；有 **JP1 扩展排针**可外接 I2C 外设 | 原理图 `2_EXPAND_IO&BAT` / `8_OTHER` |
| **按键** | ✅ S1–S4 用户键 + BOOT/RESET | 原理图 `1_PWR&SPEAKER` / `4_ESP32P4` |
| **蓝牙音频** | ❌ **A2DP 不可用**（C6 无 BR/EDR），已决定**去掉该 App** | §7.2 |

**剩余待办不是"验证"，而是"转录与适配"**（写新板 YAML 时执行）：
1. 逐条转录引脚：`ES_I2C` SCL/SDA、I2S0 五线、`PA_CTRL`、`LCD_PWM`、SD、RMII、`S1..S4` 实际 GPIO、C6 复位/启动脚。
2. JD9365 面板初始化序列与 2-lane 时序（复用已验证工程）。
3. 32 MB PSRAM 的 `sdkconfig`（模式/速率）+ ESP-Hosted + USB host（+ CSI 若用）。
4. 16 MB NOR 分区规划（去 Vibe Mode 后）。
5. C6 侧固件与 `esp_hosted` 版本对齐（板卡资料内含 `JC1060P470_C6.bin`）。

### 7.4 更新后的结论

在 JC-ESP32P4-M3-DEV 上：**Wi-Fi ✅ · DSI 显示 ✅ · 音频录放 ✅ · USB HID ✅ · SD ✅ · 以太网 ✅ · 内存/Flash ✅ · PPA/GSP ✅**。
**唯一的硬功能缺失是"蓝牙音频（A2DP / Classic）"**（C6 无 BR/EDR）——已决定去掉该 App，与 UI 重写正交。
**没有剩余的功能性验证项**；下一步是板级引脚转录 + 新板定义 + UI 重写。

---

*本文档为技术评估。芯片结论以官方资料与本仓库代码为准；未实测项已在 §2/§6/§7.3 标注。*
