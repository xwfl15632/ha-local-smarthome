# 脱敏清单（DESSENSITIZATION LOG）

本仓库是生产 HA 配置的**脱敏开源版**。原始 config 目录含真实凭据，**绝不**入库。
以下逐条记录发布前移除/替换了什么、为什么。

## 完全不入库（含真实凭据，仅生产机器上保留）
- `.storage/*`（含 config_entries.json：Tuya access/refresh token、MQTT 密码、
  Reolink admin 密码、ESPHome noise_psk、mobile_app push token/webhook、
  全部设备 MAC/私网 IP）
- `v2.db`、`home-assistant.log`、`backups/`、`media/`、`tmp/`
- 各集成 YAML 里用 `!secret` 引用的密钥值（secrets.yaml 本体也不入库，
  仅保留结构示意）

## 逐文件脱敏（入库的 YAML 里做的替换）
| 文件 | 原内容 | 替换为 | 原因 |
|---|---|---|---|
| scripts.yaml | `<内网IP>:50100` | `ANNOUNCE_HOST:50100` | 内网 IP |
| scripts.yaml | `key=30bea4cf...` | `key=CHANGE_ME_ANNOUNCE_API_KEY` | announce-proxy API key |
| automations.yaml | `mobile_app_sm_s9480` | `mobile_app_phone` | 手机型号（可定位人）|
| scripts.yaml | `alexa_media_xu_s_echo_spot` | `alexa_media_echo_spot` | 姓氏 |
| lovelace_dashboards.json | 2 处私网 IP | `10.0.0.x` 占位 | 内网拓扑 |

> 注：本日志本身也做了脱敏——真实 IP/密钥值一律不出现，只记录"替换了什么类型"。

## 保留（非敏感、有教学价值）
- 实体命名采用拼音（chuang_lian_deng_dai=窗帘灯带等），本身不含个人信息
- Tuya 设备 `0x...` 指纹 ID（非密钥，仅设备标识，开源无害）
- 设备品牌型号（Denon/Reolink/TCL/当贝/esp32 等）——开源的核心就是"这些真实设备怎么接"
