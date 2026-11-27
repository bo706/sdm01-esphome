# 云端 MQTT：从 YAML 到固件

给接手的人。按这份做，电表会用 ESPHome 原生 MQTT，经 TLS 连到云端 Broker。一个传感器一个 Topic，内容是纯文本数字。

这一期已经跑通的做法写在下面。现场的 COM 口、Wi-Fi、Broker 主机名、证书换成你自己的。示例里的值都是假的，不要抄进 `secrets.yaml`，也不要贴进对话。

Home Assistant 是另一条连接，步骤在 `docs/HA接入.md`。两条路在同一份 `sdm01-meter.yaml`、同一份固件里。关掉 Home Assistant，MQTT 继续报；Broker 停了，Home Assistant 里原来的实体还在更新。

一期固定 ESPHome **2026.10.2**。之后不要随手升级。配置键以这一版的 `esphome config` 为准。

---

## 1. 做完是什么样

- 设备连上 2.4 GHz Wi-Fi。
- Broker 端口是 8883，证书校验通过。
- 订阅 `<设备名>/#` 能看到每个测量点自己的 Topic，载荷是数字，不是 JSON。
- `<设备名>/status` 在线时是 `online`。
- `discovery` 是 false。Home Assistant 里不会多出一套 MQTT 实体。
- 密钥只在本机 `secrets.yaml`。仓库里只有 `secrets.yaml.example`。

---

## 2. 安全

单独操作时不接市电。ESP 由 USB-TTL 的 3.3V 经 7 针口供电。USB-TTL 拨在 3.3V。

带电对照要导师在场：试验台有漏保和隔离变压器，USB-TTL 用光隔离型。

只刷 ESP8685。计量 MCU 不动。YAML 里不写任何 Modbus 寄存器，不清电能。

`logger` 的 `baud_rate` 保持 0。日志用 `esphome logs` 走网络，不走这块板的下载串口。刷完并确认上线后拔掉 USB-TTL。没有市电时，ESP 只靠 TTL 的 3.3V，拔掉后设备会断电。市电上电要导师在场。只靠 TTL 供电时不要做 OTA，会掉电回退。

---

## 3. 硬件和口

芯片是 ESP8685（ESP32-C3 系列），片内 Flash 4MB。下载口和计量口不是同一组脚。

| 口 | 参数 | 接到哪里 |
|---|---|---|
| 7 针下载口 | 115200。TX = GPIO21，RX = GPIO20 | USB-TTL。板子 RXD 接 TTL 的 TX，板子 TXD 接 TTL 的 RX，共地，3.3V |
| 板内计量串口 | 19200、8N1。TX = GPIO0，RX = GPIO1。从机地址 247 | 板上已经接到计量 MCU。不要把这组脚接到 USB-TTL |
| 外壳绿色 RS485 | 9600、8N1、地址 247 | 只给电脑直读对照。这组参数不要写进 `uart:` |

进下载模式：市电断开。接上 GND、GPIO20、GPIO21、3.3V。IO9 对 GND 短接后再上电。烧完断电，拆掉 IO9 短接，再上电。黄色短接帽就是 IO9 对地，只有这一个。

Wi-Fi 只用 2.4 GHz。电脑如果连在 5 GHz，先改到电表所在的那一个 2.4 GHz 网络，再编译、烧录、看日志。

---

## 4. 安装 ESPHome

在配置所在的目录打开终端。版本必须是 2026.10.2。

```powershell
py -m pip install "esphome==2026.10.2"
esphome version
```

`esphome version` 应打印 `2026.10.2`。命令不在 PATH 上时，把下面的 `esphome` 换成 `py -m esphome`。

第一次编译会下载 RISC-V 工具链。GitHub 下不完时，按乐鑫文档改成乐鑫镜像，校验通过后再编译。不要靠升级 ESPHome 来绕过。

---

## 5. 准备 secrets

复制 `secrets.yaml.example` 为 `secrets.yaml`。这个文件已在 `.gitignore` 里，不要提交。

云端这一路用到的键：

