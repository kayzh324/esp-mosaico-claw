# ESP-MOSAICO-CLAW 二开可行性分析与最终决议

> 状态：**动手前调研已完成，待开工**
> 目的：记录一次完整的架构/耦合分析结论与最终方案，后续正式开工时直接取用，无需重新扫库。
> 适用仓库：`esp-mosaico-claw`（本仓库，产品整机工程）
> 关联文档：`docs/vibe-mode-and-ideas-analysis.md`（Vibe Mode/Ideas 实现分析）、`docs/p4-vs-s31-feasibility.md`（P4 可移植性评估）、`docs/board-jc-esp32p4-m3-dev.md`（目标板板级资料）
> 本文档基于的分析快照：`master` @ `2e1017e`，esp-claw submodule @ `62aaeed7de8f3055615448620dd00ed39fd6af13`，esp-mosaico-utils submodule @ `ca2f9c1eae4d95cd11ca49487949a8f5e7a39f9c`。
> 调研补充（本轮完成）：Vibe Mode/Ideas 实现机制、P4 可移植性（含芯片与产品层证据）、目标板 JC-ESP32P4-M3-DEV 板级事实与实测结论。

---

## 0. TL;DR（最终决议）

- **采用方案：`mosaic 服务层 + 自研响应式 UI`。**
- **UI 路线：自写 GSP 场景层**，以复用 PPA 硬件加速，保证流畅度。
- **响应式策略：编译期响应（compiler-driven）**，即布局在场景编译器阶段按目标分辨率/比例求解；**不做运行时响应式**。
- **上游策略：把 `esp-mosaico-claw` 官方仓库当上游（upstream），自己开一条分支，只做"板级适配 + UI 多形态适配"。**
- **分层原则：**
  - **跟随层（尽量不改、直接跟上游）**：板级框架、显示 HAL、服务层、`app_claw`、esp-claw。
  - **自研/分叉层（冻结为自有）**：`mosaic_ui` 的 UI 作者层（场景生成 + 响应式布局 + 多输入）。
- **esp-claw 本地二开（修 bug）与本方案正交**：把 submodule 指向本地 fork 即可。
- **目标硬件已定：ESP32-P4 + JC-ESP32P4-M3-DEV（板载 C6）+ JD9365 10.1″ DSI 无触控。** 调研确认：**Wi-Fi/DSI/音频/USB HID/SD/PPA/GSP 全部就绪**，无线走产品层已内置的 `esp_wifi_remote`+`esp_hosted`。
- **唯一硬功能缺失：蓝牙音频 A2DP**（板载 C6 无 Classic BR/EDR）→ **决议：去掉该 App**（非核心功能）。
- **Vibe Mode 整体去除**：自定义硬件没有在线市场/固件分享生态，其核心价值（Ideas 商店、在线固件、Bridge）无受众。回收约 1.96 MiB Flash，并解除"前 2 MiB 冻结 ABI + bootloader 归属"约束。
- **UI 调试改用 esp-gsp 官方 `sim_bridge`**（PC 交互测试，分辨率由 bundle 决定），并把自研 UI 约束在其已实现的 API 子集内。
- **调研结论**：mosaico 上层封装**大体与芯片解耦**，S31 锁定集中在**板级 + 启动 + Vibe Mode + 打包目标**，不在服务层。

---

## 1. 背景与仓库关系

`esp-mosaico-claw` 是 **ESP-MOSAICO 硬件的整机固件工程（factory firmware）**，project 名为 `edge_agent`（`VERSION 0.5.0`）。它不是新框架，而是：

> 以 **ESP-Claw 为 Agent 运行时内核**，向上做产品化封装（硬件 BSP + UI + 云服务 + 产品配置）的整机工程。

三者关系：

```
esp-mosaico-claw (本仓库, 产品整机)
  ├── third-party/esp-claw         (submodule, Agent 运行时内核 = 框架)
  └── third-party/esp-mosaico-utils(submodule, Vibe Mode 工厂/恢复镜像)
```

- README 原文："Built on ESP-Claw... also contains the source code for the ESP-MOSAICO factory firmware."
- 依赖方向：**单向 `mosaico → esp-claw`**。内核不反向依赖整机。

### 1.1 代码规模（决定"哪些值得复用"）

| 区域 | 行数 (C/H) | 性质 |
|---|---:|---|
| `components/mosaic_ui/` | ~32,400 | GSP 场景 UI（**最大资产，也是最大分叉点**） |
| `components/`（除 mosaic_ui） | ~27,800 | 服务层 + 板级/显示/网络/配置 |
| `components/` 合计 | ~60,200 | |
| `main/` | ~3,260 | 产品编排（wiring） |
| `boards/` | ~1,530 | 板级定义 |

---

## 2. esp-claw 的嵌入形式（六种）

记录以备查，避免重复调研：

