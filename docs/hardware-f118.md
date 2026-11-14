# JC3636F118（ESP32-S31 旋钮屏）硬件实测记录

> 实测日期：2026-11-14　硬件：GUITION JC3636F118C（铝合金 CNC 外壳，含电池）
> 工具：ESP-IDF v6.1（`--preview`，target `esp32s31`）+ 自写探测固件
> 原厂固件备份：`C:\Users\xiaosiqi\Documents\esp32\backup_f118_vendor_16MB.bin`（16 MB）

## 1. 主控与存储

| 项目 | 值 | 来源 |
|---|---|---|
| 芯片 | ESP32-S31（RISC-V 双核 320 MHz + LP 核）| esptool / 实测 |
| MAC | `1c:29:04:d0:6b:20` | esptool |
| Flash | 16 MB（GD25Q128，GD 68/4018）| esptool flash-id |
| PSRAM | **16 MB，八线 200 MHz** | 实测 `heap_caps_get_total_size(SPIRAM)` = 16384 KB |
| 内部 RAM | 启动后剩余约 414 KB | 实测 |
| 原厂固件 | `PROJECT_N7_IDF`，`v6.1-dev-6940`，2026-10-08 | 读 app 描述符 |
| USB | 原生 USB-Serial/JTAG（VID 303A PID 1001）| 设备管理器 |

## 2. 外设实测结论

| 外设 | 状态 | 细节 |
|---|---|---|
| 屏幕 ST77916 | ✅ | 360×360 QSPI，实测可正常刷红/绿/蓝/白 |
| 触摸 CST816S | ✅ | I2C 地址 `0x15`，芯片 ID 读回 `0xB6` |
| 触觉 DRV2605L | ✅ | I2C 地址 `0x5A`，**与触摸共用一条 I2C** |
| TF 卡 | ✅ | 480 MB，卡名 `APPSD`，内含 `mjpeg/ music/ pic/ fonts/ …` 演示内容 |
| QMI8658C IMU | ❌ **未贴片** | GPIO42/43 无任何应答；原理图标注 "Chip shipment without installation" |
| 编码器 | ✅ | GPIO6/7，内部上拉，静止高电平 |
| 蜂鸣器 | 已贴装（原理图）| GPIO44 |
| 音频 | PCM5100A 线路输出 + 3.5 mm 座，**无喇叭、无功放** | 原理图 |

## 3. 引脚表（实测可用）

### 屏幕（QSPI，ST77916）
```
SCLK 12   CS 13   D0 14   D1 15   D2 16   D3 17   RST 18   TE 19
背光 GPIO5（LEDC，5 kHz，10 bit，高电平点亮）
参数：360×360，RGB565 大端，INVON，QSPI 80 MHz（40 MHz 更稳）
```

### 触摸 / 触觉（共用 I2C）
```
SDA 11   SCL 10   TP_INT 8   TP_RST 9
CST816S @0x15，DRV2605L @0x5A
```

### TF 卡（SDMMC）
```
CLK 24   CMD 25   D0 20   D1 21   D2 22   D3 23
★ 必须用 SDMMC_HOST_SLOT_0（用 SLOT_1 会 timeout 0x107）
```

### 其他
```
IMU I2C：SDA 42 / SCL 43（器件未贴装）
电池 ADC：GPIO55，15k/10k 分压（待调通）
编码器：A=7  B=6（BSP 定义；原理图标签顺序相反）
PDM 麦克风：CLK 39 / DATA 37
I2S 输出：BCLK 53 / LRCK 51 / DOUT 52
```

## 4. 踩过的坑（重要）

1. **QSPI 命令必须打包成 32 位**：
   `cmd32 = (0x02 << 24) | (cmd << 8)`，像素数据用 `(0x32 << 24) | (0x2C << 8)`。
   直接发命令字节 → 屏幕全黑但背光亮。
2. **CASET/RASET 参数必须是 4 字节数组**：`{x0_hi, x0_lo, x1_hi, x1_lo}`，
   传 `int[4]` 只会发出第一个 int 的 4 个字节 → 窗口错乱。
