# 开发环境与 PC 交互测试交接记录

> 本文档记录 2026-10-19 这一轮**实际动手验证**的结论，供"新对话直接开工"使用。
> 与 `mosaico-fork-analysis-and-decision.md`（架构决议）、`vibe-mode-and-ideas-analysis.md`（Vibe Mode / sim_bridge 分析）配套阅读。
> 本文只写**已验证的事实**和**可复现的命令**；推测性内容一律标注。

---

## 0. 一句话现状

- 工具链已在本机跑通：**ESP-IDF 6.1.0（EIM 安装）→ esp32p4 编译成功**。
- PC 交互测试闭环已实测：**sim_bridge 编译 UI 后端 → 注入点击 → 截图 PNG → AI 直接读图**。
- 整机构建**卡在 esp-claw**：按用户要求不下载官方仓库，等待其优化版。
- **未对开发板做任何操作**（无 flash / monitor / erase，无 `-p`）。

---

## 1. 本机工具链（已验证）

### 1.1 安装形态

ESP-IDF 用 **EIM（Espressif Installation Manager）** 安装，目录布局与经典 `export.sh` **不同**：

| 项 | 路径/值 |
|---|---|
| 激活脚本 | `~/.espressif/tools/activate_idf_v6.1.sh` |
| IDF 源码 | `~/.espressif/v6.1/esp-idf` |
| Python venv | `~/.espressif/tools/python/v6.1/venv` |
| 工具链根 | `~/.espressif/tools` |
| ESP-IDF 版本 | **6.1.0** |

> **重要**：不要用 `~/.espressif/v6.1/esp-idf/export.sh`，它会把 `IDF_PYTHON_ENV_PATH` 指到经典路径
> `~/.espressif/python_env/idf6.1_py3.13_env`，该环境不存在，会直接报错。

每个 shell 会话先执行：

```bash
source ~/.espressif/tools/activate_idf_v6.1.sh
```

### 1.2 已安装版本与目标支持

| 组件 | 版本 |
|---|---|
| ESP-IDF | 6.1.0 |
| riscv32-esp-elf-gcc | 15.2.0 (`esp-15.2.0_20251204`) |
| CMake | 4.3.3 |
| Ninja | 1.12.1 |
| esptool | 5.5.0 |
| idf-component-manager | 3.1.2 |
| esp-bmgr-assist | 0.8.3 |
| esp-gsp-tools | 0.1.2 |
| Pillow | 13.0.0 |
| pyserial | 3.5 |

**目标支持（关键）：**

| 目标 | 状态 |
|---|---|
| `esp32p4` | ✅ 正式目标 |
| `esp32s31` | ⚠️ **preview 目标**，`idf.py --list-targets` 默认不显示 |

需要 S31 时必须显式预览：

```bash
idf.py --preview set-target esp32s31
idf.py --preview --list-targets    # 可看到 esp32s31
```

本机同时装了 v5.5.5 / v6.0.2 / v6.1，但**只有 v6.1 含 esp32s31**；P4 三个版本都支持。

### 1.3 Python 依赖

`requirements.txt` 已装入 IDF 的 v6.1 venv：

```bash
source ~/.espressif/tools/activate_idf_v6.1.sh
python -m pip install -r requirements.txt
```

### 1.4 已验证的编译命令

**P4 冒烟编译（不碰仓库、不碰板子）：**

```bash
source ~/.espressif/tools/activate_idf_v6.1.sh
cp -r "$IDF_PATH/examples/get-started/hello_world" /tmp/idf_smoke_p4
cd /tmp/idf_smoke_p4
idf.py set-target esp32p4
idf.py build
```

结果：`Successfully created ESP32-P4 image`，`hello_world.bin` 167 KB。

**板级管理器（在本仓库根目录）：**

```bash
idf.py bmgr -c ./boards -b esp_mosaico
```

结果：9 个 peripheral、15 个 device 校验通过；生成 `components/gen_bmgr_codes/` 与 `managed_components/`，**`git status` 仍然干净**（这些目录已被忽略）。

---

## 2. PC 交互测试闭环（本轮核心成果）

### 2.1 结论

**AI（我）可以在没有任何硬件的情况下完成下面这个循环：**

```
改 UI 源码/场景 JSON
  → 编译 native backend
  → 启动 sim_host
  → API 注入 tap/drag
  → screenshot 存 PNG
  → 我直接读 PNG（模型支持图像输入）
  → 改
```

