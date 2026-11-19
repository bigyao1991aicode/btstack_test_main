# btstack_test

将 [BlueKitchen BTstack](https://github.com/bluekitchen/btstack) v1.6.1 移植到博流 BL616 芯片的测试工程，运行于 [bouffalo_sdk](https://github.com/bouffalolab/bouffalo_sdk) 之上。

- 芯片：BL616（BL616DK 开发板实测）
- SDK：bouffalo_sdk **v2.3.35**（commit `63784aa0`）
- Host 协议栈：BTstack **v1.6.1**（tag `v1.6.1`，commit `2b49e57bd`）
- Controller：SDK 预编译库 **btblecontroller 1.6.207**（变体 `ble1m0s1bredr0`）
- 实测：BLE 广播可被 nRF Connect 正常扫描发现，GATT 连接可用

---

## 1. 环境准备

### 1.1 获取代码

```bash
git clone https://github.com/bouffalolab/bouffalo_sdk.git
cd ./bouffalo_sdk/examples
git clone https://github.com/bigyao1991aicode/btstack_test_main.git btstack_test
cd ./btstack_test
# btstack 以 gitlink 形式记录，需手动获取（指定 v1.6.1）
rm -rf ./btstack
git clone https://github.com/bluekitchen/btstack.git -b v1.6.1
```

### 1.2 工具链

BL616 使用 T-Head 自定义架构扩展，**必须使用 Xuantie 工具链**（GCC 10.4）：

- 下载：[bouffalolab/toolchain](https://github.com/bouffalolab/toolchain/releases/tag/compact-old) → `xuantie-gcc-riscv-v2.10.2-windows-x86_64.7z`
- ⚠️ `default` release 的 `gcc-riscv-14.3.0` **不可用**：不识别 `-march=rv32imafcpzpsfoperand_xtheade`
- 另外 SDK 需要 cmake（>=3.22）、make/ninja、python3

### 1.3 Windows 特别说明

构建脚本在 sh 环境中调用 `python3`，Windows 上可能解析到 Microsoft Store 存根（报错 `Error 0x2331`）。解决办法：把真实 `python.exe` 复制一份命名为 `python3.exe` 并放入 PATH：

```powershell
Copy-Item C:\Python314\python.exe <某目录>\python3.exe
```

## 2. 编译

```powershell
# Windows PowerShell（PATH 按实际安装位置调整）
$env:Path = "<shim目录>;<xuantie工具链>\bin;$env:Path"
cd examples\btstack_test
make CHIP=bl616 BOARD=bl616dk
```

产物位于 `build\build_out\`：`btstack_test_bl616.bin / .elf / .map / .ota`。
参考占用：ROM 384KB / 4MB，RAM 49KB / 415KB（含 PSRAM 4MB 堆）。

### 2.1 更换示例

在 `CMakeLists.txt` 修改：

```cmake
set(EXAMPLE "gatt_counter")   # 当前默认，BLE 广播/GATT 验证用
```

可用示例见 `btstack/example/*.c`。注意事项：

| 示例 | 类型 | 说明 |
|------|------|------|
| `gatt_counter` | BLE | ✅ 当前默认，广播+Battery/Counter 服务 |
| `a2dp_sink_demo` | 经典蓝牙 | A2DP 音频接收（BR/EDR，BLE 扫描器看不到它，属正常） |
| `sm_pairing_peripheral` | BLE | 配对测试 |
| `led_counter` | BLE | ❌ 依赖 `hal_led` 移植层，本工程未实现 |
| 其他 | - | 带 `.gatt` 文件的示例首次编译前需手动生成头文件（见 2.2） |

### 2.2 依赖 GATT 的示例首次编译

CMake 的 `add_custom_command` 首次编译可能不执行，若报 `fatal error: xxx.h: No such file or directory`，手动生成后再编译：

```powershell
py btstack\tool\compile_gatt.py btstack\example\<示例名>.gatt build\<示例名>.h
make CHIP=bl616 BOARD=bl616dk
```

## 3. 烧录与串口

```powershell
make flash CHIP=bl616 BOARD=bl616dk COMX=COM11 BAUDRATE=2000000
```

- 烧录波特率默认 2000000（`project_flash.build` 中 `BAUDRATE ?= 2000000`）
- 固件日志串口波特率：2000000（`CONFIG_CONSOLE_UART_BAUDRATE`）
- 烧录前请关闭占用该串口的工具

启动正常标志：

```
Version: lib_version_btblecontroller_1.6.207
BTstack up and running on 7C:3E:82:F0:A6:F2.
```

Shell 命令：使用 `btstack erase` 可擦除已保存的链路密钥/配对信息（忘记已连接设备）。

---

## 4. 相对上游（O2C14/btstack_test）的关键修改

本仓库在原工程基础上做了以下修改，使其适配 bouffalo_sdk v2.3.35：

### 4.1 新增 `defconfig`（修复无法编译）

原工程仅有 `proj.conf`（cmake `set()` 语法），而 SDK 构建系统 `project.build` **只读取 Kconfig 格式的 `defconfig`**。导致 `CONFIG_FREERTOS` 等全部未生效（默认 n），编译报 `FreeRTOS.h: No such file or directory`。

`defconfig` 将 `proj.conf` 的意图翻译为 Kconfig 格式，并剔除了 v2.3.35 中已不存在的符号（`CONFIG_VLIBC`、`CONFIG_BFLOG`）。

### 4.2 切换控制器库（修复广播不发射的根因）⭐

原工程链接自带的旧控制器库 `btblecontroller/lib/libbtblecontroller_bl616_ble1m0s1sbredr1.a`，与 SDK v2.3.35 的 RF 驱动层不匹配——**HCI 广播命令全部返回成功（status 0x00），但射频静默不发射**，任何扫描器都搜不到设备。

修复：改用 SDK 官方组件中的控制器库（与 `examples/btble/peripheral` 同款）：

```ini
# defconfig
CONFIG_BLUETOOTH            =y
CONFIG_BTBLECONTROLLER_LIB  =ble1m0s1bredr0   # btblecontroller 1.6.207
CONFIG_BLE_HOST_DISABLE     =y                # 禁用 SDK 自带 host，由 btstack 充当 host
```

```cmake
# CMakeLists.txt：注释掉旧库
# sdk_add_include_directories(./btblecontroller/btble_inc)
# sdk_add_static_library(btblecontroller/lib/libbtblecontroller_bl616_ble1m0s1sbredr1.a)
```

附带收益：ROM 占用从 ~656KB 降至 ~384KB。

### 4.3 `btstack_port.c` API 适配（2 处）

| 位置 | 修改 | 原因 |
|------|------|------|
| `bl_packet_to_host()` | `bt_onchiphci_hanlde_rx_acl` → `bt_onchiphci_handle_rx_acl` | 新控制器库修正了 API 拼写（签名不变，用 `nm` 对比确认） |
| `settings_erase()` | `ef_port_erase(0, 32768)` → `ef_env_set_default()` | v2.3.35 的 EasyFlash4 已重写为 LittleFS 后端（`components/fs/littlefs/easyflash_port`），无 `ef_port_erase` |

### 4.4 `.gitignore`

新增忽略 `build_log.txt`（构建日志）。

---

## 5. 排障记录（问题 → 根因 → 解决）

| # | 现象 | 根因 | 解决 |
|---|------|------|------|
| 1 | `FreeRTOS.h: No such file or directory` | 只有 proj.conf，Kconfig 未吃到配置 | 新增 defconfig（4.1） |
| 2 | Kconfig 阶段报 `undefined symbol BFLOG/VLIBC` | v2.3.35 已删除这两个符号 | 从 defconfig 剔除 |
| 3 | `-march=...zpsfoperand_xtheade: extension 'p' is unsupported` | 用了标准 GCC 14.3 工具链 | 换 Xuantie v2.10.2（1.2） |
| 4 | `python3.exe: Error 0x2331` | Windows Store 存根 | python3 shim（1.3） |
| 5 | 链接错误 `ef_port_erase undefined` | EasyFlash4 已改 LittleFS 后端 | 改用 `ef_env_set_default()`（4.3） |
| 6 | 链接错误 `bt_onchiphci_hanlde_rx_acl undefined` | 新控制器库 API 改名 | 改为 `handle` 拼写（4.3） |
| 7 | `fatal error: gatt_counter.h: No such file` | custom command 首次不执行 | 手动 compile_gatt.py（2.2） |
| 8 | **HCI 广播返回成功但搜不到设备** | 旧控制器库与 SDK RF 驱动不匹配，静默不发射 | 切换官方控制器库 1.6.207（4.2） |
| 9 | 用 BLE 扫描器搜 A2DP Sink | A2DP 是经典蓝牙，无 BLE 广播 | 属正常，验证 BLE 用 `gatt_counter` |

**问题 8 的定位方法**（可复用的二分排查）：
1. 编译 SDK 官方 `examples/btble/peripheral` 烧录验证 → 广播正常 ⇒ 硬件/RF 无辜，问题在 btstack 移植层
2. 对比两侧启动日志的控制器版本（`lib_version_btblecontroller_x.x.x`）
3. `riscv64-unknown-elf-nm --defined-only <lib.a>` 对比新旧库导出符号差异

---

## 6. 命令速查

```powershell
# 编译（Windows，PATH 前缀按实际调整）
$env:Path = "<shim>;<xuantie>\bin;$env:Path"
make CHIP=bl616 BOARD=bl616dk

# 烧录
make flash CHIP=bl616 BOARD=bl616dk COMX=COM11 BAUDRATE=2000000

# 清理
make clean          # 清理构建产物
make distclean      # 连同配置一起清理

# 串口（2M 波特率）查看日志；shell 中：
btstack erase       # 擦除配对信息
```