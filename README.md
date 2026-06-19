# Ajie Try — BLE Central & Peripheral with PingPong OTA

基于 **Sifli（思澈科技）SF32LB5x 系列** 的蓝牙 BLE 双模（Central + Peripheral）示例工程。同时支持 GATT Server/Client、PingPong OTA 固件升级，运行在 **RT-Thread** 实时操作系统之上。

源码路径：`example/ble/central_and_peripheral_with_pingpong_ota`

---

## 支持的平台

| 芯片系列 | 示例板卡 |
|:---|:---|
| SF32LB55X | eh-lb555 |
| SF32LB58X | sf32lb58-lcd_n16r64n4 |
| SF32LB56X | — |
| SF32LB52X | sf32lb52-lchspi-ulp |

> Flash Table 烧录在 MPI5 工程中完成。

---

## 功能特性

- **BLE Peripheral（从设备）**
  - 开机自动广播，广播名 `SIFLI_APP-<BT_ADDR>`
  - 自定义 GATT Service（128-bit UUID）
  - 支持 Read / Write / Notify / Indication
  - 手机端可通过 LightBlue（iOS）或 nRF Connect（Android）连接测试

- **BLE Central（主设备）**
  - 扫描周边 BLE 设备
  - 发起连接、搜索远程 GATT Database
  - 远程读写特征值、使能 CCCD

- **GATT 自定义服务**
  - Service UUID: `00000000-0000-0070-7061-5F696C666973`
  - Characteristic UUID: `00000000-0000-0170-7061-5F696C666973`

- **PingPong OTA 升级**
  - 通过 Sifli BLE App 无线升级固件
  - Ping/Pong 双分区备份机制，升级失败可回滚

- **Finsh Shell 命令行**
  - 丰富的 `diss` 命令集，可交互式控制 BLE 行为

---

## 硬件需求

- 一块支持本工程的 Sifli 开发板
- 一根 USB 转串口线（用于下载和 Finsh 控制台）
- 手机（用于 BLE 连接测试和 OTA 升级）

---

## 快速开始

### 1. 编译

```bash
# 以 SF32LB58 为例
cd project/hcpu
scons --board=sf32lb58-lcd_n16r64n4 -j8
```

其他板卡替换 `--board=` 参数即可。

### 2. 烧录

编译完成后进入 `project/build_<board>/` 目录：

```bash
# JLINK 下载
./download.bat

# UART 下载（SF32LB52x 系列）
./uart_download.bat
# 按提示输入串口号
```

### 3. 运行验证

上电后串口输出 Finsh 控制台，输入 `diss` 查看所有命令：

```
diss conn_idx    — 连接指定广播设备
diss scan        — 扫描周边设备
diss show_dev    — 显示已连接设备
diss read_val    — 读取远程特征值
diss write_val   — 写入远程特征值
...
```

手机端打开 LightBlue / nRF Connect，搜索 `SIFLI_APP-*` 前缀的设备并连接，即可进行 GATT 读写测试。

---

## Finsh Shell 命令参考

### 广播与扫描

| 命令 | 说明 |
|:---|:---|
| `diss scan start [dup] [interval_ms] [window_ms] [duration_ms] [rssi]` | 开始扫描 |
| `diss scan stop` | 停止扫描 |
| `diss adv_update` | 更新广播内容 |

### 连接管理

| 命令 | 说明 |
|:---|:---|
| `diss conn_idx start <idx> <addr_type> <timeout> <conn_itv> <scan_itv> <wd>` | 连接指定设备 |
| `diss conn_idx addr <addr_type> <peer_addr>` | 通过地址连接 |
| `diss conn_idx cancel` | 取消连接 |
| `diss reconn` | 重连上次设备 |
| `diss show_dev` | 显示所有已连接设备 |
| `diss bond <idx>` | 绑定设备 |
| `diss delete all_bond` | 删除所有绑定 |
| `diss exch_mtu <idx>` | 发起 MTU 交换 |

### 连接参数更新

| 命令 | 说明 |
|:---|:---|
| `diss up_conn <idx> <intv_max_ms> <intv_min_ms> <latency> <timeout_ms>` | 更新连接参数 |
| `diss up_phy <idx> <phy>` | 更新 PHY（0:1M, 1:2M, 2:Coded） |
| `diss up_len <idx> <tx_octets> <tx_time>` | 更新数据长度 |

### GATT 操作

| 命令 | 说明 |
|:---|:---|
| `diss search_svc <idx> <uuid_len> <uuid>` | 搜索远程 GATT Service |
| `diss show_rmt_service <idx>` | 显示已搜索到的 Service |
| `diss read_val <idx> <handle>` | 读取远程特征值 |
| `diss write_val <idx> <handle> <len> <hex_data>` | 写入远程特征值 |
| `diss write_cccd <idx> <handle>` | 使能 CCCD（Notifiation） |
| `diss write_test <idx> <handle> <count> <len>` | 批量写入压测 |