1. **Git submodule**：`.gitmodules` 声明 `third-party/esp-claw`（`espressif/esp-claw`）与 `third-party/esp-mosaico-utils`。构建前需 `git submodule update --init --recursive`。
2. **Component Manager 路径依赖**：各组件 `idf_component.yml` 用 `path: ../../third-party/esp-claw/...` 引用内核组件。全仓合计 **~71 个** esp-claw 组件引用点（`claw_modules/*`、`claw_capabilities/*`、`common/*`、`lua_modules/*`）。
3. **EXTRA_COMPONENT_DIRS**：根 `CMakeLists.txt` 追加 `third-party/esp-claw/components/common/esp_video`。
4. **`components/app_claw` 门面封装（防腐层核心）**：产品侧通过 `app_claw.h` 的 `app_claw_start / update_config / get_core / apply_config` 等统一入口启动与配置内核；`app_claw.c` 按 `CONFIG_APP_CLAW_CAP_*` 条件编译串接 `claw_core / claw_memory / claw_skill / claw_event_router / claw_agent_mgr / cap_lua / cap_scheduler` 等。
5. **扩展点反向注册**（保证依赖方向单向）：
   - `app_capabilities_register_external_group()`（`components/app_claw/app_capabilities.c`）
   - `app_lua_modules_register_external()`（`components/app_claw/app_lua_modules.c`）
6. **构建期脚本 / Kconfig / 数据约定**：
   - esp-claw 提供 CMake 函数：`skill_builder_configure_skill_sync`、`lua_module_builder_configure_builtin_lua_sync`；Python 工具：`generate_board_resources.py`。
   - Kconfig 命名空间 `APP_CLAW_CAP_*` / `APP_CLAW_LUA_MODULE_*` / `APP_CLAW_LUA_DRIVER_*`（`components/app_claw/Kconfig`）。
   - 直接引用的内核符号：`CONFIG_CLAW_CORE_STAGE_VERBOSITY_VERBOSE`、`CONFIG_CLAW_CAP_SYSTEM_DEBUG_LOGS`。
   - 数据/资源约定：`claw_paths`、skills 目录、`.recovery` 布局、app/launcher 格式，烘焙进只读 SYSTEM littlefs 分区。

### 2.1 esp-claw 耦合度量化

- **集中**：真正的适配逻辑在 `app_claw.c / app_capabilities.c / app_lua_modules.c`。
- **泄漏**：另有 **~22 个文件**直接 `#include "claw_*/cap_*"`，分布见附录。
- **含义**：只要不破坏 `app_claw` 用到的结构体/Kconfig/CMake 函数，本地 esp-claw 改动对 mosaico 无感。

---

## 3. 分层现状与证据

### 3.1 板级（可移植性：好，但只验证过一块板）

- 使用 **Espressif 官方组件 `espressif/esp_board_manager`（^0.6）**，非自研 → 对我们是利好。
- **数据驱动**：`board_info.yaml` / `board_peripherals.yaml` / `board_devices.yaml` + 每板 `setup_*.c` 工厂入口（`lcd_panel_factory_entry_t`、`lcd_touch_factory_entry_t`、`lcd_panel_set_brightness_entry_t`）；`idf.py bmgr -c ./boards -b <board>` 生成 `gen_bmgr_codes`。
- **板子硬编码很少**：`CONFIG_ESP_BOARD_ESP_MOSAICO` 仅 **9 处**；主要按**设备能力宏** `CONFIG_ESP_BOARD_DEV_*`（`DISPLAY_LCD`、`LCD_TOUCH`、`AUDIO_CODEC`、`CAMERA`…）分支 → 这是可移植的正确写法。
- **风险**：`boards/` 只有 `esp_mosaico` 一块板，**多板路径从未被验证**。我们是"第二个板级用户"，边界暗坑需自趟。

### 3.2 显示 HAL（可移植性：好）

`components/display_service`（~3,300 行）是合格 HAL：

- 接口枚举已含 `SPI / I80 / RGB / DSI / PARLIO`（`display_service.h`）。
- 底座 `espressif/esp_display_present` + `components/esp_lvgl_present`（LVGL v9 runtime）。
- 上报 `width/height/bits_per_pixel/panel_interface/touch_available`；提供亮度/旋转 provider；支持 `SHARED_LVGL / EXCLUSIVE_LVGL / EXCLUSIVE_RAW` 会话。
- **换屏/换接口/无触控，理论上只改板级 YAML + factory 入口。**

### 3.3 UI（可移植性：**差，核心成本所在**）

`components/mosaic_ui`（~32,400 行）**不是普通 LVGL，而是 GSP 场景体系**：

```
Python 场景生成器(本仓库) → 闭源预编译 GSP 编译器 gspc(esp-gsp-tools 下载) → *.gspb bundle → GSP 运行时(esp-gsp 组件)
```

