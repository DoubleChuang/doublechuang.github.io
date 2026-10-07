---
title: "【opencode 自架】用 Caddy + Docker 把 AI 編碼助手安全開上外網"
date: 2026-10-08
categories: [開發筆記]
tags: [opencode, Caddy, Docker, Padavan, 遠端存取]
description: "systemd 常駐 + Caddy DNS-01 憑證 + Padavan Port Forward,讓 opencode web 用 HTTPS 安全對外服務"
---

> 想在外面用手機或筆電連回家裡的 opencode,又不想把一個「可以執行 shell 的 API」裸奔在公網上。這篇記錄怎麼用 systemd + Docker Compose + Caddy,把它包成 HTTPS 對外服務。

## 目標與架構

```
瀏覽器(外網)
   │  https://<DDNS_DOMAIN>:<WAN_PORT>
   ▼
路由器(Padavan)  Port Forward:WAN <WAN_PORT> → <HOST_IP>:<TLS_PORT>
   ▼
Caddy(Docker,host network,TLS 終結)
   │  reverse_proxy 127.0.0.1:<OPENCODE_PORT>
   ▼
opencode web(systemd,0.0.0.0:<OPENCODE_PORT>,Basic Auth 密碼保護)
```

- 憑證用 Let's Encrypt **DNS-01** 驗證,**完全不需要對外開 80/443**
- 路由器只需轉發一個埠;Caddy 負責 TLS 與反向代理
- opencode 本身仍保留密碼,雙層防護

## 先釐清:`opencode serve` 與 `opencode web`

官方文件常讓人誤會這是兩個要一起跑的東西,其實:

- `opencode serve`:只有 HTTP API 的 headless server,**沒有網頁介面**
- `opencode web`:**同一個 server** 加上內建網頁 UI,由同一個 process、同一個 port 提供

也就是說 **只需要挑一個跑**,`opencode web` 是超集,API(`/global/health`、`/doc`...)與 UI 都在同一埠。不需要「serve 開一個、web 再開一個」。

## 1. 本機服務:systemd 常駐 opencode web

密碼檔(權限 600,不要進 git):

```bash
# 產生一組強密碼
openssl rand -hex 16

# ~/.config/opencode-web.env
OPENCODE_SERVER_PASSWORD=<你產生的強密碼>
```

```bash
chmod 600 ~/.config/opencode-web.env
```

systemd unit(`/etc/systemd/system/opencode-web.service`):

```ini
[Unit]
Description=opencode web server (headless, password protected)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu
Environment=HOME=/home/ubuntu
Environment=BROWSER=/bin/true
EnvironmentFile=/home/ubuntu/.config/opencode-web.env
ExecStart=/home/ubuntu/.opencode/bin/opencode web --hostname 0.0.0.0 --port <OPENCODE_PORT> --print-logs --cors https://<DDNS_DOMAIN>:<WAN_PORT>
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

幾個細節:

- `BROWSER=/bin/true`:headless 環境下避免 opencode 嘗試開啟瀏覽器
- `--print-logs`:log 進 journald,方便 `journalctl -u opencode-web`
- `OPENCODE_SERVER_PASSWORD`:opencode 會啟用 HTTP Basic Auth,使用者名稱預設 `opencode`
- `--cors`:內建 web UI 同源其實不需要 CORS;這裡只是預留給未來自訂前端

啟用與驗證:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opencode-web

# 未帶密碼應得到 401
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<OPENCODE_PORT>/global/health

# 帶密碼應得到 {"healthy":true,...}
curl -s -u "opencode:<你的密碼>" http://127.0.0.1:<OPENCODE_PORT>/global/health
```

防火牆(ufw)只需放行 web 與 TLS 兩個埠:

```bash
sudo ufw allow <OPENCODE_PORT>/tcp
sudo ufw allow <TLS_PORT>/tcp
```

## 2. HTTPS:Caddy + Let's Encrypt(DNS-01)

### 為什麼用 Docker Compose 跑 Caddy?