```yaml
wifi_ssid: "你的2.4GHz名称"
wifi_password: "你的Wi-Fi密码"
api_key: "32字节随机数的base64"
ota_password: "一长串随机密码"
mqtt_cloud_broker: "broker.example.test"
mqtt_cloud_ca: |
  -----BEGIN CERTIFICATE-----
  这里粘贴 Broker 公布的 CA 公钥
  -----END CERTIFICATE-----
```

`api_key` 是 32 字节随机数再做 base64，和 Home Assistant 添加设备时填的是同一个。`mqtt_cloud_ca` 只放公钥，保留 `BEGIN CERTIFICATE` 和 `END CERTIFICATE` 两行。私钥不出现在 YAML、Git 和对话里。

公司 Broker 如果还要账号，再增加两个键，并在第 6 节的 `mqtt:` 里引用。公共测试 Broker 没有账号时，不要写空的 `username` / `password`。

```yaml
mqtt_username: "mqtt-user"
mqtt_password: "mqtt-password"
```

负责人没给齐主机名、端口、CA 之前，不要自己编一套 Topic 前缀，也不要把测试 Broker 的参数写成最终云端。

`secrets.yaml` 必须和 `sdm01-meter.yaml` 放在同一个目录。ESPHome 只读这一份。文件已经在时直接改它，不要在别的目录再新建一份。用哪一种编辑器都可以。

### 5.1 第一次连上公共 Broker

这条云端不经过本机 Docker 里的 EMQX 管理页（`http://localhost:18083`）。一期公共测试连的是 `broker.emqx.io` 的 8883，没有用户名和密码。

下面从一台新电脑做起。详细的 YAML 含义在第 6 节，烧录细节在第 8 到第 11 节。

1. 安装 Python，再安装 ESPHome 2026.10.2，命令在第 4 节。另外安装 Mosquitto，确认有 `C:\Program Files\mosquitto\mosquitto_sub.exe`。芯片上还是原厂固件时，准备一条拨在 3.3V 的 USB-TTL。
2. 把含有 `sdm01-meter.yaml` 的整个目录拷到这台电脑。在这个目录里，把 `secrets.yaml.example` 复制成 `secrets.yaml`。同一目录里已经有 `secrets.yaml` 就打开那一份来改。
3. 在 `secrets.yaml` 里写上 2.4 GHz 的 Wi-Fi、`api_key`、`ota_password`，以及云端这两项：

```yaml
mqtt_cloud_broker: "broker.emqx.io"
mqtt_cloud_ca: |
  -----BEGIN CERTIFICATE-----
  证书全文
  -----END CERTIFICATE-----
```

证书用 EMQX 为 `broker.emqx.io` 公布的 CA 公钥。这一台电脑上现成的文件是桌面 `新测试\broker.emqx.io-ca.crt`。从 `-----BEGIN CERTIFICATE-----` 到 `-----END CERTIFICATE-----` 整段复制，包含这两行。不要加 `mqtt_username` 和 `mqtt_password`。

4. 打开 `sdm01-meter.yaml`，确认 `mqtt:` 和 `time:` 是下面这样。`broker` 和 `certificate_authority` 保持 `!secret`，不要把证书正文写进这个文件。

```yaml
mqtt:
  broker: !secret mqtt_cloud_broker
  port: 8883
  discovery: false
  certificate_authority: !secret mqtt_cloud_ca

time:
  - platform: sntp
    id: sntp_time
    servers:
      - ntp.aliyun.com
```

5. 在该目录打开 PowerShell，检查、编译，再烧进电表。改过 `secrets.yaml` 也要重新编译并上传，电表才会带上新的 Broker 和证书。

```powershell
cd d:\esphome
esphome config sdm01-meter.yaml
esphome compile sdm01-meter.yaml
```

芯片上还是原厂固件时，市电断开，USB-TTL 用 3.3V，IO9 接 GND 后上电，`<COM>` 换成设备管理器里的端口：

```powershell
esphome upload sdm01-meter.yaml --device <COM>
```

烧完断电，拆掉 IO9 短接，再上电。电表已经在跑 ESPHome、并且连着 2.4 GHz 时，改为：

