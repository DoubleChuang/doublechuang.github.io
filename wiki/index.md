---
layout: page
title: "智慧家庭 Wiki"
permalink: /wiki/
---

# 智慧家庭整合實作 Wiki

把中國區米家設備(MATE A1 新風機)與台灣區米家設備(小米插座)、塗鴉插座,統一整合進 Home Assistant,並透過 HomeKit / Siri 控制,最後架設安全的遠端存取。

## 最終架構

```
Apple Home / Siri
      │  HomeKit Bridge
      ▼
Home Assistant (Raspberry Pi 3, Docker)
      ├── 中國區米家帳號 → MATE A1 新風機 (本地 miio 控制)
      ├── 台灣區米家帳號 → 小米 WiFi 插座 (雲端控制)
      └── Tuya 整合     → 塗鴉插座 x2 (雲端控制)

遠端存取: duckdns + Port Forward + Let's Encrypt HTTPS + 2FA
```

## 章節

| 章節 | 內容 |
|---|---|
| [01 - 硬體環境](/posts/01-硬體環境) | Pi 3、老毛子路由器、SD 卡規格與最佳化 |
| [02 - Home Assistant 安裝](/posts/02-HA安裝) | Docker 安裝、armv7 停更對策、記憶體調整 |
| [03 - 米家整合](/posts/03-米家整合) | 中國區 + 台灣區雙帳號、本地/雲端控制模式 |
| [04 - 疑難排解](/posts/04-疑難排解) | 70016 風控、DNS 故障、插座離線、OAuth 失敗 |
| [05 - 塗鴉插座整合](/posts/05-塗鴉插座) | Tuya 整合流程與開發者帳號申請 |
| [06 - HomeKit 與 Siri](/posts/06-HomeKit與Siri) | HomeKit Bridge、Home Hub 限制、Siri 捷徑方案 |
| [07 - 遠端存取](/posts/07-遠端存取) | duckdns + Port Forward + Let's Encrypt + 2FA |
| [08 - 日常維護](/posts/08-日常維護) | 自動清理、資料庫保留、SD 卡升級 |

## 背景

- **問題**:MATE A1 新風機只能接入中國大陸區域的米家 app,而原本的小米插座在台灣區域,兩者無法同時使用
- **目標**:統一控制所有設備、Siri 語音控制優先、保留米家台灣區 app 使用
- **關鍵結論**:Smart Life app 無法整合米家設備(Tuya 生態),路由器抓封包逆向非正解(MIoT 協議公開),Home Assistant 是正解

## 環境規格速查

| 項目 | 規格 |
|---|---|
| Pi | Raspberry Pi 3 Model B (1GB RAM, 16GB SD) |
| 路由器 | 小米 Mini (RT-AC54U) 刷老毛子 Padavan |
| HA 版本 | 2025.11.3 (armv7 最終版, Docker) |
| 整合 | hass-xiaomi-miot (v1.1.4) + Tuya 官方整合 |
| 新風機 | MATE Air Fresh A1 (mate.airfresh.a1), <新風機-IP> |
| 插座 | 小米 WiFi 插座 (qmi.plug.tw02), <插座-IP> |
| 網域 | <你的網域>.duckdns.org (duckdns 免費) |