- Caddy 內建自動申請/續期 Let's Encrypt 憑證,比手動 acme.sh 少一堆 cron
- 用 **DNS-01** 驗證,不需要開放 80/443 給 ACME challenge
- DuckDNS 的 DNS provider 模組不在官方 image 裡,用 `xcaddy` 自己編一個

`~/docker/caddy-opencode/Dockerfile`:

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/duckdns

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

`compose.yaml`:

```yaml
services:
  caddy:
    build: .
    container_name: caddy-opencode
    restart: unless-stopped
    network_mode: host
    environment:
      DUCKDNS_TOKEN: ${DUCKDNS_TOKEN}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data:/data
      - ./config:/config
```

> `network_mode: host` 讓 Caddy 直接監聽主機埠,ufw 也才管得到;bridge 模式的 port mapping 會繞過 ufw。

`.env`(權限 600,DuckDNS token 從 duckdns.org 帳號取得):

```bash
DUCKDNS_TOKEN=<你的 DuckDNS token>
```

`Caddyfile`:

```text
{
	admin off
	auto_https disable_redirects
}

https://<DDNS_DOMAIN>:<TLS_PORT> {
	tls {
		dns duckdns {env.DUCKDNS_TOKEN}
	}
	reverse_proxy 127.0.0.1:<OPENCODE_PORT>
}
```

`admin off` 關掉本機管理 API;`auto_https disable_redirects` 避免 Caddy 額外佔用 80 埠做 HTTP 重導。啟動:

```bash
docker compose up -d --build
docker logs caddy-opencode
```

log 出現 `certificate obtained successfully` 就代表 DNS-01 驗證過關、憑證簽發成功,之後 Caddy 會自動續期。

## 3. 路由器:Padavan 加 Port Forward

Padavan(老毛子)的埠轉發不是存在單一 `vts_rulelist`,而是**索引式 nvram 變數**:`vts_num_x` 是規則數,第 N 條規則拆成 `vts_port_xN`、`vts_ipaddr_xN`、`vts_lport_xN`、`vts_proto_xN` 等。索引從 0 開始。

**動手前先備份**,並確認既有規則的索引到哪:

```bash
nvram show | grep -E '^vts_' > /tmp/vts-backup.txt
```

假設原本有 2 條規則(x0、x1),新規則接在 x2:

```bash
nvram set vts_num_x=3
nvram set vts_port_x2=<WAN_PORT>
nvram set vts_ipaddr_x2=<HOST_IP>
nvram set vts_lport_x2=<TLS_PORT>
nvram set vts_proto_x2=TCP
nvram set vts_srcip_x2=
nvram set vts_protono_x2=
nvram set vts_desc_x2=opencode
nvram commit
/sbin/restart_firewall
```

驗證可以看路由器 Web UI 的埠轉發頁面,規則會多出一條,且**既有規則原封不動**。

> 對外埠想降低被掃描的機率,就挑個非標準埠;這篇用 `<WAN_PORT>` 純粹是好記。

## 踩坑一:非互動 SSH 找不到 nvram

用 SSH 一行指令操作路由器時:

```text
$ ssh admin@<ROUTER_IP> "nvram get vts_rulelist"
sh: line 0: nvram: not found
```

原因是 dropbear 給非互動 shell 的 `PATH` 只有 `/usr/bin:/bin`,而 `nvram` 在 `/usr/sbin`。兩個解法:

```bash
# 方法一:用完整路徑
ssh admin@<ROUTER_IP> "/usr/sbin/nvram get vts_rulelist"

# 方法二:自己補 PATH(rc 腳本內部也會呼叫 nvram,建議用這個)
ssh admin@<ROUTER_IP> "export PATH=/usr/sbin:/usr/bin:/sbin:/bin; /sbin/restart_firewall"
```

尤其 `restart_firewall`、`restart_dhcpd` 這類 rc 動作會間接呼叫 `nvram`,PATH 沒補會出現各種莫名其妙的 `not found`,規則也不會真的生效。