```powershell
esphome upload sdm01-meter.yaml --device sdm01-meter.local
```

`upload` 不会编译，所以每次都先 `compile`。电表里的固件已经是这份配置时，这一步可以跳过。

6. 电表通电并连上 Wi-Fi 之后，在 PowerShell 里订阅。这条命令只让电脑查看消息，不会把电表连上云端。`<证书路径>` 换成第 3 步那个 `.crt` 文件的完整路径。

```powershell
& "C:\Program Files\mosquitto\mosquitto_sub.exe" -h broker.emqx.io -p 8883 --cafile "<证书路径>" -t "sdm01-meter/#" -v
```

大约 10 秒内出现 `sdm01-meter/status online`，以及 `sdm01-meter/sensor/voltage_a/state` 后面的数字，这条云端链路就通了。看不到新数据时，先确认电表有电、在 2.4 GHz 上，并且第 5 步的固件已经在往这个 Broker 发。

---

## 6. 写 YAML

工作文件是 `sdm01-meter.yaml`。测量点已经按规格第 7 节写好。换一台同型号电表时改设备名和 `secrets`，引脚、波特率、从机地址、寄存器不要改，除非电路图不同。

下面按块说明。每一块都要在，缺了 `esphome config` 可能仍能过，但这条路不完整。

### 6.1 设备与芯片

```yaml
esphome:
  name: sdm01-meter
  comment: 云端 MQTT 8883

esp32:
  board: esp32-c3-devkitm-1
  variant: esp32c3
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_MBEDTLS_HAVE_TIME_DATE: y
```

这几行告诉 ESPHome 固件叫什么名字、芯片是哪一种、用哪套框架编译。

- `esphome:` 是设备自身的配置块。
- `name: sdm01-meter` 是设备名。它会变成局域网里的 `sdm01-meter.local`，也是 MQTT 主题的前缀，例如 `sdm01-meter/sensor/voltage_a/state`。Home Assistant 里的实体 id 也会带上这个名字。改名之后，这些都会跟着变。
- `comment: 云端 MQTT 8883` 是一句给人看的备注，写在固件信息里。改它不会改变接线或测量，只用来标明这一版固件在做什么。
- `esp32:` 是芯片和编译方式的配置块。ESP8685 属于 ESP32-C3 这一系列，所以写在这里。
- `board: esp32-c3-devkitm-1` 是编译时套用的开发板描述，用来确定这颗芯片的引脚和 Flash 布局。板上没有这块乐鑫开发板，只是借用它的板型定义。
- `variant: esp32c3` 明确芯片系列是 ESP32-C3，避免被编成别的 ESP32。
- `framework:` 选择用哪套 SDK 来生成固件。
- `type: esp-idf` 使用乐鑫的 ESP-IDF，而不是 Arduino。这颗芯片和后面的 TLS 都按 ESP-IDF 来编译。
- `sdkconfig_options:` 是额外交给 ESP-IDF 的编译开关。
- `CONFIG_MBEDTLS_HAVE_TIME_DATE: y` 打开证书日期检查。这一版 ESP-IDF 默认不检查证书日期。这一行必须留着，时钟错误和过期证书才会连不上。

### 6.2 日志、API、OTA、Wi-Fi

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

这几行配置设备怎么打日志、怎么让 Home Assistant 连上来、怎么无线升级，以及连哪一个 Wi-Fi。

