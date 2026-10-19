# ESP-MOSAICO Vibe Mode / Ideas 固件实现分析

> 状态：分析快照（只读勘察，未改动任何产物）
> 分析对象：`third-party/esp-mosaico-utils` 子模块 @ `ca2f9c1eae4d95cd11ca49487949a8f5e7a39f9c`
> 预编译包：`version 0.2.0` · `Recovery ABI 2` · `layout 5` · `chip esp32s31` · `board esp-mosaico`
> 镜像尺寸：`bootloader.bin` 21,984 B · `partition-table.bin` 3,072 B · `ota_data_initial.bin` 8,192 B · `vibe-mode.bin` 1,488,608 B（槽位 1.75 MiB）
> 关联文档：`docs/mosaico-fork-analysis-and-decision.md`、`docs/p4-vs-s31-feasibility.md`

---

## 0. 一句话结论

Vibe Mode **不是一个组件，而是藏在 `esp-mosaico-utils` 里的第二个完整固件工程**（工程名 `vibe-mode`）。它以**预编译二进制包**形式被 `esp-mosaico-claw` 在构建期拼进 Flash，作为**永久驻留的维护/恢复固件**，与主应用通过一套冻结的 **Recovery ABI** 契约共享 Flash 头部 2 MiB。"Ideas" 是 Vibe Mode 内"下载并运行小应用"的子系统。

---

## 1. 为什么需要两个固件

这是**单应用槽（单副本，无 A/B 回滚）**设备：主应用槽可能被写坏，因此必须有一份**不可被主应用覆盖**的维护镜像常驻 Flash 作为兜底。

