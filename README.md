# 家庭 Home Assistant 智能家居

一套以本地为主的 Home Assistant 部署：语音助手（唤醒→STT→LLM→TTS）、
全屋灯光/窗帘、观影模式、Reolink 摄像头、多音区音箱、视频智能化。

HA 以 **Docker 方式运行在家庭 NAS 的 fnOS** 上；本地影音系统采用 **Jellyfin**。

---

## 整体架构

```
                                  +----------------------------------+
                                  | Home Assistant (Docker on fnOS)  |
                                  | Assist / automations / scripts   |
                                  | adaptive_lighting / intents      |
                                  +----------------------------------+
                                                    |
             +-------------------------------------------------------------------------+
             v                                  v                                     v
+------------------------+         +------------------------+            +-------------------------+
| ESPHome (ESP32-S3 x N) |         | MQTT broker            |            | Voice backends (GPU)    |
| satellite / light /    |         | Zigbee2MQTT            |            | STT  whisper large-v3   |
| switch                 |         | M200 Zigbee -> Matter  |            | TTS  Breeze-TTS 2.0     |
+------------------------+         +------------------------+            | LLM  cloud Qwen (Flash) |
                                                                         +-------------------------+
                                                    |
     +--------------------------------------------------------------------------------------------+
     | Devices: Tuya switches/lights/covers (official Tuya integration, cloud)                    |
     | M200 Zigbee sensors | Denon AVR-X4700H amp (living) | HEOS (study)                         |
     | Reolink camera (RTSP) | TV (AirPlay) | Dangbei projector (IR) | ESP32 zones                |
     +--------------------------------------------------------------------------------------------+
                                                    |
                           +------------------------------------------------+
                           | VPS (frps): remote access to HA + device SSH   |
                           +------------------------------------------------+
```

图例：语音后端 = STT（whisper large-v3）+ TTS（Breeze-TTS 2.0）均本地 GPU、wyoming 协议；
LLM 当前为云端 Qwen Flash（本地 LLM 暂未启用，见第 1 节）。设备层经官方 Tuya 集成云端接入；
当贝是**投影**（IR 遥控）；功放两台：Denon 4700H（客厅）+ HEOS（书房）。

核心原则：**能本地就本地**。STT/TTS 与视频智能化跑在本机 GPU 上，LLM 暂用云端（见第 1 节）；HA 是编排层。

---

## 目录说明

| 文件 | 内容 |
|---|---|
| [`config/configuration.yaml`](config/configuration.yaml) | 主配置：集成、语音（assist/STT/LLM/TTS）、adaptive_lighting、本地 intent、**窗帘反相模板** |
| [`config/automations.yaml`](config/automations.yaml) | 自动化：感应开/关灯（楼梯/书房/阳台/厨房）、**语音卫星卡死看门狗**、播报切声场 |
| [`config/scripts.yaml`](config/scripts.yaml) | 脚本：**echo_say 流式 TTS**（快路径+兜底）、**观影模式**、全屋/分房关灯、投影 IR 遥控 |
| [`config/lovelace_dashboards.json`](config/lovelace_dashboards.json) | 仪表盘（Lovelace UI） |

---

## 关键设计与踩坑

### 1. 语音链路
唤醒词在 ESP32 上跑，链路：
`STT(whisper large-v3, wyoming) → LLM(assist) → TTS(Breeze-TTS 2.0, wyoming) → 多音区播放`。

- **STT / TTS 全本地**（本地 GPU 上的 whisper large-v3 + Breeze-TTS 2.0，走 wyoming 协议），
  隐私 + 延迟可控 + 断网可用。
- **LLM 当前用云端 Qwen Flash**（考虑延迟，本地 LLM 暂未启用）。
  有条件建议启用本地 LLM 以确保零云——本地 LLM 的算力/延迟调优达标后再切，
  切换只改 assist 的 language model，链路其余部分不动。

### 2. echo_say 流式 TTS（快路径 + 兜底）
`scripts.yaml` 的 `echo_say_fast` 是这套系统里最复杂的脚本，设计目标：**一次合成、渐进流式推给全屋多个音箱**，避免逐台合成带来的延迟叠加。

- **快路径**：调本地 TTS 服务的 HTTP 流式接口（`ANNOUNCE_PROXY_HOST:50100/play`），
  把同一份合成音频同时推给 Denon / Homatics / 各 ESP32 音区，并用 `sensor.announce_fast`
  轮询任务状态（`job` + `playing`）。
- **兜底**：若 6 秒内快路径没到 `playing`，回落到 HA 内置
  `media-source://tts/tts.f5tts`（wyoming TTS）逐台播。
- **闭麦/开麦**：播报期间把卫星的 mute 开关打开（防自唤醒/回声），结束再打开。
- **超时随文本长度缩放**：`max(25, len*0.25+15)` 秒，长文本给足播放时间。

> 踩坑：流式路径和兜底路径的**实体列表必须保持同步**，否则会出现"某音区只快路径有、
> 兜底没有"的不对称——改播放器时两处都要动。