3. **像素数据要转大端**（RGB565_BE）。
4. **SD 卡用 SLOT_0**，不是 SLOT_1；且要显式 `cd = NO_CD, wp = NO_WP`（否则报 GPIO38 冲突）。
5. **USB 供电余量小**：背光 60% + QSPI 80 MHz 时曾导致 USB 掉线；
   降到 30% / 40 MHz 后稳定。
6. 触摸与触觉共用 I2C，读写触摸时不要把总线锁死，否则震动也失效。

## 5. 分区表（原厂）

```
nvs      0x009000    20 KB
otadata  0x00E000     8 KB
app0     0x010000  4096 KB
app1     0x410000  4096 KB
spiffs   0x810000  2048 KB
assets   0xA10000  6016 KB
coredump 0xFF0000    64 KB
```

## 6. 恢复原厂固件

```
esptool --port COM9 write-flash 0 backup_f118_vendor_16MB.bin
```

---

## 7. 厂家资料（重要来源，勿丢）

**厂家 BSP 仓库**：`https://github.com/guitionofficial/Guition`
板级目录：`components/board/guition_boards/jc3636f118/`

本地参考克隆：`C:\Users\xiaosiqi\Documents\esp32\vendor-guition`（只读参考，56 MB）

从厂家仓库用到的东西：

| 文件 | 用途 |
|---|---|
| `board_pins.h` | 引脚定义（屏幕/触摸/编码器/电池/背光）|
| `lcd_init_commands.h` | ST77916 初始化命令表（本工程 `main/lcd_init_commands.h` 即来源于此）|
| `board_peripherals.yaml` | 外设参数：I2S 音频、PDM 麦克风、SPI 屏、LEDC 背光 |
| `board_devices.yaml` | 设备参数：`lcd_cmd_bits: 32`、`data_endian: BIG`、ST77916 驱动版本 |
| `sdkconfig.defaults.board` | 板级 sdkconfig 默认值 |

> 结论：**音频、编码器、电池引脚与厂家 BSP 完全一致**（见第 8 节），
> 说明按原理图推导 + 实测的结论是对的，可以作为可信基线。

## 8. 音频（PCM5100A 耳机孔 + PDM 麦克风）

原理图（`6-Schematic diagram/JC3636F118_DAC&LCD.pdf`、`JC3636F118_GPIO.pdf`）与厂家
`board_peripherals.yaml` 互相印证：

| 功能 | 网络名 | GPIO | 外设 |
|---|---|---|---|
| 耳机孔 DAC 位时钟 | `I2S_DAC_BCK` | **53** | I2S 标准模式 TX（master）|
| 耳机孔 DAC 数据 | `I2S_DAC_DIN` | **52** | 同上 |
| 耳机孔 DAC 声道 | `I2S_DAC_LRCK/WS` | **51** | 同上 |
| 麦克风时钟 | `PDM_MIC_SCK` | **39** | I2S PDM RX（**必须在 I2S0**，只有 port 0 有 PDM2PCM）|
| 麦克风数据 | `PDM_MIC_DATA` | **37** | 同上 |

要点：

- PCM5100A 是**纯 DAC**，没有 I2C 控制、没有 mute 脚 → `voice_audio_mute()` 是空实现。
- 输出是**线路电平**（3.5 mm），没有板载喇叭。
- 厂家 `dn_sample_mode` 用 `I2S_PDM_DSR_8S`，本工程用 `DSR_16S`；两者都能工作，
  自检实测峰值 1386 / RMS 18076（有环境声即正常，峰值 < 200 说明麦克风没接好）。

## 9. 小智 AI 语音（接官方服务端）

链路：`PDM 麦克风 → Opus 编码 → WebSocket 上行 → 服务端 → Opus 解码 → PCM5100A`