```
┌──────────────────────────── Flash 前 2 MiB（冻结系统前缀）────────────────────────────┐
│ otadata │ phy_init │ sysmeta │        vibe_mode (保留 Vibe Mode)        │ coredump │
└────────────────────────────────────────────────────────────────────────────────────┘
┌────────────────────────── 0x200000 之后（可变应用区）──────────────────────────┐
│ nvs │ main_app (主应用 = ESP-Claw / 本仓库) │ ui_apps │ system (littlefs) │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 产物与源码位置

```
third-party/esp-mosaico-utils/esp-mosaico-vibe-mode/
├── include/mosaico_recovery_contract.h        ← 双方共享 ABI 宏（产品侧编译时引用）
├── product_contract.json                      ← 机器可校验产品契约
├── docs/                                      ← 软件规格 / 迁移 / 工作流 / 在线更新
├── firmware/vibe-mode/
│   ├── bootloader_components/main/            ← ★ 自定义二级 bootloader（决定启动谁）
│   │   ├── bootloader_start.c
│   │   ├── mosaico_boot_nvs.c                 ← 私有 NVS 读取器（读启动意图）
│   │   └── mosaico_boot_splash.c              ← 启动 Logo + 屏幕交接
│   ├── components/app_*                       ← app_ui / app_ideas / app_bridge / app_system_update …
│   ├── tools/prepare_recovery.py              ← 打包 + 校验 + 原子落盘
│   ├── tools/boot_intent.py                   ← 生成带标记的 otadata
│   └── prebuilt/recovery/                     ← ★ 被 mosaico-claw 消费的成品包
│       ├── bootloader.bin   partition-table.bin
│       ├── ota_data_initial.bin   vibe-mode.bin   manifest.json
└── (ESP-Iris 0.2 框架位于同级 ESP-Iris/，产品协议栈)
```

---

## 3. Recovery ABI：Flash 头部固定契约

常量定义在 `include/mosaico_recovery_contract.h`：

```c
#define MOSAICO_RECOVERY_ABI 2U
#define MOSAICO_LAYOUT_VERSION 5U
#define MOSAICO_RECOVERY_PARTITION "vibe_mode"     /* app/test */
#define MOSAICO_RECOVERY_ADDRESS   0x20000U
#define MOSAICO_RECOVERY_BYTES     0x1c0000U       /* 1.75 MiB */
#define MOSAICO_SYSMETA_PARTITION  "sysmeta"
#define MOSAICO_BOOT_NAMESPACE     "mosaico_boot_v2"
#define MOSAICO_BOOT_INTENT_KEY    "intent"
#define MOSAICO_BOOT_INTENT_RECOVERY 0x02000001U
```

分区布局（`partitions_16MB.csv` + `product_contract.json`）：

| 分区 | Offset | Size | 类型 | 用途 | ABI |
|---|---:|---:|---|---|---|
| `otadata` | `0x9000` | 8 K | data/ota | OTA 选择/回滚 | **不可变** |
| `phy_init` | `0xb000` | 4 K | data/phy | PHY 数据 | **不可变** |
| `sysmeta` | `0xc000` | 80 K | data/nvs | 系统专用 NVS | **不可变** |
| `vibe_mode` | `0x20000` | 1.75 MiB | app/test | **保留 Vibe Mode** | **不可变** |
| `coredump` | `0x1e0000` | 128 K | data/coredump | 崩溃证据 | **不可变** |
| — | **`0x200000`** | | | **↓ 应用区起点** | 可变 |
| `nvs` | `0x200000` | 24 K | data/nvs | 应用 NVS | 可变 |
| `main_app` | `0x210000` | 8000 K | app/ota_0 | 主应用 | 可变 |
| `ui_apps` | `0x9e0000` | 3200 K | data/0x41 | GSP UI 资源 | Claw 私有 |
| `system` | `0xd00000` | 3000 K | data/littlefs | 系统资源 | Claw 私有 |

`sysmeta` 内的命名空间：
- `mosaico_boot_v2` / `intent`（u32）— 启动意图
- `iris_ota_demo` — Vibe Mode 自身 OTA 状态
- `update` / `last_result` — 最后一次系统更新结果（64 字节记录，magic `0x49535953`）
- `ideas` — 开机应用身份
- `wifi` — Factory Wi-Fi 凭据
- `esp_iris` — TCP 配对 token

> ABI 只强校验**前 5 个分区**的名字/类型/子类型/offset/size/flags；`nvs`/`ota_0`/`ui_apps`/`system` 由各自固件的分区表自由定义。

---

## 4. 启动选择逻辑（核心）

启动执行的是 **Vibe Mode 包里的自定义二级 bootloader**（`bootloader_start.c`，替换 IDF 的 `call_start_cpu0`，但保留 IDF 的 OTA/回滚/测试分区规则）：

```
call_start_cpu0()
 ├─ 1. stop_inherited_usb_dma()      # S31 专属：清掉上次固件遗留的 USB DMA（panic/watchdog 路径）
 ├─ 2. bootloader_init()
 ├─ 3. mosaico_boot_splash_show()    # 画 Logo(CO5300/SPI2) + 发 LP_STORE15 屏幕交接标记
 └─ 4. select_partition_number()
        ├─ 载入分区表
        ├─ 若以下任一为真 → 启动 TEST_APP_INDEX(=vibe_mode)，且不改动 otadata：
        │    (a) factory_recovery_requested()    # GPIO7(AI键) 上电持续低电平
        │    (b) bootstrap_recovery_requested()  # otadata 偏移 32 = "MOSAICO-BOOT-02!"(16B)
        │    (c) software_recovery_requested()   # sysmeta:mosaico_boot_v2:intent == 0x02000001
        ├─ 否则走 IDF 标准 selected_boot_partition()
        ├─ 兜底：正常选择无效但 vibe_mode 分区符合 ABI → 仍启动 Vibe Mode
        └─ 读意图失败/类型不符 → 一律优先 Vibe Mode（fail-safe）
```

**设计意图**：任何"不确定"，都进维护固件（维护固件永远比主应用更可信可用）。

---

## 5. 状态机：三种启动意图与转换

```
        ┌───────────── 正常应用 (main_app / app ota_0) ─────────────┐
        │                                                           │
 ①上电时GPIO7低  ②otadata bootstrap标记      ③运行中写 NVS intent=0x02000001
        │                    │                          │
        ▼                    ▼                          ▼
   ┌──────────────────── Vibe Mode (vibe_mode / app test) ────────────────────┐
   │ 启动后先"消费意图"：intent→0；若来自②则擦除 otadata 首扇区（保留第二扇区）│
   │ 提供：Download Ideas / NAND update / USB / HTTPS Bridge / Wi-Fi / UI…     │
   └──────────────────────────────┬───────────────────────────────────────────┘
                                  │ IDF 校验并选择 OTA 分区 → 清除软件意图 → 重启
                                  ▼
                           回到正常应用
