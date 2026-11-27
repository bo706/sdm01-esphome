# 官方 MQTT 与一期原生 MQTT 的差距

T0.2 的产出。只记录，不实现。一期不写 JSON，不改 Topic 去对齐官方字段名。

| 项 | 内容 |
|---|---|
| 官方行为 | [MQTT API](https://docs.bituo-technik.com/device-api/spm-sdm-open-wi-fi-api-overview/mqtt-api-reference)，2026-10-17 |
| 官方字段 | [Telemetry Payload](https://docs.bituo-technik.com/device-api/spm-sdm-open-wi-fi-api-overview/telemetry-payload) 第 4.3 节，三相 `SDM01_3PN` / `SPM02`，2026-10-17 |
| 一期行为 | `docs/实施规格.md`。ESPHome 原生 MQTT，一个传感器一个 Topic，纯文本 |
| 测量点是否缺失 | 不缺。第 4.3 节的电气字段都能在地图第 3.2 节找到。差距是打包方式和名字 |

## 1. 通道

| 项目 | 官方固件 | 一期 | 处理 |
|---|---|---|---|
| 数据 | 一条 JSON，Topic `{型号}/{序列号}/data` | 每个传感器一条，`<topic_prefix>/sensor/<名字>/state`，内容是纯文本数字 | 一期不做 JSON。记入评估报告 |
| 命令 | 服务器发到 `/cmd`，设备从 `/cdata` 回复 | 没有命令通道 | 一期不做。记入评估报告 |
| 元数据 | 每次连上 Broker 发一条保留消息到 `/mdata`（产品名、序列号、固件版本、Wi-Fi 名） | 没有这条消息 | 一期不做。记入评估报告 |
| 离线检测 | 官方不设遗嘱。靠 `/data` 还在不在更新来判断 | ESPHome 自带 online/offline 遗嘱 | 用原生行为，不模仿官方 |
| 发现 | 官方 Topic 不会被 HA 的 ESPHome API 再发现一次 | `discovery: false`，避免 HA 里两套实体 | 按规格 |
| 相位名字 | `X`/`Y`/`Z` 表示 A/B/C | 实体名用 A/B/C | 一期不改成 X/Y/Z。记入评估报告 |
| 数值类型 | JSON 里测量值是字符串，例如 `"226.3"` | 纯文本数字 | 记入评估报告 |
| 小数位 | 电压 1 位，电流和功率 3 位，功率因数和频率 2 位，电能 2 位，不平衡度 0 位 | 等 RS485 直读后再定 | 不把官方小数位抄进 YAML |
| 上报周期 | 设备出厂常见 5 秒；文档另一处写 30 秒。以设备上读到的为准，最短 1 秒 | 开发阶段 10 秒 | 一期不追随官方周期。记入评估报告 |
| 断线补发 | 不补发离线期间的数据 | 同样只发当前值 | 一致，无需额外做 |

`/cmd` 能做的事（定位、清电能、重启、改 Wi-Fi、改 MQTT、OTA）一期全部不做。清电能和改通信参数也在 Modbus 写禁止范围内。

## 2. 三相字段对照

官方 `X`/`Y`/`Z` 就是地图里的 A/B/C。下表只说明“同一个量在两边叫什么”。一期 YAML 用右列的地址和实体名，不用左列的字段名。

| 官方 `/data` 字段 | 地图地址 | 一期实体名草稿 | 单位 |
|---|---:|---|---|
| `VoltageX` / `Y` / `Z` | 1000 / 1002 / 1004 | Voltage A/B/C | V |
| `CurrentX` / `Y` / `Z` | 1006 / 1008 / 1010 | Current A/B/C | A |
| `TotalCurrent` | 1042 | Total Current | A |
| `ActivePowerX` / `Y` / `Z` | 1012 / 1014 / 1016 | Active Power A/B/C | kW |
| `TotalActivePower` | 1044 | Total Active Power | kW |
| `ReactivePowerX` / `Y` / `Z` | 1018 / 1020 / 1022 | Reactive Power A/B/C | kVAR |
| `TotalReactivePower` | 1046 | Total Reactive Power | kVAR |
| `ApparentPowerX` / `Y` / `Z` | 1024 / 1026 / 1028 | Apparent Power A/B/C | kVA |
| `TotalApparentPower` | 1048 | Total Apparent Power | kVA |
| `PowerFactorX` / `Y` / `Z` | 1030 / 1032 / 1034 | Power Factor A/B/C | 无 |
| `OverallPowerFactor` | 1050 | Total Power Factor | 无 |
| `AverageVoltageL_N` | 1036 | Average Voltage | V |
| `AverageLineCurrent` | 1038 | Average Current | A |
| `UnbalanceLineCurrents` | 1040 | Current Unbalance | % |
| `Frequency` | 1052 | Frequency | Hz |
| `TotalForwardEnergy` | 1054 | Total Import Energy | kWh |
| `TotalReverseEnergy` | 1056 | Total Export Energy | kWh |
| `TotalEnergy` | 1058 | Total Import and Export Energy | kWh |
| `ForwardEnergyX` / `Y` / `Z` | 1060 / 1066 / 1072 | Import Energy A/B/C | kWh |
| `ReverseEnergyX` / `Y` / `Z` | 1062 / 1068 / 1074 | Export Energy A/B/C | kWh |
| `TotalEnergyX` / `Y` / `Z` | 1064 / 1070 / 1076 | Import and Export Energy A/B/C | kWh |

官方 JSON 里还有这些字段，它们不是地图第 3.2 节的测量点。一期不把它们拼进 `/data`。

| 官方字段 | 含义 | 一期 |
|---|---|---|
| `Post` | 报文计数，重启后回到小数字 | 不做 |
| `Time` | Unix 时间 | 设备用 SNTP 对时，但不放进这条上报 |
| `IP` | 设备 IPv4 | 不做 |
| `Rssi` | Wi-Fi 信号，dBm | 诊断用 `wifi_signal`，单独的实体，不是官方字段 |
| `FWVersion` | Wi-Fi 模块固件版本 | 不做。刷成 ESPHome 之后官方版本号也不再存在 |
| `MCUVersion` | 计量 MCU 固件版本 | T3 可只读 Modbus 地址 135，不放进官方 JSON |
| `SerialNumber` | 12 位十六进制序列号 | 一期不读可写的序列号寄存器 |
| `ProductModel` | 例如 `SDM01_3PN` | 不做 |
| `TP` | 型号代码，不是相数 | 不做 |

官方还有一套 `commonmode` 通用格式（功率改成 W，电能改成 Wh，字段名换成 `a_`/`b_`/`c_`）。一期不切换这种格式。

## 3. 评估报告里要原样写上的差距

- 单条 JSON `/data`
- 字段名 X/Y/Z，以及 `TotalEnergy` 这类官方名字
- `/mdata`、`/cmd`、`/cdata`
- 测量值是 JSON 字符串，不是每个 Topic 一个纯文本
- 上报周期：官方以设备上的配置为准（常见 5 秒），一期开发阶段是 10 秒
- 同一条报文里的 `Post`、`Time`、`IP`、版本号、型号