| 项 | 值 |
|---|---|
| OTA 发现 | `POST https://api.tenclass.net/xiaozhi/ota/` → 返回 `wss://api.tenclass.net/xiaozhi/v1/` |
| 握手头 | `Authorization`（若服务端给 token）、`Protocol-Version: 2`、`Device-Id`、`Client-Id` |
| 二进制帧（v2，**大端**）| `version(2) type(2) reserved(4) timestamp(4) size(4) payload`，`type` 0=Opus / 1=JSON |
| 上行 | **16 kHz** 单声道 **60 ms**（960 采样）|
| 下行 | **24 kHz** 单声道 60 ms（1440 采样）|
| 控制消息 | `listen.start/stop/detect`、`tts.start/stop`、`abort` |
| 半双工 | 播放 TTS 时不录音（本板无 AEC 参考通道）|

**必须声明 `Protocol-Version: 2`**：服务端会据此回 `version:2`；用 v1 时帧头不匹配。
如果日志出现「服务端二进制帧没有 v2 头」，把 `xiaozhi_client.c` 的 `XZ_PROTO_VER` 改回 1。

开关与地址由 **switchyard 布局 JSON** 的 `voice.enabled` / `voice.endpoint` 下发生效
（设备端 `layout_parse()` → `xiaozhi_client_configure()`），不需要单独命令。

## 10. 内存预算（ESP32-S31，内部 RAM 约 512 KB）

这台设备同时跑「LVGL + WiFi + MQTT + TLS(HA) + Opus 语音 + HTTP 配置页」，内部 RAM 很紧。
踩过的坑与最终配置：

| 项目 | 值 | 为什么 |
|---|---|---|
| `CONFIG_LV_MEM_SIZE` | **49152** | 128 KB 时内部 RAM 不足，SPI 分不到 DMA 缓冲，屏幕冻结 |
| `CONFIG_ESP_MAIN_TASK_STACK_SIZE` | **12288** | LVGL 主循环 + 网络 + cfg 处理，默认 3.5 KB 会栈溢出 |
| HTTP 服务任务栈 | **6144** | 旧工程实测：4 KB 在 `/save` 上栈溢出 |
| HTTP `max_open_sockets` | **2** | 每个 socket 占 lwIP 缓冲；4 个时 TLS 握手失败 |
| Opus 发送任务栈 | **24576（PSRAM）** | CELT 编码器极吃栈，8 KB 直接栈溢出崩溃 |
| Opus 播放任务栈 | **16384（PSRAM）** | 同上，解码 |
| 音频缓冲 | PSRAM | 内部 RAM 只留必要的 DMA 缓冲 |

**症状对照**：日志出现 `HTTP_CLIENT: Connection failed, sock < 0` 或
`ESP_ERR_HTTP_CONNECT (internal heap 30 KB)` = 内部 RAM 被挤爆，TLS 握手无法分配，
表现为 HA 拉取失败 + 小智发现失败 + 屏幕卡住。

## 11. 网页配置页（无需重刷改配置）

WiFi 连上后启动 HTTP 服务，浏览器打开设备 IP（IP 显示在屏上系统页）：

| 地址 | 功能 |
|---|---|
| `http://<ip>/` | 配网页：Wi-Fi SSID/密码、HA 地址/Token、MQTT、设备 id、网页账号 |
| `http://<ip>/save` | POST 保存到 NVS 并重启 |
| `http://<ip>/ota` | 网页上传固件升级 |
| `http://<ip>/ota/upload` | POST 原始 `.bin` → 写入另一分区 → 重启 |

- 默认账号 **admin / admin**（`secrets.h` 的 `APP_SECRET_WEB_USER/PASS`），首次开机日志会提示改掉。
- 配置存 **NVS**，`secrets.h` 只在 NVS 为空时作为首次开机默认值。
- ⚠️ **没有配网热点兜底**：本工程是 STA-only，若网页里填错 WiFi，只能重新烧写恢复。

## 12. 电池

原理图与厂家 BSP 都确认 **`BATT_ADC → GPIO55`**，分压 15k/10k。

实测该脚读数 **饱和（约 3.34 V，换算成 8.3 V）** —— 这是**引脚悬空**的表现，
也就是**电池没插**（1.25 mm 座子空着）。插上电池后应落到 3.0–4.3 V，
固件已做合理性判断，超出范围不显示，避免误导。