## 踩坑二:靜態 DHCP 用 nvram 設定無效

為了讓主機 IP 固定(避免租約到期後 Port Forward 指向空號),我照 Web UI 的欄位名稱設了:

```bash
nvram set dhcp_staticnum_x=1
nvram set dhcp_staticmac_x0=<MAC>
nvram set dhcp_staticip_x0=<HOST_IP>
nvram set dhcp_staticname_x0=<HOSTNAME>
nvram commit
/sbin/restart_dhcpd
```

Web UI 的 DHCP 頁面**看得到**這筆靜態租約,但 dnsmasq 實際讀的 `/etc/dnsmasq/dhcp/dhcp-hosts.rc` 重啟後仍是 **0 bytes**,`nslookup <HOSTNAME>` 也查不到——代表租約根本沒生效。

實測此版 Padavan 的 `restart_dhcpd` 不會從 nvram 重新生成 hosts 檔(推測是開機路徑才有,或此版行為如此)。繞過方法是直接寫進 dnsmasq 的**持久化**設定檔:

```bash
cat >> /etc/storage/dnsmasq/dnsmasq.conf <<'EOF'

### Static lease: <HOSTNAME>
dhcp-host=<MAC>,<HOST_IP>,<HOSTNAME>
EOF

# 存進 MTD,重開機也不會掉
/sbin/mtd_storage.sh save
/sbin/restart_dhcpd
```

驗證:

```bash
nslookup <HOSTNAME> <ROUTER_IP>
# Name: <HOSTNAME>.lan
# Address: <HOST_IP>
```

> 注意:若同時保留 nvram 裡的靜態租約,未來某次重開機生成 hosts 檔時可能出現重複的 `dhcp-host`。二選一,別兩邊都留。

## 驗證

本機與 TLS:

```bash
# 1. 本機直連(區網用 http://<HOST_IP>:<OPENCODE_PORT>)
curl -s -u "opencode:<你的密碼>" http://127.0.0.1:<OPENCODE_PORT>/global/health

# 2. 走 Caddy 的 TLS(憑證必須有效,不加 -k)
curl -s --resolve <DDNS_DOMAIN>:<TLS_PORT>:127.0.0.1 \
  -u "opencode:<你的密碼>" https://<DDNS_DOMAIN>:<TLS_PORT>/global/health
```

從**真正的外網**驗證(借 check-host.net 的節點):

```bash
curl -s -H 'Accept: application/json' \
  "https://check-host.net/check-tcp?host=<PUBLIC_IP>:<WAN_PORT>&max_nodes=3"
# 回傳 request_id 後再查結果
curl -s -H 'Accept: application/json' \
  "https://check-host.net/check-result/<request_id>"
```

各節點都回報 `address` + `time` 就是 TCP 連得上;若回錯誤字串就是被擋或沒開。

意外收穫:Padavan 有 **NAT loopback**,區網內也能直接連 `https://<DDNS_DOMAIN>:<WAN_PORT>`,不用分兩套網址。

## 安全性檢查清單

- opencode 的 API 可以讀寫檔案、執行 shell(`/session/:id/shell`),**能連上它等於能控制整台機器**
- `OPENCODE_SERVER_PASSWORD` 必設;對外務必走 HTTPS,否則 Basic Auth 只是 base64,等於明文
- 密碼、DuckDNS token 放 600 權限的 env 檔,永遠不進 git
- 路由器轉發只開必要埠;Caddy 不管的埠一個都不要開
- 定期 `journalctl -u opencode-web` 看一下有沒有異常連線嘗試

## 結語

整體架構其實只有三段:systemd 顧好本機服務、Caddy 顧好 TLS、路由器顧好一條 Port Forward。真正花時間的不是設定本身,而是 Padavan 那兩個坑(SSH PATH 與靜態 DHCP)。避開之後,從外網打開 `https://<DDNS_DOMAIN>:<WAN_PORT>`,就像在家裡用一樣順。