- **依赖 PPA**：`CMakeLists.txt` `REQUIRES ... esp_driver_jpeg esp_driver_ppa`；`gsp_enable_freetype()`。PPA 仅在 P4/S31 这一代存在 → 换到无 PPA 芯片则整个 mosaic_ui 作废（本案目标芯片 P4/S31+，故不构成否决项）。
- **绝对坐标 + 480×480 硬编码**：
  - `components/mosaic_ui/common/scene_common.py`：`SCREEN_W=480 / SCREEN_H=480 / CONTENT_W=480 / CONTENT_H=480`，所有 widget 用绝对 `x,y,w,h`（如状态栏电池写死 `x=420`）。
  - 规模：**15 个 `apps/*/scene/gen_scene.py` + `hub/scene/gen_scenes.py`，生成器约 5.5k 行**；`apps/` 目录共 ~15,700 行。
- **场景 profile 只管像素格式**：`common/mosaic_rgb565_auto.yaml`（`pixel_format/alignment/jpeg`），**不管布局**；bundle 名直接叫 `hub_480.json`。
- **唯一响应式先例**：`apps/welcome/scene/gen_scene.py` 用 `MOSAIC_WELCOME_WIDTH` 做宽度参数化（`offset=(width-CONTENT_WIDTH)//2`）——可作改造模板。
- 重外部依赖：`espressif/esp-gsp`、`esp_mmap_assets`、`tda-2030/esp_vision_core`、`esp_bt_audio`、`mosaico_module_interact`（来自 `esp-mosaico/esp-mosaico-bsp` 的 git 依赖）。

**结论**：响应式不是"改几个数字"，而是**先造布局抽象（scale/anchor/flex），再重写全部场景生成器**。

### 3.4 输入（可移植性：中，存在扩展点）

- **通用输入只有 pointer**：`esp_gsp_set_pointer_observer` → `on_gsp_pointer` → `post_pointer(x,y,pressed)`（`core/mosaic_esp_platform.c`，含 12px 采样门限）。
- **存在输入拦截扩展点**：`esp_gsp_set_input_interceptor`（`common/mosaic_app_shell.c`）。
- **存在键盘 widget API**：`esp_gsp_keyboard_attach / esp_gsp_keyboard_text`（settings / setup_center 密码输入在用）——是文本输入控件，**不是方向键导航/焦点模型**。
- **GPIO 按钮已有语义 action 路径**：`main/platform/mosaic_button_platform.c` 把按钮映射为 `mosaic_ui_back()` / `mosaic_ui_open_ai_create()`（走语义动作，不是合成坐标）。
- **无 USB host / HID**：`boards/esp_mosaico/components/usb_hs_console` 是 device 模式 TinyUSB 控制台；USB HID **host** 为全新代码。

**难度分级**：GPIO 按钮 = 易；HID 鼠标（合成 pointer）= 中；HID 键盘/方向键导航（需自建 focus 模型）= 中偏上。

### 3.5 构建/资源胶水层（换板会打架）

- `tools/cmake/esp_idf_patch.cmake`：**直接改 ESP-IDF 源码**（LCD/parlio、USB 描述符、HTTP client）。换 IDF 版本会重新踩。
- `tools/cmake/retained_recovery.cmake` + `partitions_16MB.csv` + esp-mosaico-utils 的 **Vibe Mode 预编译 bundle**：固定分区表/固定 `vibe_mode` 分区/固定 flash 布局，是"产品固化恢复系统"，非通用能力。
- `components/mosaico_boot_splash`：**bootloader 内预初始化 CO5300 并画开机 logo**，与 QSPI 屏强绑定。
- `fatfs_image/system/`：烘焙的 skills/apps/字体，混通用与产品专属内容。

---

## 4. 目标硬件与诉求（本次决策输入）

- **目标芯片**：ESP32-P4、ESP32-S31 及未来同代高性能模组（**确认 PPA 可用**）。旧的 S3 不列为目标（跑 esp-claw 已吃力，再叠产品层性能不足）。
- **当前目标板（已定）**：**JC-ESP32P4-M3-DEV** —— ESP32-P4 + **板载 ESP32-C6**（Wi-Fi 6/BLE，SDIO + ESP-Hosted）+ **32 MB PSRAM** + **16 MB NOR** + **MIPI-DSI 2-lane** + **MIPI-CSI** + **ES8311 音频（模拟麦 + NS4150 功放）** + TF 卡 + **HS/FS 双 USB-OTG** + 以太网。详见 `docs/board-jc-esp32p4-m3-dev.md`。
- **现有硬件（已实测）**：**JD9365 10.1″ DSI 屏，无触控**；音频可录可放；USB HID（2.4G 键鼠）可枚举；Wi-Fi / SD 正常。
- **UI 诉求**：把 mosaic 的**圆屏特例化交互**改为**响应式**，支持各种比例/分辨率。
- **输入诉求**：同时支持**触控**与**非触控**（GPIO 按钮 S1–S4、USB HID 键鼠）。
- **供电**：USB 供电（板载 IP5306 但无电量计，不做电池策略）。
- **esp-claw 诉求**：已有本地二开 fork 用于修复内核问题。