并且可以同时拿到 host 日志、`frame_info` 的脏矩形、后端进程的 stdout。

### 2.2 获取 esp-gsp（比 GitHub 快得多）

GitHub 的 `git clone` / codeload 在本机只有 ~40–50 KB/s，esp-gsp 仓库会超时。
**改用乐鑫组件仓库的 zip：**

```bash
# 查版本
curl -s https://components.espressif.com/api/components/espressif/esp-gsp \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print([v['version'] for v in d['versions']])"

# 下载 1.3.1（项目依赖为 ^1.3.0）
curl -L -o /tmp/esp-gsp.zip \
  https://components-file.espressif.com/components/espressif/esp-gsp/1.3.1/espressif__esp-gsp-v1.3.1.zip
unzip -q /tmp/esp-gsp.zip -d /tmp/esp-gsp-1.3.1
```

> 组件仓库里 esp-gsp 现有版本：`1.7.0 / 1.6.0 / 1.5.1 / 1.5.0 / 1.4.0 / 1.3.1 / 1.3.0 / 1.2.0 / 1.1.0 / 1.0.0`。
> 项目 `components/mosaic_ui/idf_component.yml` 钉的是 `espressif/esp-gsp: "^1.3.0"`。

### 2.3 GSP 工具是"按版本下载"的

`esp-gsp` 的 `.gspc_version` 指定编译器版本（1.3.1 为 **0.4.1**）；模拟器版本跟随组件版本。
`run.py` 会通过 `esp-gsp-tools` 自动下载并缓存到：

```
~/.espressif/esp-gsp-tools/gspc/v0.4.1/macos-aarch64/bin/gspc
~/.espressif/esp-gsp-tools/sim/v1.3.1/macos-aarch64/bin/sim
```

- `esp-gsp-tools list` **只列 gspc，不列 sim**（sim 由 `python -m gsp.execute` 管理），不要以为 sim 没装。
- 离线时可用 `GSP_TOOLS_NO_AUTO_DOWNLOAD=1` 只使用缓存。

### 2.4 运行官方示例（验证链路最快的办法）

```bash
source ~/.espressif/tools/activate_idf_v6.1.sh
mkdir -p /tmp/gsp_work && cd /tmp/gsp_work

python /tmp/esp-gsp-1.3.1/tools/sim_bridge/run.py \
  --project /tmp/esp-gsp-1.3.1/examples/hello_world/pc \
  --component-dir /tmp/esp-gsp-1.3.1 \
  --headless --no-browser --duration 20
```

实测输出要点：

```
-- sim_bridge: using gspc 0.4.1 from esp-gsp-tools 0.4.1
gspc: resource-usage bundle_bytes=9552 scenes=1 ...
[100%] Built target hello_backend
I (gsp_deployable): Opened deployable GSPB: scenes=1
INFO application backend TCP endpoint listening endpoint=127.0.0.1:8684
hello_backend: load=92, last_error=0
```

> native 构建**不需要激活 ESP-IDF 环境**（官方明确说明）；激活只是为了统一 Python/工具版本。

### 2.5 直接截图（一次性，无需写代码）

```bash
SIM=~/.espressif/esp-gsp-tools/sim/v1.3.1/macos-aarch64/bin/sim

"$SIM" --bundle app.gspb --headless --frames 10 \
  --dump /tmp/frame.png --dump-format png
```

实测：`PNG image data, 800 x 480`，并且 `read` 工具可以直接打开查看。

### 2.6 任意分辨率 / 竖屏（已实测 800×1280）

GSP 的 `sim_host` **不硬编码 480**，分辨率完全由 bundle 内场景的 `w`/`h` 决定。

```bash
GSPC=~/.espressif/esp-gsp-tools/gspc/v0.4.1/macos-aarch64/bin/gspc

# 场景 JSON 顶层需要： "w": 800, "h": 1280, "font": "fonts/DejaVuSans.ttf"
"$GSPC" pack scene.json -o portrait.gspb

"$SIM" --bundle portrait.gspb --headless --frames 8 \
  --dump /tmp/portrait.png --dump-format png
```

实测：`PNG image data, 800 x 1280`，卡片/列表/底部按钮全部按绝对坐标正确渲染。

