# Home Assistant：从 YAML 到固件

给接手的人。按这份做，电表会用 ESPHome 原生 API 进 Home Assistant。测量实体出现一次，单位和设备类型可用，能源面板里有总正向和总反向。

这一期已经跑通的做法写在下面。现场的地址、账号、加密 key 换成你自己的。示例都是假的，不要写进 Git，也不要贴进对话。

云端 MQTT 是另一条连接，步骤在 `docs/云端MQTT.md`。两条路在同一份 `sdm01-meter.yaml`、同一份固件里。Home Assistant 走的是 API，不是 MQTT 自动发现。`mqtt.discovery` 保持 false，避免第二套实体。

一期固定 ESPHome **2026.10.2**。之后不要随手升级。

---

## 1. 做完是什么样

- 浏览器能打开 Home Assistant。
- 设置里的 ESPHome 集成有一台设备，名字来自 YAML 的 `name`（现有文件是 `sdm01-meter`，界面上显示 Sdm01-Meter）。
- 开发者工具里能筛到这台设备的实体，平台是 `esphome`。39 个测量点都在，没有第二套同名。
- 能源面板的电网来源里有总进口和总出口，并且面板上有数。进出合计不在里面。
- 加密 key 和 OTA 密码只在本机 `secrets.yaml`。

---

## 2. 安全

单独操作时不接市电。ESP 由 USB-TTL 的 3.3V 经 7 针口供电。USB-TTL 拨在 3.3V。

带电对照要导师在场：试验台有漏保和隔离变压器，USB-TTL 用光隔离型。

只刷 ESP8685。计量 MCU 不动。YAML 里不写任何 Modbus 寄存器。

`logger` 的 `baud_rate` 保持 0。确认设备从 Wi-Fi 上线后拔掉 USB-TTL。没有市电时，拔掉 TTL 设备会断电。只靠 TTL 的 3.3V 不要做 OTA。

Home Assistant 的主人账号密码、长期访问令牌，和电表的 `api_key` 一样，只留在本机。

---

## 3. 硬件和口

芯片是 ESP8685（ESP32-C3 系列），片内 Flash 4MB。

| 口 | 参数 | 接到哪里 |
|---|---|---|
| 7 针下载口 | 115200。TX = GPIO21，RX = GPIO20 | USB-TTL。板子 RXD 接 TTL 的 TX，板子 TXD 接 TTL 的 RX，共地，3.3V。只在第一次烧录和救砖时使用 |
| 板内计量串口 | 19200、8N1。TX = GPIO0，RX = GPIO1。从机地址 247 | 板上已经接到计量 MCU |
| 外壳绿色 RS485 | 9600、8N1、地址 247 | 电脑直读对照。不要写进 `uart:` |

进下载模式：市电断开，接 GND、GPIO20、GPIO21、3.3V，IO9 对 GND 短接后再上电。烧完断电，拆掉这一个短接帽，再上电。

Wi-Fi 只用 2.4 GHz。电脑、电表、Home Assistant 在能互相访问的网络上。电脑若在 5 GHz，先改到电表那个 2.4 GHz 再操作。

---

## 4. 安装 ESPHome

```powershell
py -m pip install "esphome==2026.10.2"
esphome version
```

应打印 `2026.10.2`。不在 PATH 上时用 `py -m esphome`。

第一次编译要下 RISC-V 工具链。GitHub 下不完时改乐鑫镜像，校验通过后再编译。不要升级 ESPHome 版本来绕过。

---

## 5. 安装 Home Assistant

这一期用的是局域网里已经在跑的 Home Assistant，浏览器打开 `http://<HA的IP>:8123`。机器还在，就用原来的地址和账号，从第 8 节添加设备。不要为了这份文档再装一套，否则能源面板和已有实体会对不上。

