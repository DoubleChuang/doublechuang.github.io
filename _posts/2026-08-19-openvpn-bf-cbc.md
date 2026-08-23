---
title: "【OpenVPN 排錯實錄】Cipher BF-CBC not supported:路由器伺服器為何一直失敗?"
date: 2026-08-19
categories: [智能家居]
tags: [OpenVPN, Padavan, 疑難排解]
description: "Padavan 路由器 OpenVPN 伺服器 BF-CBC cipher 排錯實錄"
---

> Debugging OpenVPN on a Router: Why "Cipher BF-CBC not supported" Kept Killing My Server
>
> A real-world troubleshooting story — when a pile of WARNINGs hides a fatal error, your OpenVPN server won't even start. Here's how I traced it down on a Padavan router, from nc port checks to reading the router's own web-page source to find the cipher mapping.

## VPN 怎麼又連不上了?

「VPN 連不上了。」——這大概是每個架過 VPN 的人都會遇到的日常。

這次的問題出在 OpenVPN server 上。對外測試 118.165.26.160:1194(TCP)和區網內 192.168.1.1:1194,結果都是:

```text
nc: connectx to 118.165.26.160 port 1194 (tcp) failed: Connection refused
nc: connectx to 192.168.1.1 port 1194 (tcp) failed: Connection refused
```

**Connection refused** 不是 timeout,也不是被防火牆擋住(那通常是 timeout / filtered)。refused 代表對端有回應、但**根本沒有服務在監聽這個 port**。也就是說,OpenVPN server 很可能壓根沒起來。

## 背景:環境

- 路由器:Xiaomi Mini(RT-AC54U-GPIO-30-xiaomimini-128M)
- 韌體:Padavan 3.4.3.9-099_26-03-1(BusyBox shell)
- OpenVPN server 版本:OpenVPN 2.5.11 mipsel-unknown-linux-gnu(OpenSSL 3.0.20-dev、LZO、LZ4)
- 設定方式:透過路由器 Web UI 的 VPN Server 頁面啟用,進階自訂設定放在 /etc/storage/openvpn/server/server.conf

## 症狀:一堆 WARNING,還有幾行致命的

登入路由器看 log(/tmp/syslog.log),看到一堆「警告」:

```text
WARNING: Compression for receiving enabled. ...
DEPRECATED OPTION: ncp-disable. ...
DEPRECATED OPTION: --cipher set to 'BF-CBC' but missing in --data-ciphers ...
NOTE: the current --script-security setting may allow this configuration ...
```

一般人看到 **WARNING / DEPRECATED** 往往鬆一口氣:「啊,只是警告而已。」但真正致命的那行混在後面:

```text
openvpn-srv[11920]: Cipher BF-CBC not supported
```

然後 process 就死了,沒有下文。**這才是殺手。**

> 一個關鍵的心法:**WARNING 可以不理,但出現「failed / not supported / error」就一定要停下來看。** 這些 WARNING 只是干擾項。

## Debug 過程

### Step 1:確認 server 根本沒在跑

```bash
$ ps w | grep openvpn | grep -v grep
#（沒有輸出 → 沒有 process）

$ netstat -tln | grep 1194
#（沒有輸出 → 沒有監聽）
```

沒有 process、沒有監聽,跟 port refused 完全吻合。

### Step 2:在 log 裡挖出致命錯誤

把 log 用 grep openvpn 撈出來,才發現每次啟動的結尾都是同一句:

```text
RT-AC54U: starting OpenVPN server...
openvpn-srv[11919]: WARNING: Compression for receiving enabled. ...
openvpn-srv[11919]: DEPRECATED OPTION: ncp-disable. ...
openvpn-srv[11919]: DEPRECATED OPTION: --cipher set to 'BF-CBC' ...
openvpn-srv[11920]: NOTE: the current --script-security setting ...
openvpn-srv[11920]: Cipher BF-CBC not supported
```

**Cipher BF-CBC not supported** — 這個 OpenVPN 版本根本不認識 BF-CBC。

### Step 3:驗證編譯時支援哪些 cipher

用 `openvpn --show-ciphers` 列出實際支援的加密:

```text
OpenVPN 2.5.11 mipsel-unknown-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] ...

AES-128-CBC   (128 bit key, 128 bit block)
AES-256-CBC   (256 bit key, 128 bit block)
AES-128-GCM   (128 bit key, ...)
AES-256-GCM   (256 bit key, ...)
CHACHA20-POLY1305 (256 bit key, ...)
```

有 AES、有 GCM、有 CHACHA20,**就是沒有 BF-CBC(Blowfish)**。這代表這顆 OpenVPN 在編譯時被拿掉了 Blowfish 支援(新版 OpenSSL / 安全考量常會 no-blowfish)。你可以在設定裡亂寫沒被編譯進去的 cipher,啟動就直接失敗。

### Step 4:對照 client config 與 server config

Client 端(.ovpn)原本是:

```text
cipher BF-CBC
data-ciphers CHACHA20-POLY1305:AES-256-GCM:AES-128-GCM
compress lzo
```

而 router 產生的 /etc/openvpn/server/server.conf 也有一模一樣的 BF-CBC:

```text
proto tcp4-server
port 1194
cipher BF-CBC
data-ciphers CHACHA20-POLY1305:AES-256-GCM:AES-128-GCM
compress lzo
```