> 场景 JSON 结构参考 `esp-gsp-1.3.1/examples/hello_world/scenes/hello_800.json`。
> `gspc pack` 会打印资源占用：`bundle_bytes / animation_frame_peak_bytes / image_cache_budget_bytes`。

### 2.7 交互 + 后端全链路（已实测）

`run.py` 内部做的事（可手工复现）：

1. 启动 `sim`：
   ```
   --bundle <app.gspb> --headless
   --backend-enable --backend-required --backend-idle-timeout 30
   --ready-file <ready.json>
   --api-listen tcp://127.0.0.1:<API_PORT>
   --frames 0 --log-file <log>
   ```
2. 读 `ready.json` 拿到 `backend` 地址：
   ```json
   {"api":"tcp://127.0.0.1:9010","backend":"tcp://127.0.0.1:8684","bridge_version":1,"browser":null,"pid":42188,"version":1}
   ```
3. 启动应用后端：`hello_backend --endpoint tcp://127.0.0.1:8684 --duration-ms 25000`

实测输出：

```
capabilities: {"abi_version":1, "backend_extensions":{...}, ...}
wait 40 frames: {'ok': True}
screenshot: {'height':480, 'path':'/tmp/full_1.png', 'width':800}
tap: {'ok': True}
screenshot: {'height':480, 'path':'/tmp/full_2.png', 'width':800}
frame_info: {"damage":[{"x1":60,"x2":740,"y1":330,"y2":358}], "full":false, "seq":9}
backend output: hello_backend: load=30, last_error=0
```

第二张图里进度条变成 30% —— **后端数据 → 渲染 → AI 读图**，环闭合。
`damage` 只包含进度条区域，说明脏矩形跟踪可用（可用于性能/重绘范围断言）。

### 2.8 API 协议（1.3.1）

- **JSON-RPC 2.0**
- 传输：`stdio` / `tcp://127.0.0.1:PORT` / `unix:///path.sock`
- 帧格式：**`Content-Length: N\r\n\r\n` + JSON body**（不是换行分隔！）

最小客户端：

```python
import json, socket

sock = socket.create_connection(("127.0.0.1", 9010))

def rpc(method, params=None, rid=1):
    body = json.dumps({"jsonrpc": "2.0", "id": rid,
                       "method": method, "params": params or {}}).encode()
    sock.sendall(b"Content-Length: " + str(len(body)).encode() + b"\r\n\r\n" + body)
    buf = b""
    while b"\r\n\r\n" not in buf:
        buf += sock.recv(4096)
    header, _, rest = buf.partition(b"\r\n\r\n")
    n = int(header.split(b":")[1].strip())
    body = rest
    while len(body) < n:
        body += sock.recv(n - len(body))
    return json.loads(body[:n]).get("result")
```

**1.3.1 可用方法：**

| 分类 | 方法 |
|---|---|
| 输入 | `tap`, `drag`, `feed_pointer`, `feed_touch` |
| 场景/状态 | `goto_scene`, `reset`, `set_value`, `set_text`, `set_color`, `set_visible`, `set_component_i32`, `keyboard_attach`, `set_cursor`, `set_swipe_transition`, `drawer_open/close/is_open`, `page_flow_set_page`, `list_bind_component/snap/fling/scroll_to`, `set_swipe_enabled`, `fling_messages` |
| 工具 | `ping`, `capabilities`, `frame_info`, `wait`, `invalidate`, `screenshot`, `subscribe`, `unsubscribe`, `quit` |

**通知事件：** `scene_changed`, `callback`, `frame`, `list_bind`, `list_bind_overflow`, `binary_result`, `backend_state`, `log`, `wasi_log`

**错误码：** `-32600` / `-32601` / `-32602` / `-32603` / `-32010 (input_busy)`

### 2.9 sim_host 关键 CLI

| 参数 | 说明 |
|---|---|
| `--bundle <PATH>` | `.gspb` 场景包 |
| `--headless` | 不启浏览器 |
| `--frames <N>` | N 帧后退出；`0` = 一直跑 |
| `--dump <PATH>` + `--dump-format png` | 结束时存图 |
| `--tap X Y` / `--drag x1 y1 x2 y2` | 命令行脚本化输入（可重复，按顺序执行） |
| `--api-listen tcp://...` | 启用 JSON-RPC 控制通道 |
| `--backend-enable` / `--backend-listen` | 应用后端通道 |
| `--backend-required` / `--backend-idle-timeout <s>` | 等后端连上再推进 |
| `--ready-file <PATH>` | 写出实际端口（自动化必需） |
| `--log-level` / `--log-file` | 日志 |
| `--fps` | 仿真帧率（默认 60） |
| `--input-mode shared \| api-exclusive \| browser-exclusive` | 输入归属 |
| `--invalidate-every-frame` | 强制全屏重绘（视觉检查） |
| `--capabilities` | 打印能力集后退出 |