### 3. 窗帘反相模板
Tuya 设备（灯/开关/窗帘）全部经 **HA 官方 Tuya 集成（Tuya 网关，即智能生活 App
账号 / 涂鸦网关）云端接入**——不走 LocalTuya：家里大量 Tuya 设备是子设备
（无独立 LAN IP），本地化接不进来。

Tuya 窗帘的 position 语义与 HA 标准相反（0=全开 / 100=全关，或方向相反）。
`configuration.yaml` 里用 `template` cover 包一层，把 position 取反后再暴露给上层，
这样自动化/脚本里就不用到处写 `100 - position`。

> 踩坑：不反相时，"开帘"命令会让帘子往反方向走，且 `is_opening` 等属性全错。
> 模板层统一处理，上层无感。

### 4. 本地 LLM intent 路由（精确匹配绕 LLM）
对高频、语义确定的指令（如"关灯""打开观影模式"），用 `conversation` intent 的
**精确/正则匹配**直接命中，不经过 LLM 推理。好处是**确定、零延迟、不烧算力**；
只有模糊指令才 fallback 到 LLM（当前为云端 Qwen，见第 1 节）。

### 5. 语音卫星卡死看门狗
`automations.yaml` 的 `voice_satellite_stuck_watchdog`：assist_satellite 卡在
`responding`/`thinking` 超 3 分钟（常见于 TTS 出音频失败导致会话无法收尾），
自动 `announce` 复位并推送结果。
> 踩坑来源：某次 TTS 故障导致卫星卡死 8.5 小时、"Okay Nabu" 全无反应，才加的这道保险。

### 6. 观影模式脚本
`scripts.yaml` 的 `guan_ying_mo_shi`：当贝投影上电（唯一关机路径是 IR）→ IR 遥控进主页 →
电视开 → 客厅功放(Denon AVR-X4700H)开并切输入源 → 关主灯、留氛围灯(15%) →
开 adaptive_lighting manual_control → 关阳台/过道 → 关帘。
（功放共两台：Denon 4700H 在客厅、HEOS 在书房；观影只动客厅这台。）
结束脚本做逆操作，并按日落时间决定是否开帘。
> 踩坑：灯 `unavailable` 等待要用 `wait_template` 给足 timeout 并 `continue_on_timeout`，
> 否则个别灯慢上线会让整个脚本卡住。

### 7. 感应灯（楼梯/书房/阳台/厨房）
统一套路：`occupancy` 变 `on` 且**照度低于阈值**才开灯；变 `off` 延时 2 秒关灯。
照度条件避免白天/已经够亮时空开灯。

### 8. Reolink 摄像头（RTSP）
HA 走 RTSP 接入（`camera.*`），隐私模式 = 开遮蔽 + 停录像。人数统计是商用 NVR 专属，
家用机 AI 只有人/兽/婴儿二值——**多人数/人脸识别走本地 GPU 另起服务**，不在 HA 里做。

### 9. M200 Zigbee → Matter
三台 FP310 走 M200 的 Zigbee 侧 → Matter 桥进 HA（无 Thread）。
> 踩坑：二次 commission 后**设备名会重置为默认**，自动化里引用的实体要重新指回；
> 删设备的正解是走 MQTT entry（z2m 独立容器做纯网关 + MQTT 自动发现）。

### 10. frp 远程访问
HA 与设备 SSH 都经 VPS 的 frp 反向隧道暴露到外网，家里是 NAT 后无法直连。
隧道状态 ≠ 后端存活（端口 OPEN 不代表服务活），排障要逐层验证。

### 11. 视频采集与智能化（Reolink + 本地 GPU）
HA 的 Reolink 集成只出 RTSP 流和"有人"二值信号，人数/认脸在 HA 之外另起本地服务：

- **人数检测**：盯 HA 的 `binary_sensor.living_room_person`，有人时抓帧给本地
  视觉语言模型数人头（秒级），人在场期间每 30 秒复检（覆盖"第 3 人悄悄进来"）。
- **人脸识别**：InsightFace（CPU 跑，不占 GPU），家庭成员人脸库入库 +
  留一验证定阈值，识别出的名字直接拼进通知。
- **多人录像联动**：检测到 ≥3 人 → 推送通知 + 远端拉起 NAS 上的 ffmpeg 录像
  （RTSP 直拉、`-c copy` 不转码）；人走或录满 10 分钟自动停。
- 隐私边界：摄像头在 HA 里可开遮蔽模式；录像落 NAS 外置盘，NAS cron 定期清理。

> 设计原则：HA 只做"触发源"（presence 信号），重活（数人头/认脸/录像）全在
> 本地 GPU 和 NAS 上，家用 NVR 没有的 AI 能力自己补。

---

## 如何复用
1. 把 `config/` 下 YAML 拷进你的 HA `config/` 目录（先备份原配置）。
2. 实体 ID 是按你家的房间/设备拼音命名的——**需要改成你自己的实体**。
3. `scripts.yaml` 里的占位符（`ANNOUNCE_PROXY_HOST`、`CHANGE_ME_...`）填回你自己的值。
4. 语音链路依赖的本地 GPU 服务（STT/TTS）需自行部署，LLM（本地或云端）自行配置，本仓库不含服务源码。