- `logger:` 是日志配置。
- `baud_rate: 0` 表示不从串口输出日志。下载口和计量口都不拿来打日志，日志改由 `esphome logs` 经网络读取。这一行一直保持 0。
- `level: INFO` 是日志详细程度。INFO 只保留正常运行信息。核对通信时可以临时改成 `DEBUG`，核对完再改回 `INFO`。
- `api:` 是给 Home Assistant 用的原生接口。Home Assistant 添加这台电表时走的就是这里，不是 MQTT。同一份固件里留着。
- `encryption:` 表示这条接口要加密。
- `key: !secret api_key` 是加密密钥。`!secret` 表示去本机 `secrets.yaml` 里取 `api_key` 的值，不把密钥写在这份 YAML 里。Home Assistant 添加设备时，加密密钥一栏就粘贴这一串，端口填 `6053`。
- `ota:` 是无线升级。设备已经在网上之后，改配置用它传新固件，不用再插 USB-TTL。
- `- platform: esphome` 表示用 ESPHome 自己的升级方式。前面的 `-` 是因为这里可以写不止一种升级方式，现在只有这一种。
- `password: !secret ota_password` 是升级密码，同样从 `secrets.yaml` 读取。密码不对，新固件传不进去。用 `password` 会多占一些 flash 和 RAM，`esphome config` 可能警告。按现有工单继续用密码，不要改成另一套加密方式，除非导师改规格。
- `wifi:` 是无线网络配置。
- `ssid: !secret wifi_ssid` 是要连接的 Wi-Fi 名称，从 `secrets.yaml` 读取。必须是 2.4 GHz。
- `password: !secret wifi_password` 是这个 Wi-Fi 的密码。
- `ap:` 是设备自己发出的热点，只在连不上上面的 Wi-Fi 时出现。
- `ssid: "SDM01-Setup"` 是这个热点的名称。这里没有写密码，所以热点是开放的。按需才打开热点这一期不实现。
- `captive_portal:` 打开配网页。手机连上 `SDM01-Setup` 后，会弹出页面让你重填 Wi-Fi。

### 6.3 MQTT

```yaml
mqtt:
  broker: !secret mqtt_cloud_broker
  port: 8883
  discovery: false
  certificate_authority: !secret mqtt_cloud_ca
```

这几行是设备连云端 Broker 的最低配置。

- `mqtt:` 是 MQTT 连接配置块。测量值由 ESPHome 按原生规则发出，这里不手写每一条 Topic。
- `broker: !secret mqtt_cloud_broker` 是 Broker 的主机名，从 `secrets.yaml` 读取。这里写域名，并且要和证书上的名字一致。一期公共测试用的主机名是 `broker.emqx.io`。
- `port: 8883` 是 TLS 端口，最终固件固定用这个端口。1883 只在早期本地明文试验里用过，不要刷回最终固件。
- `discovery: false` 关闭 Home Assistant 的 MQTT 自动发现。从第一份 MQTT 配置就这样写。写成 `true` 时，Home Assistant 的 MQTT 集成会再发现一套同名实体。
- `certificate_authority: !secret mqtt_cloud_ca` 是校验 Broker 用的 CA 公钥，从 `secrets.yaml` 读取。只放公钥，私钥不写进来。

这里不写 `topic_prefix` 时，主题前缀就是上面的 `esphome.name`。负责人给了前缀才加一行 `topic_prefix: !secret mqtt_topic_prefix`。公司 Broker 要求账号时才加：

```yaml
  username: !secret mqtt_username
  password: !secret mqtt_password
```

不要写自定义发布、JSON 拼接，也不要为了凑官方 `/data` 去改 Topic。官方单条 JSON、字段名 X/Y/Z、`/cmd`、`/mdata` 都是一期不做的差距，见 `docs/官方MQTT差距对照.md`。

### 6.4 时间

```yaml
time:
  - platform: sntp
    id: sntp_time
    servers:
      - ntp.aliyun.com
```

这几行让设备从网络对时。TLS 校验证书日期之前要先有正确时间，所以这块要留着。

- `time:` 是时钟配置块。
- `- platform: sntp` 表示用 SNTP 对时。前面的 `-` 是因为这里可以写不止一个时间源，现在只有这一个。
- `id: sntp_time` 是这个时钟在配置里的名字，别的配置块要引用它时用这个名字。
- `servers:` 是对时服务器列表。
- `- ntp.aliyun.com` 是当前用的服务器。网络环境不同时可以改成别的 NTP 地址，这一行本身要留着。

### 6.5 读表

