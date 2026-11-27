# SDM01 ESPHome

Bituo SDM01 电表里的 ESP8685，用 ESPHome 官方组件和 YAML 同时做两件事：用原生 API 接入 Home Assistant，用原生 MQTT 经 TLS 上报每个测量点。两条路在同一份固件里，互不依赖。

固定 **ESPHome 2026.10.2**。不要升级。配置键以这一版的 `esphome config` 为准。

## 仓库里有什么

| 路径 | 内容 |
|---|---|
| `sdm01-meter.yaml` | 正在使用的设备配置 |
| `secrets.yaml.example` | 密钥模板，里面都是假值 |
| `docs/云端MQTT.md` | 云端 MQTT：YAML、编译、烧录、订阅 |
| `docs/HA接入.md` | Home Assistant：原生 API、实体、能源面板 |
| `docs/确认表.md` | 硬件和参数的确认记录 |
| `docs/工单.md` | 一期任务记录 |
| `发给AI/` | 给另一个 AI 的项目包，内容和上面对应 |

现场地址、Wi-Fi、API 密钥、OTA 密码和 CA 只放在本机 `secrets.yaml`。这个文件、`.esphome/` 编译目录、`backup/` 和 `*.bin` 都在 `.gitignore` 里。

## 固件在做什么

- 设备名 `sdm01-meter`。局域网名称是 `sdm01-meter.local`。
- ESP8685 做 Modbus 主站。计量口是 GPIO0 发送、GPIO1 接收，19200、8N1，从机地址 247。功能码 03，数值类型 FP32。
- 保持寄存器 1000 到 1076，步长 2，共 39 个测量点。另有一个空闲堆传感器。轮询间隔 10 秒。
- Home Assistant 走加密的原生 API。实体带单位和设备类型。能源面板只用总正向、总反向电能。
- MQTT 走 8883 和 CA 校验，不带用户名，`discovery: false`。不写 `topic_prefix` 时，主题前缀就是设备名。`Voltage A` 对应 `sdm01-meter/sensor/voltage_a/state`，载荷是纯文本数字。在线时 `sdm01-meter/status` 是 `online`。
- 连不上 Wi-Fi 时开放无密码热点 `SDM01-Setup`，并打开 `captive_portal`。
- 时钟用 `ntp.aliyun.com`。打开了证书日期检查，时间不对时 TLS 会失败。

YAML 里不写 Modbus 寄存器，没有 lambda，没有自定义组件，没有 `web_server`。

## 开始

在本目录打开终端。

```powershell
py -m pip install "esphome==2026.10.2"
esphome version
copy secrets.yaml.example secrets.yaml
```

`esphome version` 应打印 `2026.10.2`。命令不在 PATH 上时，把 `esphome` 换成 `py -m esphome`。

然后编辑 `secrets.yaml`，填入自己的 Wi-Fi、API 密钥、OTA 密码、Broker 主机名和 CA。示例值不能直接用。云端步骤见 `docs/云端MQTT.md`，Home Assistant 步骤见 `docs/HA接入.md`。

改完配置后先编译再烧录。`esphome upload` 不会编译，它烧的是上一次编出来的固件。

```powershell
esphome compile sdm01-meter.yaml
esphome upload sdm01-meter.yaml --device COM口
```

第一次烧录要把板子放进下载模式。之后优先用 OTA。

## 硬件

芯片是 ESP8685（ESP32-C3 系列），片内 Flash 4MB。下载口和计量口不是同一组脚。

| 口 | 参数 | 接到哪里 |
|---|---|---|
| 7 针下载口 | 115200。TX = GPIO21，RX = GPIO20 | USB-TTL。交叉连接，共地，3.3V |
| 板内计量串口 | 19200、8N1。TX = GPIO0，RX = GPIO1。地址 247 | 已经接到计量 MCU，不要接到 USB-TTL |
| 外壳绿色 RS485 | 9600、8N1、地址 247 | 只给电脑直读对照。不要写进 `uart:` |

进下载模式：市电断开，接上 GND、GPIO20、GPIO21、3.3V，把 IO9 对 GND 短接后再上电。烧完断电，拆掉这个短接，再上电。Wi-Fi 只用 2.4 GHz。

## 安全

单独操作时不接市电。ESP 由 USB-TTL 的 3.3V 供电，适配器拨在 3.3V。带电对照要有人在场，试验台使用漏保、隔离变压器和光隔离 USB-TTL。

只刷 ESP8685，不动计量 MCU。`logger` 的 `baud_rate` 保持 0，日志用 `esphome logs` 走网络。确认设备从 Wi-Fi 上线后再拔 USB-TTL。没有市电时，ESP 只靠 TTL 供电，这时不要做 OTA。