問題很清楚:**兩邊都用了路由器不支援的 BF-CBC。**

### Step 5:反查 Web UI 的 cipher 對應表

那 BF-CBC 是哪來的?是 Web UI 的「加密方式」下拉選單存的數值。Padavan 的 vpns_ov_ciph 數值對應是寫在 Web 頁面原始碼裡的,直接在路由器上挖:

```bash
$ grep -n "vpns_ov_ciph" /www/vpnsrv.asp
```

對應表長這樣(節錄):

| 數值 | Cipher |
|---|---|
| 0 | [none] |
| 3 | **BF-CBC**(Blowfish) ← 原本選的 |
| 4 | AES-128-CBC |
| 5 | AES-192-CBC |
| 8 | **AES-256-CBC** ← 我改成的 |

現在 nvram 裡存的是 vpns_ov_ciph=3,就是 BF-CBC。

### Step 6:修復

改成路由器支援的 AES-256-CBC,並把過時的 ncp-disable 拿掉(改用現代的 cipher 協商):

```bash
$ nvram set vpns_ov_ciph=8
$ nvram commit
```

```bash
# 編輯 /etc/storage/openvpn/server/server.conf,移除 ncp-disable
# 重新啟動(注意:要用 symlink 名稱,不是 /sbin/rc)
$ /sbin/restart_vpn_server
```

> 小知識:Padavan 的 /sbin/rc 是靠 **argv[0]**(symlink 名稱)分派指令的。直接敲 `/sbin/rc restart_vpn_server` 會得到 exit code 22(EINVAL),什麼事都沒發生。要用 `/sbin/restart_vpn_server` 這個 symlink 才會真的重啟。

### Step 7:驗證

```bash
$ ps w | grep openvpn
30895 nobody  ... /usr/sbin/openvpn --daemon openvpn-srv --config server.conf

$ netstat -tln | grep 1194
tcp  0  0 0.0.0.0:1194  0.0.0.0:*  LISTEN
```

再從外部測一次,全部通了:

```text
Connection to 192.168.1.1 port 1194 [tcp/openvpn] succeeded!
Connection to 118.165.26.160 port 1194 [tcp/openvpn] succeeded!
```

Client 端也同步把 `cipher BF-CBC` 改成 `cipher AES-256-CBC`,兩邊一致。收工。

## 技術重點整理

### 1. WARNING ≠ 不是問題

這次最危險的就是「一堆 WARNING 讓人誤以為沒問題」。關鍵要看有沒有 **error / failed / not supported** 這類字眼:

- `WARNING: Compression ...` → 只是安全提示,可以忽略
- `DEPRECATED OPTION: ...` → 以後的版本才會移除,暫時沒影響
- **`Cipher BF-CBC not supported`** → **致命錯誤,server 直接不啟動**

### 2. 為什麼這台路由器沒有 BF-CBC?

OpenVPN 的 cipher 支援取決於編譯時的 OpenSSL。新版韌體為了安全常把老舊的 Blowfish 拿掉。你的設定再怎麼寫,只要不是編譯進去的 cipher,啟動就爆。

**檢查方法:**

```bash
openvpn --show-ciphers
```

### 3. cipher、data-ciphers、ncp-disable 的關係

- data-ciphers:開啟 NCP(cipher negotiation)後,兩邊協商時可用的清單。
- cipher:舊版固定加密;在 2.4+ 有 NCP 時變成「fallback」。
- ncp-disable:關掉協商,強制用 cipher —— 它是過時的除錯功能,OpenVPN 2.6 會移除。

正確做法:data-ciphers 列現代 AEAD(CHACHA20-POLY1305 / AES-GCM),別用 ncp-disable,讓兩邊自動挑一個都支援的。

### 4. 路由器上的選項是「數字」,不是字串

Padavan 這類韌體把 Web UI 的選項存成 nvram 數字(vpns_ov_ciph=3),Web 頁面再翻譯成字串寫進設定檔。**Web 頁面原始碼(/www/*.asp)就是現成的對照表**,grep 一下就能找到。

## 結語

這個 bug 的教訓其實很簡單:

1. **先確認服務真的在跑**(ps / netstat),再談設定。
2. **不要被 WARNING 騙到**——找 error 等級的訊息。
3. **cipher 一定要確認是該版本支援的**,`openvpn --show-ciphers` 是你的好朋友。
4. **韌體 Web UI 的數字選項,去 /www/*.asp 翻對照表最準。**

如果你也用 Padavan / 小米路由器架 OpenVPN,記得檢查你的「加密方式」是不是也默默選了 BF-CBC。換成 AES-256-CBC 或直接用 GCM,啟動就會順利很多。

寫給自己也寫給需要的人。如果你也遇過類似的怪 bug,歡迎留言分享。

---

原文發布於 [Medium](https://medium.com/@ethan9141/openvpn-%E6%8E%92%E9%8C%AF%E5%AF%A6%E9%8C%84-cipher-bf-cbc-not-supported-%E8%B7%AF%E7%94%B1%E5%99%A8%E4%BC%BA%E6%9C%8D%E5%99%A8%E7%82%BA%E4%BD%95%E4%B8%80%E7%9B%B4%E5%A4%B1%E6%95%97-2a5a651932f6)