---

## 3. 已验证 vs 未验证

### ✅ 已验证

- IDF 6.1 激活、P4 编译链路（hello_world）
- `idf.py bmgr` 在本仓库可用
- `sim_bridge` native 后端可编译、可连接、可收发
- `screenshot` → PNG，模型可直接读图
- 800×480 渲染
- **800×1280 竖屏**渲染（自建场景）
- `tap` 注入、`wait(frames)`、`frame_info` 脏矩形
- 后端定时数据驱动 UI（load=30）
- 分辨率由 bundle 决定，无 480 硬编码

### ❌ 未验证 / 做不到

| 项 | 原因 |
|---|---|
| 真机帧率 / 带宽 / PPA | 模拟器明确声明不替代设备时序；静态场景跑出 2455 fps 无意义 |
| 真机屏幕截图 | 产品层无该接口（见 §5） |
| 整机固件编译 | esp-claw 子模块按用户要求未下载 |
| 真机日志 | 需联网 + 开启设置项（见 §5） |
| 触摸 | 目标板无触控 |
| `inspect_object` / `hit_test` / `list_objects` | **esp-gsp 1.3.1 没有这些方法**，master / 1.7.x 才有 |

> 若需要"不看图也能断言布局"，需要升级 esp-gsp，或退化为"截图 + 已知坐标"。

---

## 4. 已知阻塞与成本（开工前必须记住）

### 4.1 现有 `mosaic_ui` 不能直接在 PC 上编译

`sim_bridge` 只提供 2 个头文件：
`gsp_sim_bridge.h`、`gsp_sim_backend.h`，**不含任何 ESP-IDF 桩**。

官方约束原文：显示/触摸初始化、FreeRTOS 任务、驱动、设备服务必须留在 portable UI 源码之外，用小型 HAL/PC mock 替换。

`components/mosaic_ui` 的 UI 层统计：

| 项 | 数量 |
|---|---|
| `.c` 文件 | 45 |
| `.h` 文件 | 35 |
| 引用 IDF 头的 `.c` | **20 / 45** |
| 纯 `esp_gsp.h + libc` 的 `.c` | 25 |

IDF 头使用文件数：

```
esp_log.h       16
esp_err.h       16
freertos/task.h  7
freertos/semphr.h 6
freertos/portmacro.h 3
freertos/queue.h 3
freertos/event_groups.h 1
esp_timer.h      4
esp_heap_caps.h  5
nvs.h            2
sdkconfig.h      3
esp_attr.h       6
esp_check.h      3
esp_pthread.h    1
```

→ 要做 PC 构建，需要一层 host shim（`esp_log` → stdout、`esp_err` → typedef、FreeRTOS → pthread/空实现、`esp_timer` → clock、`heap_caps` → malloc、`nvs` → 空实现）。
**规模约 10–15 个头、几百行，可控但不是"加个 CMakeLists"就行。**

### 4.2 18 个 GSP API 缺失（1.3.x）

`mosaic_ui` 用到的 65 个 GSP API 中：36 已实现、3 静态内联、8 仅设备端（全部 16 处调用都在
`components/mosaic_ui/core/mosaic_esp_platform.c`）、**18 个模拟器未实现**：

```
stack_view_push / pop / get_top / is_animating   (41 调用 / 6 文件)
set_overlay_contributor
overlay_builder_round_rect / glyph_a8            (16 调用 / 1 文件)
component_animate_position_to / get_position / stop_position_animation (14 / 3)
set_input_interceptor                             (6 / 2)
component_set_many / set_properties               (4 / 2)
on_pinch                                          (2 / 1)
message_list_bind_component / changed             (2 / 1)
config_override_set
stop
```

替代方案：导航用 `esp_gsp_goto_scene` + 显式页面状态机；chrome 用普通场景组件而非 overlay；
动画用 `esp_gsp_component_animate_property`；输入用 `esp_gsp_on_event` + 自有归一化层。

