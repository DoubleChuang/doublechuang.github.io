---
title: "【Linux Kernel】修正 FUSB302 USB-C/DP Alt Mode 的 Pin Assignment 判斷"
date: 2023-11-14
categories: [開發筆記]
tags: [Linux Kernel, USB-C, DisplayPort, PD, FUSB302]
description: "修正 fusb302 驅動在 DisplayPort Alternate Mode 協商時,因 signal capability 與實際 pin support 不一致而選錯 pin assignment 的問題"
---

> 在 USB Type-C / Power Delivery 的 DisplayPort Alternate Mode 協商中,裝置可能在 signal capability 與實際支援的 pin assignment 上不一致,導致驅動選錯 pin。本篇記錄在 Rockchip 4.19 kernel 的 `drivers/mfd/fusb302.c` 所做的修正。

## 背景

在 USB-C 走 DisplayPort Alternate Mode 時,DFP(下行埠)要決定使用哪一組 pin assignment(A~F)。這些 pin 分屬兩類信號:

- **USB Gen 2 信號**(pin A、B):以 USB3.1 Gen2 的速度承載 DP
- **DP v1.3 信號**(pin C~F):以 DP v1.3 信號承載

協商時,DFP 會從對端裝置(Lightware)的 DP capability VDO 裡讀出:

- `PD_DP_SIGNAL_GEN2`:是否宣告支援 USB Gen 2 信號
- pin capabilities:Lightware 裝置實際支援哪些 pin

問題就出在:這兩者**不一定一致**。例如 Lightware 裝置的 DisplayPort handshake 同時送了 `PD_DP_SIGNAL_GEN2` 和 `PD_DP_SIGNAL_DP1_3` 兩個 signal bit,但它的可設定 pin **並不支援 USB Gen 2**。

## 修正前的邏輯

原始程式碼只看 `PD_DP_SIGNAL_GEN2` 一個 flag,就直接在兩類 pin 之間二選一:

```c
/* revisit if DFP drives USB Gen 2 signals */
if (PD_DP_SIGNAL_GEN2(caps))
    pin_caps &= ~MODE_DP_PIN_DP_MASK;
else
    pin_caps &= ~MODE_DP_PIN_BR2_MASK;
```

意思是:只要宣告 Gen2,就丟掉所有 DP pin(pin C~F);否則丟掉 USB Gen2 pin(pin A、B)。**沒有檢查對端裝置是否真的支援這些 pin**,遇到 signal/cap 不一致時就會選出一個不支援的 pin assignment。

## 修正方式

改成「雙重確認」——只有當 signal 與實際 pin support 都成立時才採用對應的 pin:

```c
#define PD_DP_SIGNAL_DP1_3(x)   (((x) >> 2) & 0x1)

dp_pin_support = pin_caps & MODE_DP_PIN_DP_MASK;   // DP v1.3
usb_gen2_pin_support = pin_caps & MODE_DP_PIN_BR2_MASK; // USB GEN2

if (PD_DP_SIGNAL_GEN2(caps) && usb_gen2_pin_support) {
    pin_caps = usb_gen2_pin_support;
} else if (PD_DP_SIGNAL_DP1_3(caps) && dp_pin_support) {
    pin_caps = dp_pin_support;
} else {
    pin_caps &= ~(MODE_DP_PIN_DP_MASK | MODE_DP_PIN_BR2_MASK);
}
```

規則:

1. 只有在**同時**宣告 USB Gen 2 signal 且對端裝置有支援 USB Gen2 pin 時,才選用 USB Gen2 pin
2. 否則,只有同時宣告 DP v1.3 signal 且對端裝置有 DP pin 時,才選用 DP pin
3. 兩者皆不符時,清掉 DP / USB pin,避免協商出不支援的模式

## 結語

這次修正源自 Lightware 裝置在 handshake 中 signal 與 pin capability 不一致所觸發。透過在 `pd_dfp_dp_get_pin_assignment()` 中交叉比對 signal 與 pin support,確保選出的 pin assignment 一定在對端裝置支援範圍內,避免 USB-C 轉接或顯示異常。
