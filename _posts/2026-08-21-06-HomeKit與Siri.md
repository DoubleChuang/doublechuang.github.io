---
title: "06 - HomeKit 與 Siri"
date: 2026-08-21
categories: [智能家居]
tags: [HomeKit, Siri]
description: "HomeKit Bridge 設定、Home Hub 限制與 Siri 捷徑方案"
series_no: "06"
---

{% include series-nav.html %}

# HomeKit 與 Siri

## HomeKit Bridge


HA 內建 HomeKit 整合,把設備曝露到 Apple 家庭 app:

1. 設定 → 裝置與服務 → + 新增整合 → **HomeKit**(不是 HomeKit Controller)
2. 勾選要曝露的設備(新風機、插座)
3. 出現 QR code + 8 位數配對碼
4. iPhone 家庭 app → + → 加入配件 → 掃碼

> 踩坑:不小心建了兩個 Bridge 時,保留已配對的(檢查 `.storage/homekit.*.state` 中 `paired_clients` > 0),刪除另一個。

## Home Hub(家庭中樞)限制

**踩坑紀錄**:人在外面時 HomeKit 無法控制——這是正常的。

原因:

1. HomeKit 是**區域網路協定**,沒有 Home Hub 時只能在同一個 WiFi 內運作
2. **VPN 也救不了**:Home app 靠 mDNS 發現配件(VPN 隧道預設丟棄多播),且 Apple 在沒有 Home Hub 時根本不會發起遠端連線
3. **Pi 也當不了 Home Hub**:Home Hub 需要 Apple 認證的專屬憑證與 iCloud 協定,只授權 HomePod / Apple TV / 常駐 iPad。任何第三方設備都無法模擬

## 解決方案比較

| 方案 | 成本 | 外面用 Siri | 說明 |
|---|---|---|---|
| HomePod mini / Apple TV | ~NT$3000 | ✅ | Apple 官方正解,全家自動 |
| Siri Shortcuts + HA app | 免費 | ✅(需 VPN 或遠端存取) | 每支手機設一次 |
| mDNS 反射器 hack | 免費 | 不保證 | 依賴 iOS 版本,社群案例時好時壞 |

## 推薦:Siri Shortcuts(免費方案)

HA 官方支援 Siri Shortcuts,把 iPhone 當語音入口:

1. 手機裝 **Home Assistant app**,登入 HA
2. HA app 設定 → 啟用 **Siri Shortcuts**(支援中文語音)
3. 在「捷徑」app 建立:「嘿 Siri,開啟新風機」→ 呼叫 HA 的新風機實體
4. 人在外面:連 VPN 或遠端存取 → 說「嘿 Siri,開啟新風機」

## 在家用 Siri

HomeKit Bridge 配對完成後,在家(同一 WiFi)就可以「嘿 Siri,打開新風機」「嘿 Siri,關電腦插座」。