### 4.3 esp-claw

- 子模块 `third-party/esp-claw` 当前状态：**未初始化**（`-62aaeed...`）。
- **用户明确要求：不要下载官方仓库**，后续由其提供优化版。
- 因此：整机构建、`idf.py build`、真实 UI 集成**都要等**。

清理命令（如误 clone 后需要回退）：

```bash
rm -rf third-party/esp-claw
rm -rf .git/modules/third-party/esp-claw
mkdir -p third-party/esp-claw
git submodule status     # 应显示前缀 '-'
```

### 4.4 不要烧录

开发板当前连接电脑做其他用途。**禁止**：

- `idf.py flash`
- `idf.py monitor`
- `idf.py erase-flash`
- 任何带 `-p <PORT>` 的命令

本轮全程只做编译与 PC 模拟。

---

## 5. 设备侧"日志 / 截图"现状

### 5.1 真机日志：产品层已有

| 能力 | 位置 |
|---|---|
| WebSocket 实时日志 | `WS /ws/logs` |
| 日志开关状态 | `GET /api/logs/status` |

- 实现在 `components/http_server/http_server_logs_api.c`
- 通过 `esp_log_set_vprintf()` 挂接，队列深度 8、单行 384 字节、最多 2 个 WS 客户端
- 开关：设置页 `web_logs_enabled` → `http_server_set_web_logs_enabled()`
- `http_server_start()` 在 `main/main.c` 默认调用

**结论：只要设备联网且知道 IP，就能远程拉真机日志**——与 Vibe Mode 无关，是产品层自带能力。
建议：调试构建默认开启该设置项。

### 5.2 真机截图：没有

- `http_server` 路由里无截图接口（现有：`/api/capabilities`、`/api/config`、`/api/files*`、`/api/logs/status`、`/api/restart`、`/api/status`、`/api/lua-modules`、`/api/webim/*`、`/api/wechat/*`）。
- `display_service` 的 `frame_buffer` 只是 `display_service_raw_blit_t` 的**输入**，`get_raw_info` 只返回尺寸/像素格式，**没有导出读帧 API**。
- Vibe Mode 的 `factory_ui.c` 有 "retained framebuffer" 注释，但属于 presenter 内部，不对外暴露。
- Vibe Mode 的 Iris **只提供指针注入**（`app_iris_input_register` → `touch(x,y,pressed)`）与控制命令（inventory / recovery_ota），**没有日志流、没有帧缓冲导出**；其 `snapshot` 是网络/Bridge 状态 JSON。

**结论：要让 AI 看到真机画面，需要新增 `/api/screenshot`**（读保留帧缓冲 → PNG → HTTP）。
这是一个明确的新功能项，不是"保留 Vibe Mode 就能顺带得到"的。

---

## 6. 架构含义（本轮新增结论）

要同时满足"上板"和"PC 可交互测试"，新 UI 必须这样切：

```
core/          纯 C + esp_gsp.h + libc，单线程事件驱动（PC 可编译）
port_device/   FreeRTOS 任务、display_service、wifi/audio/sd
port_pc/       mock 服务 + 主循环 + 日志桥
```

- 同一份 `core/` 既编进固件，也编进 `sim_bridge`。
- 这不是额外开销：GSP 设备端本来就要求所有调用在同一任务上，天然单线程。
- 它同时强制 UI 与设备解耦，对"多板 / 多形态"本来就是正确方向。
- 与 `mosaic_platform_ops_t`（`components/mosaic_ui/runtime/mosaic_runtime.h`）的既有思路一致，可直接沿用/收窄。

对应到 sim_bridge 的入口：

```c
gsp_bridge_app_init();   /* 建立 UI、注册回调、设置定时器 */
gsp_bridge_app_deinit();
```

PC 侧只需要 `pc/CMakeLists.txt`：

```cmake
gsp_add_backend(<target> SOURCES <portable ui.c> SCENES <scene.json>)
```

---

## 7. 下一步（新对话开工清单）

> 前置：等待用户提供 esp-claw 优化版并替换 `third-party/esp-claw`。

### 阶段 A：不依赖 esp-claw，现在就能做