```yaml
uart:
  id: modbus_uart
  tx_pin: GPIO0
  rx_pin: GPIO1
  baud_rate: 19200
  data_bits: 8
  parity: NONE
  stop_bits: 1

modbus:
  id: modbus_hub
  uart_id: modbus_uart
  role: client

modbus_controller:
  - id: meter
    address: 247
    modbus_id: modbus_hub
    update_interval: 10s
```

这几行把 ESP 配成计量 MCU 的 Modbus 主站，从板内串口读表。

- `uart:` 是串口配置。这里用的是板内计量口，不是 7 针下载口。
- `id: modbus_uart` 是这条串口在配置里的名字，下面的 `modbus` 用它来指定走哪一个口。
- `tx_pin: GPIO0` 是 ESP 的发送脚，电路图上网名是 `ESP_IO0_U1_TX`。GPIO21 是下载口，写到这里会读不到数。
- `rx_pin: GPIO1` 是 ESP 的接收脚，电路图上网名是 `ESP_IO1_U1_RX`。GPIO20 同样是下载口。
- `baud_rate: 19200` 是计量口的速率。外壳绿色 RS485 是 9600，那组参数只给电脑直读对照，不要抄进 `uart:`。读不通时不要改这个数，也不要改从机地址。
- `data_bits: 8` 是每个字节 8 位数据。
- `parity: NONE` 是无校验。
- `stop_bits: 1` 是 1 位停止位。这三行合起来就是 8N1。
- `modbus:` 是 Modbus 协议配置。
- `id: modbus_hub` 是这条总线在配置里的名字。
- `uart_id: modbus_uart` 表示 Modbus 走上面那条串口。
- `role: client` 表示 ESP 是主站，先发出读请求。计量 MCU 是从站，只回答。
- `modbus_controller:` 是总线上的一台从站设备。
- `- id: meter` 是这台电表在配置里的名字。前面的 `-` 是因为一条总线可以挂多台从站，现在只有这一台。传感器用 `modbus_controller_id: meter` 指向它。
- `address: 247` 是从机地址。读不通时不要扫地址。
- `modbus_id: modbus_hub` 表示这台从站挂在上面那条 Modbus 总线上。
- `update_interval: 10s` 是轮询间隔。开发阶段用 10 秒，每个测量点大约每 10 秒读一次。

每个测量点再写一块传感器。电压的写法：

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

- `sensor:` 是传感器列表。Home Assistant 的实体和 MQTT 的一条条 Topic 都从这里来。
- `- platform: modbus_controller` 表示这个值从 Modbus 从站读出来。前面的 `-` 是因为后面还有很多个测量点，每个点一块。
- `modbus_controller_id: meter` 指向上面那台从站地址 247 的电表。
- `name: "Voltage A"` 是显示名。MQTT 主题里会变成 `voltage_a`，Home Assistant 实体 id 里也会带上这个名字。相位用 A/B/C。
- `address: 1000` 是地图上的寄存器地址，十进制，从 0 数起。A 相电压是 1000。一个浮点数占两个寄存器，下一个测量点的地址加 2。
- `register_type: holding` 表示读保持寄存器，功能码 03。
- `value_type: FP32` 表示这两个寄存器是大端 FLOAT32，高字在前。地图就是这种字节序。不要改成 `FP32_R`，也不要加 `multiply`。
- `unit_of_measurement: V` 是单位。功率用 kW / kVAR / kVA。功率因数不写单位。
- `device_class: voltage` 告诉 Home Assistant 这是电压。电流不平衡度不写 `device_class`。
- `state_class: measurement` 表示这是瞬时测量，会上下变化。电能的写法见下面的表。

地址从 1000 到 1076，步长 2，一共 39 个。名单在 `docs/实施规格.md` 第 7 节，实体已经在 `sdm01-meter.yaml`。

电能分三种：

| 地址 | 实体 | state_class |
|---|---|---|
| 1054、1060、1066、1072 | 正向 | `total_increasing` |
| 1056、1062、1068、1074 | 反向 | `total_increasing` |
| 1058、1064、1070、1076 | 进出合计 | 留空 |

进出合计是绝对值相加。正向和反向只会增加，重启不会清零。清零只会发生在写入清电能寄存器时，一期禁止写。

