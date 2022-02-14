---
title: LDM / STM 為何被移除？
date: 2020-10-25 19:43:19
tags: aarch64
---
## 前言
日前在工作上實作 `arm m0` 的 `context switch` 時，使用 `STM` 指令來進行大量的暫存器到記憶體的數值保存。而在實作樹莓派的 `context switch` 時，發現參考的代碼使用 `STP` 指令。 `STP` 指令一次只能寫入兩個暫存器，因此在代碼編寫上需要大量重複。在實驗後發現 `aarch64` 並未提供 `STM` 指令，因此開始尋找背後的原因。

## 原因
在 [Cortex-A57 Software Optimization Guide
Software Optimization Guide](https://developer.arm.com/documentation/uan0015/b/) 中於 `4.5 Load/Store Throughput` 提到，由於 `Cortex-A57` 處理器支援在一時間周期中同時進行讀寫的指令。換句話說，當指令為一讀取指令與一寫入指令相鄰時，以上兩條指令可以再單一 `cpu` 周期中被執行完畢，反之亦然。因此手冊中建議要將讀寫指令交插使用以提升效率。
建議寫法如下:
```
Loop_start:
SUBS r2,r2,#64
LDRD r3,r4,[r1,#0]
STRD r3,r4,[r0,#0]
LDRD r3,r4,[r1,#8]
STRD r3,r4,[r0,#8]
LDRD r3,r4,[r1,#16]
STRD r3,r4,[r0,#16]
LDRD r3,r4,[r1,#24]
STRD r3,r4,[r0,#24]
LDRD r3,r4,[r1,#32]
STRD r3,r4,[r0,#32]
LDRD r3,r4,[r1,#40]
STRD r3,r4,[r0,#40]
LDRD r3,r4,[r1,#48]
STRD r3,r4,[r0,#48]
LDRD r3,r4,[r1,#56]
STRD r3,r4,[r0,#56]
ADD r1,r1,#64
ADD r0,r0,#64
BGT Loop_start
```
可以發現上面的範例程式將讀寫指令交錯放置，這樣可以大量減少記憶體存取所需的 `cpu` 周期。

## 結論
在 `aarch64` 架構下，如果要進行資料的讀寫，建議將讀/寫指令交叉放置。這樣可以藉由 `cpu` 架構上的設計來提高效率。
而 `LDM/STM` 指令則鼓勵一次大範圍的讀或寫。這樣變無法發揮該架構下的優勢，因此這兩個指令便遭到移除。

## Reference
[你猜 为什么A64为什么没有LDM和STM指令了，而是用LDP跟STP呢？](https://www.jianshu.com/p/62ea9cfecf80)
[Cortex-A57 Software Optimization Guide
Software Optimization Guide](https://developer.arm.com/documentation/uan0015/b/)