1. **建 P4 板级骨架** `boards/jc_esp32p4_m3/`
   - `board_info.yaml` + `board_peripherals.yaml` + `board_devices.yaml`
   - DSI/JD9365 800×1280、ES8311、I2S、SD(4-bit SDIO)、S1–S4、扩展 I2C
   - 校验：`idf.py bmgr -c ./boards -b jc_esp32p4_m3`
   - 参考：`boards/esp_mosaico/`、`docs/board-jc-esp32p4-m3-dev.md`
2. **判断 `mosaic_ui` 上 PC 的真实成本**（预期会撞 §4.1 / §4.2）
   - 先只编译 `core/` + `runtime/`，把 IDF 头逐个打桩
   - 输出：shim 头清单 + 行数 + 缺失 API 的实际影响面

### 阶段 B：esp-claw 到位后

3. 整机 `idf.py --preview set-target esp32s31`（或 P4）`+ idf.py build` 基线
4. Vibe Mode 移除（见 `vibe-mode-and-ideas-analysis.md` §11.1）
   - 删 `tools/cmake/retained_recovery.cmake` 及其 include
   - 删 `components/app_claw/app_claw_cli.c` 的 `request_vibe_mode` / `cmd_vibe`
   - 删 `components/app_claw/CMakeLists.txt` 的 utils include
   - 重写 `partitions_16MB.csv`
   - `tools/package_firmware.py:19` `TARGET` 改为 `esp32p4`
   - 移除 `third-party/esp-mosaico-utils` 子模块
5. Spike 1b：真机 800×1280 GSP 帧率 / 内存 / 带宽

### 阶段 C：UI 设计（仍待决）

6. 响应式 DSL 形式（设计基准 + scale 规则 vs 容器/锚点模型）
7. focus/nav 交互模型（方向键、确认、返回、滚动）
8. 自研 UI 与上游 `mosaic_ui` 的目录共存/替换方式
9. 无 IMU / 磁力计 / 电量计 / NAND 后的功能裁剪范围

### 可选的设备侧增强

10. `GET /api/screenshot`（读保留帧缓冲 → PNG）
11. 调试构建默认开 `/ws/logs`

---

## 8. 注意事项 / 踩坑记录

1. **不要用 `export.sh`**，用 EIM 的 `activate_idf_v6.1.sh`。
2. `esp32s31` 必须 `--preview`。
3. GitHub clone 极慢（~45 KB/s），esp-gsp 从 `components-file.espressif.com` 下 zip。
4. `esp-gsp-tools list` 不显示 sim，别误判。
5. API 用 **Content-Length 帧**，不是换行分隔。
6. `sim_host` 的 fps 数字对静态场景无意义。
7. `/tmp` 内容重启后可能丢失（esp-gsp 解压目录、验证脚本都在 `/tmp`），需要时按 §2.2 重下。
8. `idf.py bmgr` 会生成 `components/gen_bmgr_codes/` 与 `managed_components/`，但不污染 git。
9. esp-claw 子模块保持未初始化；不要 `git submodule update --init --recursive`（会拉官方仓库）。
10. **任何情况下不要 flash / monitor 当前开发板。**

---

## 9. 本轮产物位置（临时）

| 产物 | 路径 |
|---|---|
| esp-gsp 1.3.1 解压 | `/tmp/esp-gsp-1.3.1` |
| gspc 0.4.1 | `~/.espressif/esp-gsp-tools/gspc/v0.4.1/macos-aarch64/bin/gspc` |
| sim 1.3.1 | `~/.espressif/esp-gsp-tools/sim/v1.3.1/macos-aarch64/bin/sim` |
| hello PC 构建 | `/tmp/gsp_work/build/sim_bridge/hello_world/pc` |
| 800×1280 场景 | `/tmp/portrait/scene.json`、`/tmp/portrait/portrait.gspb` |
| 全链路脚本 | `/tmp/full_sim.py` |
| 截图样例 | `/tmp/hello.png`、`/tmp/portrait.png`、`/tmp/full_1.png`、`/tmp/full_2.png` |

`~/.espressif/esp-gsp-tools` 下的工具是持久缓存；其余在 `/tmp`。

---

## 10. 仓库状态（本文件提交前）

- 分支：`feat/board-ui-adaptation`
- 远端：`origin` = 用户 fork（`github.com/kayzh324/esp-mosaico-claw`），`upstream` = 官方
- `third-party/esp-mosaico-utils`：已初始化，钉在 `ca2f9c1`
- `third-party/esp-claw`：**未初始化**（等待用户优化版）
- 工作区：干净