再加官方的空闲堆，方便看设备还活着：

```yaml
  - platform: debug
    free:
      name: "Heap Free"
```

- `- platform: debug` 表示这个值来自 ESPHome 自带的调试组件，不是 Modbus 寄存器。
- `free:` 读取剩余的空闲堆内存。
- `name: "Heap Free"` 是显示名。MQTT 主题是 `sdm01-meter/sensor/heap_free/state`，载荷是剩余字节数。

不要加 `lambda`、`custom_pdu`、`web_server`。`esp32_improv` 试过，空闲堆掉得太多，不加。

### 6.6 检查配置

```powershell
esphome config sdm01-meter.yaml
```

通过后再编译。报错就改配置。不要加 lambda 绕过。

核对：传感器 39 个加空闲堆；抽查 1000 是 Voltage A，1054 是 Total Import Energy 且 `state_class: total_increasing`，1058 没有 `state_class`；`uart` 是 GPIO0 / GPIO1、19200；`mqtt` 端口 8883、`discovery: false`。

---

## 7. 配置 Broker

设备侧只认主机名、8883 和 CA 公钥。Broker 侧把 TLS 监听准备好，证书上的名字必须和 `mqtt_cloud_broker` 里的主机名一致。用 IP 去连、证书上只有域名，握手会失败。

### 7.1 公共 EMQX（一期实际用的测试云）

导师指定用公共 EMQX，不再等公司 Broker。

- 主机名 `broker.emqx.io`
- 端口 8883
- CA 用 EMQX 公布的公钥。烧录前从电脑用这份 CA 连一次 8883，证书校验通过，证书名是 `*.emqx.io`
- 没有账号。YAML 里不要带用户名和密码
- 不写 `topic_prefix`

这个 Broker 上的主题任何人都能订到。它只用于把原生 Topic 跑通。公司 Broker 的主机名、账号、CA、`topic_prefix` 到了以后，只改 `secrets.yaml`，Topic 的形状不要改。

### 7.2 公司或自建 Broker

向负责人要齐这几项再改最终固件：主机名、端口（8883）、要不要账号、CA 公钥、有没有规定 `topic_prefix`。

自建 Mosquitto 时，8883 监听使用服务端证书，设备用对应的 CA 公钥校验服务端。证书的 CN 或 SAN 写成设备配置里的那个主机名。设备不向 Broker 出示客户端证书。

本地 1883 明文只用于确认 Topic 形状，确认后换成 8883，不要把 1883 留在最终固件里。

烧录前在电脑上先连一次，确认证书校验通过，再把同一份 CA 放进 `mqtt_cloud_ca`。

### 7.3 Topic 和报文

不写 `topic_prefix` 时，前缀是设备名。`Voltage A` 会变成 `voltage_a`。

| Topic | 载荷 | 含义 |
|---|---|---|
| `sdm01-meter/sensor/voltage_a/state` | `220` 这种纯文本 | A 相电压。电流、功率、电能同样是 `<名字>/state` |
| `sdm01-meter/sensor/heap_free/state` | 字节数 | 空闲堆 |
| `sdm01-meter/status` | `online` 或 `offline` | ESPHome 自带的上线和遗嘱 |
| `sdm01-meter/debug` | 日志文本 | `debug` 组件。测量值不在这里 |

一期公共 Broker 上订到过 42 个主题：39 个测量点、空闲堆、`debug`、`status`。测量载荷里没有 JSON，没有一组 X/Y/Z 字段。数值大约每 10 秒一条，和 `update_interval` 一致。数值没变时，Broker 上可能还留着上一条保留消息，看时间戳，不要把旧数当成新测量。

---

## 8. 备份原厂固件

第一次覆盖之前做。做两次，两份必须一致。离开工位时再留一份副本。

IO9 接 GND 后上电，进入下载模式。`<COM>` 换成设备管理器里的端口，这一台当时是 <下载口>。