### 系统

| 命令 | 说明 |
|:---|:---|
| `nvds reset_all 1` | 出厂化 BLE Flash 数据（首次使用建议执行） |
| `nvds update addr 6 <addr>` | 设置蓝牙 MAC 地址 |

---

## menuconfig 配置要点

编译前需在 `menuconfig` 中使能以下选项：

1. **使能蓝牙**：`BLUETOOTH`
2. **使能 GAP / GATT Client / BLE Connection Manager**
3. **使能 NVDS**
4. **使能 PingPong OTA**：`DFU` + `DFU_COMPRESS`
5. **打开 Solution OTA 宏**：`SOL_DFU`

---

## PingPong OTA 升级

### 工程配置

1. **SConstruct**：FTAB 需通过 `addchildproj` 方式添加
2. **ptab.json**：需划分与 `HCPU_FLASH_CODE_LOAD_REGION` 同等大小的 `main` + `base2` 区域作为 Pong 区域
3. **Bootloader**：需添加跳转逻辑

### 制作升级包

```bash
./imgtoolv37.exe gen_dfu \
    --img_para main 0 0 \
    --key=s01 \
    --sigkey=sig \
    --dfu_id=1 \
    --hw_ver=51 \
    --sdk_ver=7001 \
    --fw_ver=1001001 \
    --com_type=0 \
    --bksize=2048 \
    --align=0
```

- `main` — 待升级的 bin 名称（如 `hcpu` 则对应 `hcpu.bin`）
- 第一个数字参数：`16` = 压缩，`0` = 不压缩（**必须选 0**）
- 第二个数字参数：image id（hcpu 为 0）

### 手机端升级

1. 打开 **Sifli BLE App**
2. 搜索板子 BLE 广播并连接
3. 选择 **NAND DFU**
4. 先选 **ctrl** 文件，再选制作好的 **outmain.bin**
5. 点击 **Start** 开始升级

---

## 项目结构

```
Ajie_try/
├── src/
│   ├── main.c              # 核心应用代码（BLE 事件处理、GATT 回调、Finsh 命令）
│   └── SConscript           # 源码构建脚本
├── project/
│   ├── hcpu/                # HCPU（高性能 CPU）工程
│   │   ├── SConstruct        #   顶层构建入口
│   │   ├── SConscript        #   构建规则
│   │   ├── proj.conf         #   项目配置（RT-Thread + BLE + DFU）
│   │   ├── Kconfig           #   menuconfig 菜单
│   │   ├── rtconfig.py       #   RT-Thread 配置导出
│   │   └── board/            #   板级支持文件
│   ├── acpu/                # ACPU（辅助 CPU）工程
│   ├── ftab/                # Flash Table 工程
│   └── fsroot/              # 文件系统根目录（可加入固件文件系统）
├── assets/                  # 文档截图
├── LICENSE                  # 许可证
├── README.md                # 本文件
└── .gitignore
```

---

## 代码入口

| 函数 | 说明 |
|:---|:---|
| `main()` | 系统调度入口，初始化 mailbox，等待 `BLE_POWER_ON_IND`，注册 GATT Service 并开启广播 |
| `ble_app_event_handler()` | 注册到 BLE Stack 的事件回调，处理 GAP/GATT/连接等所有 BLE 事件 |
| `ble_app_gatts_get_cbk()` | GATT Server **读** 回调 |
| `ble_app_gatts_set_cbk()` | GATT Server **写** 回调（含 CCCD 处理） |
| `ble_app_gattc_event_handler()` | GATT Client 事件回调（Notify / Read Response） |
| `cmd_diss()` | Finsh Shell 命令入口 |

---

## 注意事项

1. **首次使用**：建议先执行 `nvds reset_all 1` 出厂化 Flash，避免数据冲突
2. **LCPU 镜像**：工程所需的 `lcpu_img.c` 来源于 `SDK\example\rom_bin\lcpu_general_ble_img\lcpu_img.c`，如修改目录请确保 `prebuild.bat` 能正确拷贝
3. **手机 APP 推荐**：iPhone 用 **LightBlue**，Android 用 **nRF Connect**
4. **OTA 时务必选「不压缩」**：压缩模式当前不支持
5. **广播名称**：格式为 `SIFLI_APP-<6字节蓝牙地址>`，每块板子唯一

---

## 更新记录

| 版本 | 日期 | 说明 |
|:---|:---|:---|
| 0.0.1 | 07/2025 | 初始版本 — Central + Peripheral + PingPong OTA |

---

## 许可

Copyright (c) 2021-2025 Sifli Technology. All rights reserved.

本软件仅可与 **Sifli 集成电路** 配合使用。