```

| 触发器 | 场景 | 写入者 |
|---|---|---|
| **GPIO7 低电平** | 上电物理强制进维护 | 用户（AI 键） |
| **otadata bootstrap 标记** | **工厂/recover 安装**：即使已有可用应用也先强制进 Vibe Mode | `tools/boot_intent.py` 生成，`mosaico.py recover` 烧入 |
| **sysmeta NVS intent** | **运行中的应用主动请求**进维护 | 应用（见 §7） |

细节：
- **① bootstrap 包**：`ota_data_initial.bin` = 8192 B，除偏移 32 处 16 字节 `MOSAICO-BOOT-02!` 外全 `0xff`。首次工厂/recover 后"打开即维护"。消费后**只擦首扇区、保留第二扇区**。
- **② 软件请求**：`request_vibe_mode()` 先校验 `vibe_mode` 分区 address/size 符合 ABI、`esp_image_verify()` 校验保留镜像，再写 `sysmeta/mosaico_boot_v2/intent`，**不碰 otadata**，然后重启。
- **③ 回正常应用**：由 Vibe Mode 侧走 IDF 标准 OTA 选择 → 清除软件意图 → 重启。**调用者负责串行化启动转换**。单槽无 A/B，不承诺无损回滚。
- **fail-safe**：bootloader 读意图失败/类型不符时，优先选 Vibe Mode。

---

## 6. 构建期集成：`tools/cmake/retained_recovery.cmake`

这是 mosaico-claw 把"别人的固件"拼进自己烧录清单的地方（根 `CMakeLists.txt` 末尾 include）。

1. **定位 bundle**：`third-party/esp-mosaico-utils/esp-mosaico-vibe-mode/firmware/vibe-mode/prebuilt/recovery`，可用 `-D ESP_MOSAICO_VIBE_MODE_BUNDLE=<path>` 覆盖（用于"用当前源码重新构建的 Vibe Mode"）。
2. **每次构建都完整校验** bundle：调用 `prepare_recovery.py`，核对 target / layout / 各镜像 offset+size+SHA-256 / secure-boot+flash-enc 一致 / otadata 必须是 0.2 bootstrap / recovery ≤ 1.75 MiB / ESP 镜像 magic。输出到 `build/recovery/bundle`。
3. **生成"空白 otadata"**：`recovery_claw_otadata` = 8192 × `0xff`。注释原文：*Blank otadata selects Claw instead of the upstream Vibe Mode marker.*
4. **重写 `flash` 目标的镜像清单**为固定全量布局：

   | 分区 | 来源 |
   |---|---|
   | bootloader @ `0x2000` | **bundle 的 bootloader**（不是本地构建的） |
   | partition-table @ `0x8000` | 本地构建 |
   | otadata | **空白**（→ 选 Claw） |
   | vibe_mode | bundle 的 `vibe-mode.bin` |
   | app @ `0x210000` | 本地 `main_app` |
   | ui_apps / system | 本地资源 |

5. **`update-flash`** = 仅 `app + ui_apps + system`（不重刷 bootloader/vibe_mode）→ 日常更新。
6. **`claw_ota_size`** 做镜像大小检查。
7. **保留语义**：保留应用 NVS 与 Vibe Mode 的 sysmeta（不覆盖），不整片擦除。

### 三个重要推论
1. 产品的 **bootloader 来自 Vibe Mode 包**，不是本仓库的 → 启动选择逻辑、启动 Logo、屏幕交接都由 utils 那个 bootloader 决定。
2. 因此本仓库 `components/mosaico_boot_splash/` 的 **bootloader 分支（`bootloader/mosaico_boot_splash.c`）实质是死代码**：其 README 声称经 `BOOTLOADER_EXTRA_COMPONENT_DIRS` 注册，但**全仓无任何地方设置该变量**。真正生效的只有 app 侧兜底绘制 `app/mosaic_boot_splash.c`（main/CMakeLists.txt 以普通组件方式依赖）。
3. `tools/cmake/esp_idf_patch.cmake` **不触碰 bootloader/ota/partition**（改的是 LCD/parlio、USB 描述符、HTTP client）。

---

## 7. mosaico-claw 侧的触发点

| 方向 | 机制 | 文件 |
|---|---|---|
| Claw → Vibe Mode | 请求维护：写 NVS `intent` | `components/app_claw/app_claw_cli.c`（`request_vibe_mode` / `cmd_vibe`） |
| Claw → utils 编译期 | 引用 ABI 宏 | `components/app_claw/CMakeLists.txt:52`（加 `.../esp-mosaico-vibe-mode/include`） |
| 构建系统 | 拼 flash 清单 + 校验 bundle | `tools/cmake/retained_recovery.cmake` |
| 分区表 | 前 5 分区冻结 | `partitions_16MB.csv` |
| 构建期选择 | 按 flash 大小选分区表 | `tools/cmake/flash_partition_defaults.cmake` |
| 打包发布 | 元数据引用 recovery 版本/哈希 | `tools/package_firmware.py`（`installation.recovery_version/compatibility/recovery_sha256`） |
| CI | 改动触发构建 | `.gitlab/ci/*` |

> **产品侧进入 Vibe Mode 的入口只有两个**：上电 GPIO7，或串口 console 的 `vibe` 命令。**`mosaic_ui` 内没有任何 UI 入口**（全仓 `vibe` 只命中 `app_claw_cli.c`）。产品化时需自行补 UI/服务入口。
>
> **命名陷阱**：`main/app_recovery.c` 的 `app_recovery_mark_healthy()` 与 Vibe Mode **无关**，它是 IDF OTA 回滚确认（把 `main_app/ota_0` 标记 valid）。

---

## 8. Vibe Mode 运行时组件结构

`main` 是薄入口，按顺序组装（顺序即依赖）：

```
硬件静音+sysmeta → provider/文件卷 → Iris USB → Ideas/UI/输入 → 自动更新+网络 → HEALTHY
```

| 组件 | 职责 |
|---|---|
| `app_system` | 启动安全、保留元数据（sysmeta） |
| `app_storage` | NAND（SPI NAND + wear-leveling + LittleFS，挂 `/nand`）访问控制 |
| `app_network` | Wi-Fi 与发现 |
| `app_system_update` | 系统更新**策略与事务**（USB/TCP/NAND 三来源共用写入后端） |
| `app_nand_update` | 离线系统更新来源（扫 `/nand/system-update/`） |
| `app_ideas` | Ideas 库 + 展示资源 |
| `app_bridge` | HTTPS 维护 Bridge（`control_protocol 2` 独立写入授权） |
| `app_ui` | 显示/输入 + **自带轻量 UI**（`light/vui.c` 自绘 RGB565、`vui_decode.c` 内嵌 PNG 解码） |
| `app_iris` | Iris 产品服务适配（USB/TCP） |

要点：
- **Vibe Mode 完全不用 GSP/LVGL**：`vui.c` 自绘 + `ui/main.json` 作者布局 + `compile_vui.py` 预烘焙字形/图标/RGB565+A8 → `vui_assets.bin`。**这与 mosaico 的 GSP 路线是两套东西。**
- 固件与 PC 模拟器**共用同一套 C 控制器**（`vibe_ui.c` + `vui.c`），可 `cmake -S pc` 起浏览器模拟。
- 用的是 **ESP-Iris 0.2** 框架（也在 utils 内）。

---

## 9. Ideas 子系统工作逻辑

云端契约 `mosaico-ideas/device-release/1`，默认 origin `https://mosaico-ideas.espressif.com`：

```
用户输入 6 位编号
  → GET /api/v1/ideas/{code}/manifest.json   (Accept: multipart/related)
     一次返回：根清单 + 固件清单 + 当前语言 display + 字体(MIFF) + 图标
     有界流式解析 + ETag + 每部分长度/SHA-256 校验
     （根清单 ≤64 KiB，固件清单 3072 B，单语言资源 ≤1 MiB，整响 ≤1121280 B）
  → 预览页显示名称/图标/描述/版本/大小（缓存于内存，不落 NAND）
  → 用户确认 Download
     → GET /api/v1/ideas/files/{sha256}  取缺失组件
     → 落盘 /nand/ideas/<code>/current/（先 .incoming 完整校验，再目录重命名原子替换）
  → 用户点 Open
     → 从 NAND 读取 → 写 NOR 应用槽 → 回读校验 SHA-256 → 提交分区表+启动目标 → 重启
```

硬性约束：
- **NOR 单应用槽，不是 A/B**。擦写前设 Vibe Mode 启动意图，成功后清除；失败可回 Vibe Mode 重试。
- Ideas **不允许更新 bootloader 或保留的 Vibe Mode 镜像**。
- 离线打开需核对清单；同版本且 NOR 完全一致才跳过写入。
- 排版/换行/字形子集化**在发布端完成**（MIFF），设备只解码所需字形；`ideas_font.c` / `ideas_presentation.c` **不依赖 GSP/IDF** → 未来换 UI 框架可复用同一批下载资源。
- 最多 16 个 Ideas，单个 `<code>` 只保留一个完整版本；`.previous` 仅用于中断恢复。

---

## 10. 与 mosaico-claw 的关系与耦合

**方向是单向的**：Claw 依赖 utils 的 ABI 宏与预编译包；utils 不认识 Claw。

| 耦合面 | 性质 | 二开影响 |
|---|---|---|
| 前 2 MiB 分区布局 | **冻结 ABI** | 你的分区表必须保持 5 个不可变分区完全一致 |
| bootloader 归属 | 用 bundle 的 bootloader | 自定义启动行为要回 utils 改 |
| `flash`/`update-flash` 目标 | 被 `retained_recovery.cmake` 接管重写 | 你的烧录流程受约束 |
| 预编译包 | S31 专属二进制 | 换芯片/换屏必须回 utils 重建 |
| `app_claw_cli.c` 的 `vibe` 命令 | 唯一的软件入口 | 可保留/可去除 |

---

## 11. 对"服务层 + 自研 UI"方案的影响与建议

1. Vibe Mode **独立于 UI 层**，与重写 UI 无关。它是**保留还是去除**的架构决策，不是"要改"的决策。
2. **保留的价值**：单槽设备需要一个刷不坏的维护固件；它自带 USB/TCP 恢复、NAND 更新、HTTPS Bridge、Wi-Fi 配网——是服务层之外同样高价值的资产。
3. **保留的代价**（比普通"跟随层"更硬）：
   - 前 2 MiB 布局冻结，分区表 5 分区必须逐项一致；
   - 产品烧的是 bundle 的 bootloader，改启动逻辑要回 utils；
   - `flash`/`update-flash` 被接管重写；
   - 用"当前源码 Vibe Mode"必须能构建 utils 工程（需 ESP-Iris + esp-mosaico-bsp + 工具链）；
   - bootloader Logo/交接是 **CO5300 QSPI 专属**，且 bootloader 有 **24 KiB 固定容量上限**（S31 实测 24,432 B，仅余 144 B）——**换屏必须回 utils 改 bootloader 并重测容量**。
4. **建议**：
   - 若产品不需要"工厂级刷不坏维护固件"→ **整体去除**（去掉 `retained_recovery.cmake`、`app_claw_cli.c` 的 `vibe` 命令、`esp-mosaico-utils` 子模块引用，分区表自作）。可一次性消掉**冻结 ABI + 接管 flash 目标 + 异常 bootloader 归属**三层硬约束，也免掉每个屏都维护 bootloader 的负担。
   - 若保留 → 视其为**与 UI 层同级的"冻结层"**，并在开工清单加"bootloader 适配新屏 + 容量校验"。
   - 折中：**剥离机制、复用思想**——保留"单槽 + 常驻维护固件 + NVS 启动意图 + otadata bootstrap"的设计模式，但用**自建的小维护固件**替换 S31 专属的 Vibe Mode 产物（避免被冻结 ABI 与 bootloader 归属绑死）。

---

## 12. 关键实现文件索引

| 主题 | 路径（相对 `third-party/esp-mosaico-utils/`） |
|---|---|
| ABI 契约头 | `esp-mosaico-vibe-mode/include/mosaico_recovery_contract.h` |
| 产品契约 | `esp-mosaico-vibe-mode/product_contract.json` |
| 启动选择逻辑 | `esp-mosaico-vibe-mode/firmware/vibe-mode/bootloader_components/main/bootloader_start.c` |
| 私有 NVS 读取 | `…/bootloader_components/main/mosaico_boot_nvs.c` |
| 启动 Logo/交接 | `…/bootloader_components/main/mosaico_boot_splash.c` |
| 打包/校验/原子落盘 | `…/firmware/vibe-mode/tools/prepare_recovery.py` |
| bootstrap otadata 生成 | `…/firmware/vibe-mode/tools/boot_intent.py` |
| Ideas 核心 | `…/firmware/vibe-mode/components/app_ideas/src/*.c` |
| 轻量 UI | `…/firmware/vibe-mode/components/app_ui/light/vui.c`、`vui_decode.c` |
| 系统更新事务 | `…/firmware/vibe-mode/components/app_system_update/src/*.c` |
| HTTPS Bridge | `…/firmware/vibe-mode/components/app_bridge/src/*.c` |
| 软件规格 | `esp-mosaico-vibe-mode/docs/esp-mosaico-vibe-mode-software-spec-zh.md` |
| 启动契约与迁移 | `esp-mosaico-vibe-mode/docs/migration-0.2.md` |

产品侧：
| 主题 | 路径（相对仓库根） |
|---|---|
| 进入 Vibe Mode 命令 | `components/app_claw/app_claw_cli.c`（`request_vibe_mode` / `cmd_vibe`） |
| 构建期集成 | `tools/cmake/retained_recovery.cmake` |
| 分区表 | `partitions_16MB.csv` |
| boot splash（app 侧） | `components/mosaico_boot_splash/app/mosaic_boot_splash.c` |
| 发布打包元数据 | `tools/package_firmware.py` |

---

*本文档为只读勘察快照。开工时以 §11 建议与主决策文档 §8 待决项为起点。*