还没有 Home Assistant 时，在一台专用电脑或虚拟机上装 Home Assistant Operating System（官方镜像，按 [home-assistant.io 的安装页](https://www.home-assistant.io/installation/) 写盘或导入虚拟机）。不要装在日常办公的 Windows 系统盘上。

1. 用网线或已支持的无线把这台机器接到和电表相同的局域网。电表本身只连 2.4 GHz，Home Assistant 主机可以在有线或其他能路由到该网段的网络上。
2. 开机后等几分钟。浏览器打开 `http://homeassistant.local:8123`。打不开时，到路由器里找这台机器的地址，改用 `http://<HA的IP>:8123`。
3. 按页面创建主人账号。这一步的密码就是以后登录用的，不要写进仓库。
4. 地点和时区按电表所在地设置。能源面板按这个时区切天。
5. 初始向导里可以先不添加别的集成。ESPHome 设备等固件上线后再加。

装好的标志：能登录，设置页面打得开，左下角能看到运行中。

---

## 6. 准备 secrets

复制 `secrets.yaml.example` 为 `secrets.yaml`。它在 `.gitignore` 里。

这一路一定要有：

```yaml
wifi_ssid: "你的2.4GHz名称"
wifi_password: "你的Wi-Fi密码"
api_key: "32字节随机数的base64"
ota_password: "一长串随机密码"
```

`api_key` 生成一次：32 字节随机数，再 base64。添加设备时 Home Assistant 会向你要这串 key。已经烧进设备的固件用的是哪一串，就填哪一串。换 key 必须重新编译并 OTA，然后在集成里改成新 key。

同一份固件还连云端 MQTT，所以 `mqtt_cloud_broker` 和 `mqtt_cloud_ca` 也要在。写法、Broker 和 Topic 见 `docs/云端MQTT.md`。只做 Home Assistant、暂时没有 Broker 时，不要留下一个会一直重连的空 `mqtt:`。向导师确认能不能先拿掉 `mqtt:` 块。拿掉之后，这份文档里的 API、实体和能源面板仍然成立；和云端同时在线的验收要等 MQTT 加回来再做。

---

## 7. 写 YAML

工作文件是 `sdm01-meter.yaml`。Home Assistant 能显示什么，由这里的 `api`、实体名、`device_class`、`state_class`、单位决定。发现协议不在 Home Assistant 里再配一份传感器。

### 7.1 设备和芯片

```yaml
esphome:
  name: sdm01-meter
  comment: HA native API

esp32:
  board: esp32-c3-devkitm-1
  variant: esp32c3
  framework:
    type: esp-idf
```

`name: sdm01-meter` 会变成实体 id 前缀 `sensor.sdm01_meter_...`，mDNS 名是 `sdm01-meter.local`。一台局域网里不要有两个相同的 `name`。

云端 TLS 仍开着时，保留 `sdkconfig_options` 里的 `CONFIG_MBEDTLS_HAVE_TIME_DATE: y`，否则证书日期检查不生效。详见 MQTT 文档。

### 7.2 日志、API、OTA、Wi-Fi

```yaml
logger:
  baud_rate: 0
  level: INFO

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "SDM01-Setup"

captive_portal:
```

没有 `api:` 和加密 key，Home Assistant 的 ESPHome 集成加不上这台设备。

`baud_rate: 0` 保持。日志用 `esphome logs`，走 API，不走下载串口。

热点只在连不上 Wi-Fi 时出现，名称 `SDM01-Setup`，不设密码。

### 7.3 时间、MQTT、读表

同一份文件里还有 `time`（SNTP）、`mqtt`（8883、`discovery: false`、CA）、`uart`、`modbus`、`modbus_controller`。这些块的逐项写法和云端文档第 6 节相同，这里只列 Home Assistant 会受影响的约束：

- 计量口是 GPIO0 发送、GPIO1 接收，19200、8N1，从机 247，`update_interval: 10s`。下载口 GPIO21 / GPIO20 不要写进 `uart:`。
- `value_type: FP32`，`register_type: holding`。不要加 `multiply`，不要改成 `FP32_R`。
- `mqtt.discovery` 保持 false。

### 7.4 传感器怎么写，Home Assistant 才认

每个测量点一块。名字用 A/B/C，不用官方 MQTT 的 X/Y/Z。

```yaml
sensor:
  - platform: modbus_controller
    modbus_controller_id: meter
    name: "Voltage A"
    address: 1000
    register_type: holding
    value_type: FP32
    unit_of_measurement: V
    device_class: voltage
    state_class: measurement
```

`name` 变成界面上的友好名 `Voltage A`，实体 id 是 `sensor.sdm01_meter_voltage_a`。

39 个测量点的地址、单位、`device_class` 在 `docs/实施规格.md` 第 7 节。`sdm01-meter.yaml` 里已经按该表写完。抽查时至少看这几行：1000 Voltage A（V，voltage，measurement），1012 Active Power A（kW，power），1040 Current Unbalance（%，没有 device_class），1052 Frequency（Hz，frequency）。

电能决定能源面板里看不看得见：

| 地址 | 友好名 | 单位 | device_class | state_class | 能源面板 |
|---|---|---|---|---|---|
| 1054 | Total Import Energy | kWh | energy | total_increasing | 作为电网消耗加入 |
| 1056 | Total Export Energy | kWh | energy | total_increasing | 作为电网回馈加入 |
| 1060、1066、1072 | Import Energy A/B/C | kWh | energy | total_increasing | 不加 |
| 1062、1068、1074 | Export Energy A/B/C | kWh | energy | total_increasing | 不加 |
| 1058、1064、1070、1076 | 进出合计 | kWh | energy | 留空 | 不会出现在可选列表里，也不要加 |

没有 `state_class: total_increasing` 的电能，能源面板的来源列表里不会出现。进出合计是绝对值相加，规格要求留空。分相即使写了 `total_increasing`，面板里也只放总进口和总出口。

空闲堆：

```yaml
  - platform: debug
    free:
      name: "Heap Free"
```

它是诊断实体，不是能源来源。

不要写 `lambda`。不要加会写寄存器的 `number`、`switch`、`output`。

### 7.5 检查配置

```powershell
esphome config sdm01-meter.yaml
```

通过的标志：命令结束且没有错误；39 个测量传感器都在；1054 和 1056 有 `total_increasing`；1058 没有 `state_class`；`api.encryption.key` 来自 `!secret`，文件里看不到 key 原文。

`ota` 使用 `password` 时，config 可能警告多占 flash 和 RAM。按现有规格继续用密码。

---

## 8. 备份、编译、烧录

第一次覆盖原厂固件之前，先整片备份两次。`<COM>` 换成实际端口。

```powershell
esptool --port <COM> --baud 115200 chip-id
esptool --port <COM> --baud 115200 flash-id
esptool --port <COM> --baud 115200 --before default-reset read-flash 0 ALL backup-<MAC>-<日期>-1.bin
esptool --port <COM> --baud 115200 --before default-reset read-flash 0 ALL backup-<MAC>-<日期>-2.bin
Get-FileHash backup-<MAC>-<日期>-1.bin, backup-<MAC>-<日期>-2.bin
```

两个文件都是 4194304 字节且哈希相同。写回原厂：

```powershell
esptool --port <COM> --baud 115200 --before default-reset write-flash --flash-mode keep --flash-freq keep --flash-size keep 0 backup-<MAC>-<日期>-1.bin
```

校验通过后断电，拆掉 IO9 短接，再上电。

编译：

```powershell
esphome compile sdm01-meter.yaml
```

`esphome upload` 不会编译。改完 YAML 必须先 compile，再烧。直接 upload 会把上一份 bin 刷回去，Home Assistant 里看到的仍是旧实体。

第一次烧录：IO9 接 GND 后上电，电脑在 2.4 GHz。

```powershell
esphome compile sdm01-meter.yaml
esphome upload sdm01-meter.yaml --device <COM>
```

哈希校验通过后断电，拆掉 IO9，再上电。设备连上 Wi-Fi 后：

```powershell
esphome logs sdm01-meter.yaml
```

日志里有 `Log Baud Rate: 0` 和这次的 `compiled on`。看完关掉窗口。多个 `esphome logs` 同时开着会占满 API，Home Assistant 和后续日志都会握手失败。

确认在线后拔掉 USB-TTL。

设备已经由市电供电、并且在网上时，以后只走 OTA：

```powershell
esphome compile sdm01-meter.yaml
esphome upload sdm01-meter.yaml --device sdm01-meter.local
```

密码用 `ota_password`。传完再看一次日志，`compiled on` 必须是刚才那一次。

改完 `state_class` 之后也要重新编译并 OTA。只在 Home Assistant 里刷新页面，不会让一个没有 `total_increasing` 的实体突然出现在能源来源列表里。

---

## 9. 在 Home Assistant 里添加设备

固件已上线、API 加密开着之后：

1. 登录 `http://<HA的IP>:8123`。
2. 打开 **设置 → 设备与服务**。
3. 右下角 **添加集成**，搜索 **ESPHome**。
4. 主机填 `sdm01-meter.local`。解析失败时填电表的 IP。
5. 端口保持 ESPHome API 的默认端口。
6. 加密密钥粘贴 `secrets.yaml` 里的 `api_key`。不要勾选或改成别的发现方式。
7. 提交后，设备页出现这一台。固件版本应是 `2026.10.2`，MAC 和 `esptool chip-id` 一致。

同一台设备已经加过时，不要再添加一次。密钥改了就打开这一条集成，更新密钥。

添加成功的标志：

- 集成条目状态是已加载。
- 设备只有一台这个名字。
- 测量实体的平台是 `esphome`。

---

## 10. 核对实体

左侧边栏里没有 **开发者工具**。右上角放大镜搜索「开发者工具」也会显示 Nothing found。登录办公室这台 Home Assistant 后，在地址栏直接打开：

`http://<HA的IP>:8123/developer-tools/state`

这就是 **开发者工具 → 状态**。在列表上方的筛选框里输入 `sensor.sdm01_meter_`，不要按回车去别的页面。表格应只留下这台电表的实体。

页面提示没有权限时，当前账号不是管理员，换管理员账号再打开这个地址。

应看到 39 个测量实体，再加上空闲堆（固件里有 `Heap Free` 时）。友好名各出现一次。没有第二套同名，也没有平台是 `mqtt` 的副本。

抽查属性和规格第 7 节一致：

| 实体 | 单位 | device_class | state_class |
|---|---|---|---|
| sensor.sdm01_meter_voltage_a | V | voltage | measurement |
| sensor.sdm01_meter_current_a | A | current | measurement |
| sensor.sdm01_meter_active_power_a | kW | power | measurement |
| sensor.sdm01_meter_reactive_power_a | kVAR | reactive_power | measurement |
| sensor.sdm01_meter_apparent_power_a | kVA | apparent_power | measurement |
| sensor.sdm01_meter_power_factor_a | 无 | power_factor | measurement |
| sensor.sdm01_meter_frequency | Hz | frequency | measurement |
| sensor.sdm01_meter_current_unbalance | % | 无 | measurement |
| sensor.sdm01_meter_total_import_energy | kWh | energy | total_increasing |
| sensor.sdm01_meter_total_export_energy | kWh | energy | total_increasing |
| sensor.sdm01_meter_total_import_and_export_energy | kWh | energy | 无 |

有市电时电压大约是两百多伏，总进口是累计电量，总出口可以为 0。设备页上有时显示成整数，是因为 YAML 里还没写小数位。小数位等和绿色 RS485 对照之后再加，不要先用小数位把数量级盖住。

状态是 `unknown`：这一轮 Modbus 没有读到。从机地址或接线不对时，测量点会变成 `unknown`，空闲堆仍可能有数。不要把 Broker 上留着的旧 MQTT 数字当成 Home Assistant 的当前值。

状态长时间停在同一个时间戳：数值没变时 Home Assistant 不刷新时间。反向电能一直是 0 就会这样。看设备日志里这一路是不是仍读到 0。

筛选框没反应时，用页面自己的筛选。窗口太窄时属性列可能挤没，点开单个实体看属性。不要点 **设置状态**。

---

## 11. 能源面板

先确认总进口、总出口的 `state_class` 已经是 `total_increasing`，并且这份固件已经 OTA 上去。然后：

1. 打开 **设置 → 仪表盘 → 能源**。页面上如果已有能源配置，用配置入口编辑电网。
2. **电网 → 添加消耗**。来源选 `Sdm01-Meter Total Import Energy`（实体 `sensor.sdm01_meter_total_import_energy`）。
3. **电网 → 添加回馈**。来源选 `Sdm01-Meter Total Export Energy`（实体 `sensor.sdm01_meter_total_export_energy`）。
4. 分相进口、分相出口、四路进出合计都不加。
5. 面板上原来就有的其他电表来源不要删。这一期里另一台设备的来源留在配置里，即使实体暂时未定义。

刚加完可能还没有柱状图。统计要积累一段时间，常见是等到出现 5 分钟统计之后，第二天的面板上才有数。头一天加完是空的，先不要改 YAML、不要重刷。

第二天打开能源面板，日期选当天：来源表里能看到总进口的电量，总出口可以为 0，图上对应时段有柱。实体上的总进口是表内累计值，比面板上“这一天用了多少”大，这是正常的。

进出合计仍应不在来源表里。它在开发者工具里能看到，单位 kWh，没有 `state_class`。

---

## 12. 两条路互不影响

固件里 `api` 和 `mqtt` 同时开着，`discovery` 是 false。

只断开这一台的 ESPHome 集成时，整台 Home Assistant 仍在运行。云端或本地 8883 上应继续收到 `sdm01-meter/sensor/voltage_a/state`，载荷是纯文本数字。`status` 不必因为集成断开而变成 offline。把集成重新加载后，实体恢复更新，不用重启电表。

只停 Broker 的 8883 时，原来的 `sensor.sdm01_meter_` 实体仍应更新，平台仍是 `esphome`，不会变成另一套 MQTT 实体。Broker 恢复后也不用重启电表。

出现第二套同名实体：先看 YAML 里 `discovery` 是不是被改成 true，以及 Home Assistant 的 MQTT 集成有没有发现到这台设备。处理方式是关掉发现，而不是再加一组 sensor。

---

## 13. 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 集成添加时密钥失败 | 填的 key 不是烧进这版固件的 `api_key` | 用编译这版固件的那份 `secrets.yaml`。换 key 就要重新 OTA |
| 发现不了 `sdm01-meter.local` | 不在同一网段，或电脑在 5 GHz 且隔离了 2.4 GHz | 填 IP。电脑改到电表的 2.4 GHz |
| 设备在，测量全是 unknown | 计量口没读到。常见是 `uart` 误用了 GPIO21 / GPIO20 | 改回 GPIO0 / GPIO1，重新编译并 OTA。不要扫波特率 |
| 实体在，能源来源列表里没有总进口 | 固件里还没有 `total_increasing`，或 OTA 没换成新 bin | `esphome compile` 后再 upload。日志里的 `compiled on` 要对上 |
| 能源面板当天没有柱 | 统计还没积累 | 等到第二天再看。不要为了出图去改寄存器 |
| 日志和 Home Assistant 同时连不上 | 开着多个 `esphome logs` | 关掉多余的日志进程再试 |
| 拔掉 USB-TTL 后设备消失 | 没有市电，ESP 只靠 TTL 的 3.3V | 单独操作时这是预期的。市电上电要导师在场 |
| OTA 后实体仍是旧的 | `upload` 没先 `compile` | 重新编译再 OTA |