```powershell
esptool --port <COM> --baud 115200 chip-id
esptool --port <COM> --baud 115200 flash-id
esptool --port <COM> --baud 115200 --before default-reset read-flash 0 ALL backup-<MAC>-<日期>-1.bin
esptool --port <COM> --baud 115200 --before default-reset read-flash 0 ALL backup-<MAC>-<日期>-2.bin
Get-FileHash backup-<MAC>-<日期>-1.bin, backup-<MAC>-<日期>-2.bin
```

两个文件都是 4194304 字节，哈希相同，才算备份完成。探针松动会在中途断开，按住后整段重读，不要把半截文件当成备份。

需要写回原厂时：

```powershell
esptool --port <COM> --baud 115200 --before default-reset write-flash --flash-mode keep --flash-freq keep --flash-size keep 0 backup-<MAC>-<日期>-1.bin
```

`Hash of data verified` 之后断电，拆掉 IO9 短接，再上电。原厂起来后绿灯常亮即可。不要为了验证去写 Modbus。

---

## 9. 编译

```powershell
esphome compile sdm01-meter.yaml
```

成功后固件在 `.esphome/build/sdm01-meter/.pioenvs/sdm01-meter/firmware.bin`（ESP-IDF 这一版的产物名字以编译日志最后打印的路径为准）。看日志里的编译时间，后面烧进去的必须是这一次的产物。

`esphome upload` 不会编译。改完 YAML 若直接 upload，刷上去的是上一份 bin。每次改完都先 `compile`，再烧。也可以用 `esphome run sdm01-meter.yaml`，它会先编译再上传。

---

## 10. 第一次串口烧录

1. 市电断开。USB-TTL 用 3.3V。
2. 7 针：GND、板 RXD（GPIO20）接 TTL TX、板 TXD（GPIO21）接 TTL RX、3.3V。
3. IO9 接 GND，然后上电。
4. 确认电脑在 2.4 GHz，并且 `secrets.yaml` 里的 Wi-Fi 是这个网络。

```powershell
esphome compile sdm01-meter.yaml
esphome upload sdm01-meter.yaml --device <COM>
```

看到哈希校验通过后断电，拆掉 IO9 短接，再上电。

设备会连 Wi-Fi。电脑先连到电表所在的那个 2.4 GHz，再用下面两种办法里的一种确认它在线上，并核对是不是刚才那块板。

用设备名确认。在 PowerShell 里执行：

```powershell
ping sdm01-meter.local
```

有回复就说明电表已经连上 Wi-Fi。记下回复里的 IP，再执行：

```powershell
arp -a
```

在这台 IP 那一行看物理地址。Windows 写成 `<电表MAC>` 这种带横线的形式。

`ping` 解析不了 `sdm01-meter.local` 时，打开路由器管理页，在已连接设备或 DHCP 列表里找名称 `sdm01-meter`。那里会有它的 IP 和 MAC。

烧录前执行 `esptool --port <COM> chip-id` 时，输出里有一行 MAC。这一块板记下的是 `<电表MAC>`。冒号和横线只是写法不同，逐段相同就是同一块板。对上之后再看日志：

```powershell
esphome logs sdm01-meter.yaml
```

日志经 API 过来，里面有 `Log Baud Rate: 0`，以及这次的 `compiled on` 时间。串口上不该有业务日志。看完关掉这个窗口。多开几个 `esphome logs` 会占满设备的 API 连接，后面的握手会被立刻关掉。

确认在线后拔掉 USB-TTL。没有市电时设备会随 TTL 一起断电，这是预期的。

---

## 11. 之后改配置：OTA

设备已经在电网上、并且连着 Wi-Fi 时，不再插 USB-TTL。

```powershell
esphome compile sdm01-meter.yaml
esphome upload sdm01-meter.yaml --device sdm01-meter.local
```

上传用的是 `ota_password`。密码错会被拒绝。上传中断后，重新上电再执行一次同样的 compile 和 upload。

上传后立刻 `esphome logs`，核对 `compiled on` 是刚才那一次。对不上就说明刷进去的仍是旧 bin，重新 compile 再 OTA。不要用 `esphome upload` 单独补刷。

只改 `comment` 也可以做一次 OTA，用来确认这条升级路是通的。

---

## 12. 订阅验证

