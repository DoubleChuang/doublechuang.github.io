---
title: "【收操 · 緩】用 Cloudflare Turnstile 守住 AI 模型額度"
date: 2026-11-15
categories: [開發筆記]
tags: [Cloudflare, Turnstile, Workers AI, Vue, 成本防護]
description: "靜態網站 + Workers AI 的 AI 功能，用 Turnstile 擋掉偽造 Origin 的機器人，保住每日免費模型額度"
---

> 「幫我配」是收操編輯器裡的一個 AI 入口：打一句「昨天跑 10K，右邊屁股很緊，只有十分鐘」，Worker 會把這句話轉成五個選項。功能很小，但每次呼叫都會消耗 Workers AI 的免費額度；端點放在公網上，就得面對有人拿 curl 把它當免費 API 打的問題。這篇記錄最後怎麼用 Cloudflare Turnstile 把「真人」和「機器人」分開。

## 架構：只有一個地方會花錢

整個網站是 GitHub Pages 上的靜態 Vue app，排課邏輯全部在前端；唯一會呼叫模型的是 `/parse`：

```
瀏覽器
  │  POST /parse { text }
  ▼
Cloudflare Worker（Origin/Referer、限流、每日上限）
  │
  ▼
Workers AI（免費 10,000 neurons/day）
```

Worker 只做欄位抽取，不做排課。這個設計讓「AI 掛掉」不影響主要功能，但也代表 `/parse` 是一個獨立的公開端點。

## 第一道門擋不住 curl

原本的防護有三層：

- Origin / Referer 白名單
- 每分鐘每 IP 限流
- 每日總量上限（KV）

但 Origin 檢查只對瀏覽器有效，對 curl 無效：

```bash
curl -X POST https://cooldown-parse.<你的帳號>.workers.dev/parse \
  -H "Origin: https://doublechuang.github.io" \
  -H "content-type: application/json" \
  -d '{"text":"昨天跑十公里，右邊屁股很緊"}'
```

照樣可以打。限流是「打得快就擋」，每日上限是「打到 300 次就停」，都不是「只給真人用」。免費額度用完雖然不會產生帳單，但當天就沒了。

## 為什麼選 Turnstile

- 免費方案就是無限次挑戰，沒有按次計費
- 不用把 GitHub Pages 搬進 Cloudflare，也不用改 DNS
- `interaction-only` 模式下，多數訪客根本看不到 widget，可疑流量才會被要求互動
- 有 siteverify API，server 端可以真的驗證 token，不是只看欄位有沒有值

這個組合不能直接用 Cloudflare WAF：GitHub Pages 的網域不是自己的 Cloudflare zone，Worker 在 `workers.dev` 也沒有 WAF 規則可設。Turnstile 是這個架構裡唯一能分辨「真人瀏覽器」的一層。

## 整體流程

```
使用者按下「幫我填」
  → 前端執行 Turnstile，拿到 token（一次性、5 分鐘）
  → POST /parse { text, turnstileToken }
  → Worker 用 secret 呼叫 siteverify
  → 成功才呼叫 Workers AI
  → 失敗回 403，前端退回手動五題
```

## 前端：正常訪客看不到 widget

Site key 放在靜態設定裡：

```json
{
  "parseEndpoint": "https://cooldown-parse.<你的帳號>.workers.dev/parse",
  "turnstileSiteKey": "0x4AAAA..."
}
```

Site key 是公開值，放在 `public/config.json` 沒問題；Secret Key 才是要保密的。

載入時用 explicit 模式，按下按鈕才執行：

```ts
const widgetId = turnstile.render(container, {
  sitekey: siteKey,
  execution: 'execute',
  appearance: 'interaction-only',
  language: 'zh-tw',
  callback: (token) => resolve(token),
  'error-callback': () => reject(new Error('人機驗證沒過，請再試一次。')),
  'timeout-callback': () => reject(new Error('人機驗證逾時，請再試一次。')),
})

turnstile.execute(widgetId)
```

幾個實作細節：

- token 只能用一次、有效期 5 分鐘，所以每次送出前重置 widget
- 外層再加 60 秒 timeout，避免使用者卡在挑戰時按鈕一直轉
- 驗證失敗就顯示訊息、留在手動五題，**不呼叫模型**
- widget 只在有一句話輸入框的那一步建立；沒開 AI 的站台完全不會載入 Cloudflare script

送出時把 token 一起帶上：

```ts
const token = await gate.execute()
const parsed = await requestParse(endpoint, text, { turnstileToken: token })
```

## Worker：server 端才是真正的門

```ts
const body = new FormData()
body.append('secret', env.TURNSTILE_SECRET)
body.append('response', token)
body.append('remoteip', ip)

const response = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
  method: 'POST',
  body,
})
const result = await response.json()

if (result.success !== true) {
  return json({ error: 'turnstile failed', code: 'turnstile' }, 403, origin)
}
```

- Secret 用 `wrangler secret put` 存進 Cloudflare，不進 repo
- 403 也要帶 CORS header，前端才讀得到 `code`、顯示正確訊息
- 沒設 secret 時會跳過驗證（本機開發方便），但 production 一定要設，否則等於沒開
- 上線順序：先部署帶 site key 的網站，再設 Worker secret；反過來會讓所有 AI 請求先 403

## 費用

Turnstile 免費方案就是無限次驗證，沒有按次計費；20 個 widget、每個 widget 10 個 hostname，對這種小專案很夠。真正有額度的是 Workers AI（免費 10,000 neurons/day）和 Worker 每日請求數；Turnstile 不增加費用，反而是用來保住前者。

## 實測

- 不帶 token：`403 {"error":"turnstile failed","code":"turnstile"}`，模型沒被呼叫
- 帶假 token：一樣 403，代表真的送到 siteverify 驗證過
- 正常瀏覽器：通過後 token 會出現在 POST body，AI 正常填好五題
- headless 自動化：會被跳挑戰，60 秒後顯示「人機驗證逾時」——這正是要的效果

## 踩到的坑

1. **`wrangler secret put` 的參數是名稱，不是值。** 指令是 `npx wrangler secret put TURNSTILE_SECRET`，按下 Enter 後才輸入 secret；名稱打錯的話，`env.TURNSTILE_SECRET` 會是 undefined，程式就跳過驗證。用 `wrangler secret list` 確認名稱。
2. Site key 公開沒關係；Secret Key 不能進 git、不能貼在 issue 或對話裡。
3. 403 的 CORS header 不能忘，否則前端只會看到 `Failed to fetch`，而不是「人機驗證沒過」。
4. 官方 test key 只適合本機；test secret 會讓所有 token 都通過，放到 production 等於開後門。
5. Widget 的 hostname 要包含正式網域；需要本機測試就再加 `localhost`。

## 結語

這次改動的程式量不大：前端一個 Turnstile 模組、Worker 一次 siteverify、加上一個設定欄位。但它把「偽造 Origin 打 API」這個洞補起來，而且對正常使用者幾乎無感——大多數人根本不會看到 widget。對於放在 GitHub Pages、只有一支小 Worker 的專案，免費、不用搬 CDN、設定也不複雜，蠻好用。

專案：<https://github.com/DoubleChuang/cooldown_flow>
網站：<https://doublechuang.github.io/cooldown-flow/>