---

## 5. 最终决议方案

### 5.1 总体架构

**`mosaic 服务层 + 自研响应式 UI`**，即保留 mosaico 的集成与编排资产，**替换 UI 作者层**。

```
┌──────────────────────────────────────────────────────────────┐
│ 自研响应式 UI (GSP 场景层)                                     │  ← 分叉/自研层
│   编译期响应式布局库 (scale/anchor/flex) + 多输入适配 + 各 app   │
├──────────────────────────────────────────────────────────────┤
│ mosaic 服务层 (跟随上游)                                        │  ← 跟随层
│   display_service / esp_lvgl_present / http_server /           │
│   wifi_manager / app_config / app_settings_service /           │
│   asr_service / weather_service / update_check_service / ...   │
├──────────────────────────────────────────────────────────────┤
│ app_claw (防腐层, 跟随上游)  +  board layer (跟随上游)          │  ← 跟随层
├──────────────────────────────────────────────────────────────┤
│ third-party/esp-claw (submodule → 指向本地 fork)               │  ← 内核
│ third-party/esp-mosaico-utils (Vibe Mode, 本方案基本不用)       │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 UI 路线：自写 GSP 场景层

- **动机**：复用 PPA 硬件加速与 `.gspb` bundle 体系，保证流畅度。
- **做法**：保留 GSP 运行时 + `gspc` 编译链，**在 Python 场景作者层自建响应式布局能力**，并按新形态编写/改造场景。
- **不做**：不改造 GSP 引擎（闭源、不在本仓库、无法改），也不做运行时缩放/重排。

### 5.3 响应式策略：编译期响应（compiler-driven）

- 布局在**场景编译阶段**按目标分辨率/宽高比/DPI 求解，输出针对该形态的 `.gspb`。
- 理由：PPA 合成与 GSP 场景模型基于绝对几何 → 编译期求解可保持零运行时开销与最佳流畅度。
- 落地方式（候选，spike 中细化）：
  - 在 `scene_common.py` 之上引入**布局 DSL**：以"设计基准（design tokens）+ 锚点/容器/弹性规则"描述，生成器按目标 `(W,H,ppi,safe-area)` 求绝对坐标。
  - 保留一份"设计基准分辨率"，其余分辨率/比例全部由规则推导。
  - 参照 `welcome` 的 `MOSAIC_WELCOME_WIDTH` 参数化模式作为迁移起点。
- **多形态产物**：同一逻辑场景可按形态生成多个 bundle 变体，构建期按所选板/屏 profile 选取。

### 5.4 输入归一化层

在 `display_service` 之上新增**输入归一化层**，把多来源输入统一为三类事件再喂给 UI：

| 输入源 | 归一化为 |
|---|---|
| 触控（若有） | `pointer(x,y,pressed)` |
| GPIO 按钮 | `action(name)`（复用现有语义 action 路径） |
| USB HID 鼠标 | `pointer(x,y,pressed)`（合成虚拟光标） |
| USB HID 键盘 / 遥控方向键 | `key(code)` + `focus` 导航（**需自建，GSP 无原生场景焦点模型**） |

- 复用点：`esp_gsp_set_input_interceptor`、`mosaic_button_platform.c` 的语义 action 先例。
- 新增点：**focus/nav 模型**（方向键在 UI 元素间移动 + 确认键触发 `click`），USB `usb_host` + `hid_host`。

### 5.5 分层实施边界（保留 vs 自研）

| 组件/区域 | 处置 | 说明 |
|---|---|---|
| `boards/<new>/`（YAML + setup_*.c） | **新增** | 板级定义，天然少冲突 |
| `espressif/esp_board_manager` | 跟随 | 官方组件 |
| `components/display_service` | 跟随(+少量定制) | DSI/分辨率/旋转/亮度；TE 参数按新板覆写 |
| `components/esp_lvgl_present` | 跟随 | LVGL presenter |
| `components/http_server` / `wifi_manager` / `app_config` / `app_settings_service` / `asr_service` / `weather_service` / `update_check_service` / `wechat_binding_service` | 跟随 | 服务层 |
| `components/app_claw` | 跟随(+必要适配) | 防腐层；只在 esp-claw API 漂移时改 |
| `components/cap_system` / `cap_im_ai_create` | 跟随(本地覆盖) | 注意本地分叉需人工同步上游 |
| `third-party/esp-claw` | submodule → 本地 fork | 独立迭代 |
| `components/mosaic_ui`（场景作者层/apps） | **自研替代** | 响应式布局 + 新形态场景 |
| `components/mosaic_ui/apps/bluetooth` | **去除** | A2DP(Classic) 依赖；板载 C6 无 BR/EDR，P4 上不可用（见 `docs/p4-vs-s31-feasibility.md` §7.2） |
| `components/mosaico_boot_splash` | **去除** | CO5300/QSPI 专属 bootloader 手递手；换 DSI 屏后无意义 |
| `third-party/esp-mosaico-utils` + `retained_recovery.cmake` + `partitions_16MB.csv` | **整体去除** | Vibe Mode 无在线生态受众（见 §5.8）；回收约 1.96 MiB Flash，解除前 2 MiB 冻结 ABI 与 bootloader 归属 |
| `tools/cmake/esp_idf_patch.cmake` | 视需要 | 改 IDF 源码；换 IDF 版本需复检 |

### 5.6 上游协作与分支工作流

- **上游**：`esp-mosaico/esp-mosaico-claw` 作为 `upstream` remote；`third-party/esp-claw` submodule 指向**本地 esp-claw fork**。
- **自研分支**：从官方 `master` 切一条长期分支，**只做板级适配 + UI 多形态适配**，不掺入无关产品改动。
- **冲突面控制**：
  - 优先**新增**（`boards/<new>/`、自研 UI 目录、新的独立组件），避免编辑上游高频共享文件。
  - 维护一份 **"被迫修改的共享文件清单"**（`main/main.c` 条件、根 `CMakeLists.txt`、`sdkconfig.defaults`、`tools/cmake/*` 等）；每次同步上游只重点 review 这些。
  - esp-claw 侧改动**尽量收敛到 `app_claw` 不依赖的部分**，减少对 mosaico 的连带影响。
- **预期**：UI 层与上游 UI 全面分叉（接受）；服务层/板级/集成层保持可跟随（收益来源）。

### 5.7 动手前调研闭环（本轮结论）

调研产出三份配套文档，结论如下：

| 调研项 | 结论 | 文档 |
|---|---|---|
| **Vibe Mode / Ideas 是什么** | 不是组件，而是藏在 `esp-mosaico-utils` 里的**第二个完整固件**（预编译包），以 Recovery ABI 共享 Flash 前 2 MiB，作为刷不坏的维护固件 | `docs/vibe-mode-and-ideas-analysis.md` |
| **P4 能否承载本方案** | **能**。PPA/GSP/2D-DMA 齐全，`esp-gsp` 官方目标含 `esp32p4` 且有 P4+MIPI-DSI benchmark；P4 原生 DSI，**比 S31 更适合大屏** | `docs/p4-vs-s31-feasibility.md` |
| **mosaico 上层封装能否移植** | **能**。服务层大体与芯片解耦；`wifi_manager` 已内置 `esp32p4` 分支（`esp_wifi_remote`+`esp_hosted`），`display_service` 已有真实 DSI 处理，`esp_lvgl_present` 已有 MIPI 配置宏。S31 锁定只在**板级 + 启动 + Vibe Mode + 打包目标** | 同上 §2.7 |
| **目标板能力** | **JC-ESP32P4-M3-DEV 已实测**：Wi-Fi、JD9365 DSI、音频录放、USB HID、SD 全部正常；另有以太网；**无传感器/电量计/NAND** | `docs/board-jc-esp32p4-m3-dev.md` |
| **唯一硬缺失** | **蓝牙音频 A2DP**（板载 C6 无 Classic BR/EDR）→ **决议：去掉该 App** | 同上 §7.2 |

**由此固化的决议：**
1. 目标板锁定 **JC-ESP32P4-M3-DEV**（P4 + 板载 C6 + 32 MB PSRAM + 16 MB NOR + DSI）。
2. 无线**不新增架构**，启用产品层已就绪的 `esp_wifi_remote` + `esp_hosted`。
3. **删除** `mosaic_ui/apps/bluetooth`（A2DP 不可用，且非核心）。
4. **去除** boot splash（CO5300/QSPI 专属，且 bootloader 分支本就是死代码）。
5. 功能裁剪：无 IMU/磁力计/电量计 → 裁 `app_claw_hw_bridge` 资源表；供电按 USB 固定，不做电池策略。
6. **Vibe Mode 整体去除**（见 §5.8）。

### 5.8 Vibe Mode 决议：整体去除

**触发点**：自定义硬件**没有在线市场、没有用户分享固件的生态**。Vibe Mode 的核心价值（Ideas 在线应用商店、在线固件清单 `dl.espressif.com/mosaico/`、HTTPS 远程维护 Bridge）在本产品上**不存在受众**。

**决议**：**整体去除**，不保留运行时，也不做"自建维护固件"。

**回收的硬约束**（此前都被视为"与 UI 同级的冻结层"）：

| 约束 | 去除后 |
|---|---|
| 前 2 MiB 冻结 ABI（5 个不可变分区） | **解除**，分区表可自由排布 |
| Flash 空间 | **回收约 1.96 MiB**（`vibe_mode` 1.75 MiB + `sysmeta` 80 KiB） |
| bootloader 归属（产品烧 bundle 的 bootloader） | **解除**，用自己构建的 bootloader |
| bootloader 24 KiB 容量上限 + CO5300 Logo | **解除**（换 DSI 屏不再需要回 utils 改 bootloader） |
| `flash`/`update-flash` 被 `retained_recovery.cmake` 接管 | **解除**（该文件 118 行整体删除） |
| S31 专属预编译包 | **解除**，不再需要为 P4 重建 Vibe Mode |

**产品侧改动点（已全量核实，范围极小）**：

| 位置 | 处置 |
|---|---|
| `tools/cmake/retained_recovery.cmake`（118 行） | 删除 |
| `CMakeLists.txt:107` 的 `include(...)` | 删除该行 |
| `components/app_claw/app_claw_cli.c` 的 `request_vibe_mode`/`cmd_vibe`（约 101 行） | 删除 |
| `components/app_claw/CMakeLists.txt:52` 的 utils include | 删除 |
| `partitions_16MB.csv` | 重写 |
| `tools/package_firmware.py:19` `TARGET = "esp32s31"` | 改 `esp32p4` |
| `third-party/esp-mosaico-utils` 子模块 | **可整体移除**（无其它引用） |

**为什么不是"裁剪"而是"整体去除"**：Vibe Mode 的 12 个组件中，`app_ideas`（在线商店）、`app_update_check`（在线清单）、`app_bridge`（HTTPS Bridge）、`app_network`（仅服务上述）都属在线生态；`app_nand_update`/`app_storage` 依赖 NAND（**本板无**）；`app_settings`/`app_device` 依赖电量计+IMU（**本板无**）；`app_ui` 是 Vibe Mode 自己的 480×480 `vui` UI（**不是** GSP UI）。剩下的只有 `app_system` + USB 更新路径，已不足以支撑保留一个固件。

**⚠️ 两类"在线"必须区分**：
- **删**：*固件/应用分发生态*（Ideas 商店、在线固件清单、Bridge）。
- **留**：*产品自身的 AI 服务端点*——`trial_provider`、`weather_service`、`update_check_service`、`wechat_binding_service`。这些是产品功能，不是生态。

**唯一保留的是"主机交互测试"能力，但不是 Vibe Mode 的模拟器。**

Vibe Mode 的 `pc/` 模拟器跑的是 `vui.c`（自绘 480×480 RGB565，`simulator.py` 中 `480` 硬编码于画布/像素缓冲/坐标映射三处），与 GSP 场景层是两套栈，**测不了新 UI**。正确做法是用 **esp-gsp 官方 `sim_bridge`**（`tools/sim_bridge/`，自 1.2.0 起随组件发布）：它把可移植 C UI 编译成本机动态库，用 `sim_host` 渲染到浏览器，支持指针注入、截图断言、自动化会话，且**分辨率由 bundle 决定**，800×1280 可直接预览。

**一条重要的设计约束（实测得出）**：`mosaic_ui` 用到的 65 个 `esp_gsp_*` API 中，`sim_bridge` 已实现 36 个、内联 3 个、设备专属 8 个（16 个调用点全部集中在 `core/mosaic_esp_platform.c` 单文件），**真缺失 18 个**，其中包括 `stack_view_*`（41 处/6 文件）与 `overlay_contributor`（16 处/1 文件）这类承载性能力。

因此：

> **自研 UI 应约束在 `sim_bridge` 已实现的 API 子集内** → 天然获得 PC 交互测试能力。

对应替换方案：导航用 `goto_scene` + 显式页面状态机（不用 `stack_view`）；顶栏/角标用场景内普通组件（不用 `overlay`）；动画用 `component_animate_property`（不用 `animate_position_to`）；输入用 `on_event` + 自有归一化层（不用 `input_interceptor`）。代价是放弃 stack_view 转场与 overlay 的 PPA 合成层，收益是**改 UI 不必烧录**，且倒逼 UI 与设备解耦。详见 `docs/vibe-mode-and-ideas-analysis.md` §11.1–§12。

---

## 6. 关键风险与未知

| 风险 | 影响 | 缓解 |
|---|---|---|
| GSP 在非 480/大分辨率（如 1280×800）下的**渲染性能/内存/带宽**未知 | 可能被迫换 UI 框架 | **spike 先实测**（见 §7） |
| PPA 对大分辨率 RGB565 的填充/带宽 | 帧率不足 | 实测；必要时降色深/分区刷新 |
| `mosaic_ui` 场景层与上游 UI **全面分叉** | 上游 UI 更新无法直接跟 | 接受；靠服务层收益补偿 |
| USB HID host 全新代码 | 工期不确定 | 独立模块、可后置 |
| GSP 无原生 focus/nav 模型 | 键鼠/遥控导航需自建 | 自建输入归一化层 |
| boot splash（CO5300/QSPI 专属）耦合 bootloader | 换屏需改 bootloader 逻辑 | spike 中先跑通"无 splash"基线 |
| 多板路径**从未被上游验证** | 边界暗坑 | 我们从最小板开始，逐步加回设备 |
| 无触控 + 大屏的交互设计缺失 | UX 不完整 | 与输入方案同步设计 |
| ~~P4 无 Wi-Fi~~ | **已消除** | 板载 C6 + 产品层已内置 `esp_wifi_remote`/`esp_hosted` |
| **蓝牙音频 A2DP 不可用** | 失去"蓝牙音箱"App | **已决议去除该 App**（非核心） |
| **无 IMU/磁力计/电量计** | 部分产品功能需裁剪 | 裁 `app_claw_hw_bridge` 资源表；可选外接 I2C |

---

## 7. Spike 验证清单（开工第一步，1–2 天可出结论）

> **前置已闭合**（本轮调研完成，无需再验）：Wi-Fi/esp_hosted、DSI 显示、音频录放、USB HID、SD、PPA/GSP 可用性、产品层 P4 就绪度、蓝牙 A2DP 不可用（已决议去除）。
> 因此 Spike 只保留**真正未知**的三项。

1. **GSP 非 480 实测**：在 800×1280 跑一个 GSP 场景，测帧率/内存/带宽 → 决定是否坚持 GSP（唯一可能推翻 UI 路线的点）。
   - **可先在 PC 上做一半**：`sim_bridge` 能按 800×1280 的 bundle 渲染，先验证布局/交互/文字排版；帧率/带宽仍必须上板实测（模拟器明确声明不替代设备时序验证）。
   - **PC 侧已验证（2026-10-19）**：800×1280 竖屏场景编译渲染正常；`tap` 注入、`screenshot`、`frame_info` 脏矩形、后端定时数据驱动均可用；AI 可直接读 PNG 完成闭环。命令与结论见 `dev-env-and-pc-sim-handoff.md`。**剩余只有真机性能一项。**
2. **响应式单点改造**：照 `welcome` 的 width 参数化改一个 app（如 `settings`），度量"单 app 改造行数/耗时" → ×15 估总量。
3. **冲突面盘点**：记录被迫编辑的共享文件清单 = 真实合并冲突面。

**Go/No-Go 判定**：
- 若第 1 项性能达标且第 3 项冲突面可控 → 按本方案执行。
- 若第 1 项不达标 → 回退到"服务层 + LVGL 自研 UI"（放弃 GSP/PPA 合成），其余不变。
- 若第 3 项冲突面过大（大量深改共享文件）→ 调整为"**抽取组件自建工程**"：vendor 服务层组件 + esp-claw submodule，不整仓跟踪。

---

## 8. 待决问题（开工时确认）

**已关闭（本轮调研结论）：**
- [x] ~~目标屏具体分辨率/色深/接口时序~~ → **JD9365 10.1″，800×1280，DSI 2-lane，无触控**（已实测点亮）。
- [x] ~~USB HID host 栈选型~~ → P4 原生 `usb_host` + `hid_host`，HS OTG 口，已实测 2.4G 键鼠。
- [x] ~~P4 无线方案~~ → **板载 C6（SDIO + ESP-Hosted）**；产品层 `wifi_manager` 已内置 `target in [esp32p4]` 分支。
- [x] ~~蓝牙音频~~ → **C6 无 Classic BR/EDR，A2DP 不可用 → 决议去除 `mosaic_ui/apps/bluetooth`**。
- [x] ~~boot splash~~ → 换 P4+DSI 后 **直接去除**（原实现为 CO5300/QSPI + `LP_STORE15` 手递手，且其 bootloader 分支本就是死代码）。
- [x] ~~Vibe Mode 去留~~ → **整体去除**（无在线生态受众）；只保留"主机交互测试"能力，且改用 **esp-gsp 官方 `sim_bridge`**（详见 §5.8）。

**仍待决：**
- [ ] 响应式布局 DSL 的具体形式（设计基准 + scale 规则 vs 容器/锚点模型）。
- [ ] focus/nav 交互模型细节（方向键移动、确认、返回、滚动）。
- [ ] 自研 UI 目录的组织与上游 `mosaic_ui` 的共存/替换方式（是否物理移除）。
- [ ] 无传感器/电量计/NAND 后的**功能裁剪范围**（`app_claw_hw_bridge` 资源表、相机是否启用）。

---

## 附录 A：证据索引（速查）

| 主题 | 位置 |
|---|---|
| submodule 声明 | `.gitmodules` |
| project 名/版本 | `CMakeLists.txt`（`project(edge_agent VERSION 0.5.0)`） |
| app_claw 门面 | `components/app_claw/include/app_claw.h`、`components/app_claw/app_claw.c` |
| 能力扩展点 | `components/app_claw/app_capabilities.c`（`app_capabilities_register_external_group`） |
| Lua 扩展点 | `components/app_claw/app_lua_modules.c`（`app_lua_modules_register_external`） |
| 硬件桥 | `components/app_claw/app_claw_hw_bridge.c`（`claw_hw_registry` ↔ board manager） |
| 能力/模块开关 | `components/app_claw/Kconfig` |
| 板级定义 | `boards/esp_mosaico/{board_info,board_peripherals,board_devices}.yaml`、`boards/esp_mosaico/setup_device.c` |
| 显示 HAL | `components/display_service/include/display_service.h`、`src/display_service_target.c` |
| UI 场景体系 | `components/mosaic_ui/CMakeLists.txt`、`common/mosaic_scene_build.cmake`、`common/scene_common.py`、`tools/generate_bundle.py` |
| 响应式唯一先例 | `components/mosaic_ui/apps/welcome/scene/gen_scene.py`（`MOSAIC_WELCOME_WIDTH`） |
| 输入通路 | `components/mosaic_ui/core/mosaic_esp_platform.c`（`esp_gsp_set_pointer_observer`）、`core/mosaic_loader.c`（`post_pointer`） |
| GPIO 按钮 → 语义 action | `main/platform/mosaic_button_platform.c` |
| 输入拦截扩展点 | `components/mosaic_ui/common/mosaic_app_shell.c`（`esp_gsp_set_input_interceptor`） |
| GSP UI 开关 | `main/Kconfig.projbuild`（`APP_CLAW_MOSAIC_GSP_ENABLE`） |
| 构建期 IDF 补丁 | `tools/cmake/esp_idf_patch.cmake` |
| Vibe Mode/恢复固化 | `tools/cmake/retained_recovery.cmake`、`partitions_16MB.csv` |
| boot splash | `components/mosaico_boot_splash/` |
| 产品数据/资源 | `fatfs_image/system/` |
| 工具链/PC 模拟器实测 | `docs/dev-env-and-pc-sim-handoff.md` |
| 设备侧日志接口 | `components/http_server/http_server_logs_api.c`（`WS /ws/logs`、`GET /api/logs/status`） |
| 设备侧截图 | **不存在**；需新增 `/api/screenshot`（见 handoff §5.2） |

## 附录 B：直接 include `claw_*/cap_*` 的文件（耦合泄漏点，~22 个）

```
components/app_claw/app_capabilities.c
components/app_claw/app_claw_cli.c
components/app_claw/app_claw_cli.h
components/app_claw/app_claw.c
components/app_claw/include/app_claw_hw_bridge.h
components/app_claw/include/app_claw.h
components/cap_im_ai_create/src/cap_im_ai_create.c
components/cap_system/src/cap_system_platform.c
components/cap_system/src/cap_system.c
components/http_server/http_server_webim_api.c
components/http_server/http_server_webim_sessions.c
components/mosaic_ui/apps/album/album_app.c
components/mosaic_ui/apps/camera/camera_app.c
components/mosaic_ui/apps/music/music_controller.c
components/mosaic_ui/common/ai_create_controller.c
components/mosaic_ui/common/ai_create_controller.h
components/mosaic_ui/common/ai_create_session_history.c
components/mosaic_ui/common/works_runtime.c
components/mosaic_ui/core/mosaic_esp_platform.c
components/wechat_binding_service/wechat_binding_service.c
main/main.c
main/platform/mosaic_settings_platform.c
```

> 注：`mosaic_ui/*` 的若干引用属于"UI 与内核交互"的合理耦合；若未来希望进一步降低耦合（提高可跟随性），可考虑把这些交互收敛到 `app_claw` 暴露的接口。

## 附录 C：上游"sync"提交（说明 co-evolution 是常态）

```
0d15044 feat(app-store): add on-device app management
22390cb fix(lua): sync sensor integration and module fixes
217e697 refactor(imu): share native board sensor with Lua
31e4e4a refactor(app-claw): sync hardware resource registry
8cd9d5f feat(web-chat): integrate unified conversations
ee1d446 feat(search): sync Bocha support to Mosaico
9e4b8c2 feat(board): enable IR and ADC Lua support
```

含义：Mosaico 需求（外设/传感器/搜索/IM）常**先落 esp-claw 再 bump 进来**；外设/能力类需求可能要回到 esp-claw 做配套。这与"服务层跟随上游"的收益一致。

---

*本文档为一次性分析快照 + 决策记录。开工时以 §7 Spike 与 §8 待决问题为起点。*