电脑要能访问同一个 Broker。这条命令只订阅，电表要已经通电、连上 Wi-Fi，并且固件里烧进了同一个 Broker 和 CA。CA 用放进 `mqtt_cloud_ca` 的那一份公钥。

公共 Broker 在这一台上可以这样看。`<证书路径>` 换成 `mqtt_cloud_ca` 对应的那个 `.crt` 文件：

```powershell
& "C:\Program Files\mosquitto\mosquitto_sub.exe" -h broker.emqx.io -p 8883 --cafile "<证书路径>" -t "sdm01-meter/#" -v
```

换 Broker 时改成：

```powershell
mosquitto_sub -h <broker主机名> -p 8883 --cafile ca.pem -t "sdm01-meter/#" -v
```

没有账号就不要加 `-u` / `-P`。公司 Broker 有账号时再加上。

大约等一个轮询周期（10 秒）以上。应看到：

- `sdm01-meter/status` 为 `online`
- `sdm01-meter/sensor/voltage_a/state` 是数字，有市电时大约在 200 到 250
- `sdm01-meter/sensor/total_import_energy/state` 是数字，单位在实体上是 kWh，Topic 载荷里只有数
- 没有 `{` 开头的 JSON
- 没有 `homeassistant/` 发现主题

也可以用 MQTT Explorer，连 8883 并带上同一份 CA，订阅 `sdm01-meter/#`，留一张截图。

有市电时，再让导师用绿色 RS485（9600、8N1、地址 247）读地址 1000。两边的符号、单位、数量级一致即可。差零点几伏，多半是两次采样的时间差。差一个数量级就停，不要改字序去凑 220。

---

## 13. 连不上时怎么看

设备会重试，不应崩溃。Home Assistant 里原来的 API 实体还在。用 `esphome logs` 看 MQTT 行。

| 现象 | 日志里常见什么 | 怎么处理 |
|---|---|---|
| CA 不对 | 握手失败。esp-tls `0x801a`，mbedtls `-0x2700` | 换成 Broker 公布的那一份公钥，重新编译并 OTA |
| 时钟不对 | 同样是 `-0x2700`。证书还没到生效日，Broker 侧是 certificate unknown | 先让 SNTP 对上当前时间。时钟已经在跑时，只把 NTP 指到一个到不了的地址不会把时间清掉，所以这种改法看不出失败 |
| 主机名和证书不一致 | 同样是 `-0x2700` | `broker` 写成证书上的域名，不要写成 IP |
| 日期检查没编进去 | 错误时间和过期证书都不失败 | 确认 `CONFIG_MBEDTLS_HAVE_TIME_DATE: y`，重新编译。只改 yaml 不编译就 upload，固件里仍然没有 |
| 过期证书 | 一期没有做成这条失败试验 | 找导师补。不要为了试这个去改正在跑的最终固件，除非这一轮验收明确要重做 |
| `No PING_RESP` 后 TCP 断开 | `status` 短暂 `offline`，下一秒附近回到 `online` | 公共网络上出现过，设备自己重连。测量 Topic 随后继续更新 |
| 订阅到的数一直不变 | 保留的是上一条。设备其实已经 `offline` | 看 `status` 和新消息的时间。ping 不通、`status` 是 offline 时，订到的电压是旧数 |

四种反面情况（错误 CA、过期证书、错误时间、域名不一致）都应当连不上，同时设备不崩溃，Home Assistant 的 API 仍在。过期证书这一条还没做完。

---

## 14. 和 Home Assistant 的关系

`api:` 和 `mqtt:` 同时开着。验收时可以只停这一台的 ESPHome 集成：MQTT 仍应收到 `sdm01-meter/sensor/voltage_a/state`。只停 8883 时，Home Assistant 里原来的实体仍应更新。恢复时不用重启电表。

`discovery: false` 时，开发者工具里这台设备的实体平台都是 `esphome`。出现第二套同名实体，先查发现是不是被打开了。

Home Assistant 怎么装、怎么加集成、能源面板加哪两个实体，写在 `docs/HA接入.md`。
