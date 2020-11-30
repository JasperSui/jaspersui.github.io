---
layout:     post
title:      "利用 Sentry 實作分散式追蹤 (Distributed Tracing)"
subtitle:   ""
date:       2020-11-29 02:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Sentry
    - Distributed Tracing
---
## 前言

<a href="https://sentry.io/"><img src="https://vectorlogoseek.com/wp-content/uploads/2020/02/sentry-io-vector-logo.png" style="max-width:50%;"></a>

目前手上碰的專案都是用 [Sentry](https://sentry.io/) 來協助追蹤 API 和 Asynchronous Tasks 的，可以用一種識別碼來作為 Filter 的一種粒度，篩出對應服務的 Issue 和 Performance。

整體用起來的效果很好，每個月的費用也很親民，尤其對於 Python 用戶體驗更是一流，強烈建議若是有相關需求的話可以引入 Sentry 到專案內，可以省去不少麻煩。

本篇會大量用到 Sentry 官方文檔提供的資源來說明，將前提建立在你是 Sentry 的用戶下，只要懂了基本的概念就可以應用到各種情境，讓焦點不被其他雜訊模糊。

以下就會介紹最近讀到的一種**微服務 (Microservices)** 專案架構下常使用的 **Tracing Solution ------ 分散式追蹤 (Distributed Tracing)**，大略說明如何透過 Sentry 來幫助我們快速達成目的，並探討一些 Use Case！

## Distributed Tracing

**「Tracing 做得好，Debug 沒煩惱。」**這句話對我來說算是蠻中肯的。

誰都不想在大半夜的時候收到電話，挺起身子打開電腦，連到機器裡面，在茫茫 Log 海中尋找那一個 Error Message。情況好一點的時候可能花個幾分鐘就找到了，運氣不好，要查的時候 Log 早就被洗到不知道哪去了。

甚至是在 A 服務查了老半天，結果回過神來才想起來這個錯應該是會報在 B 服務的，找到原因後才發現一個小時已經過去了。

上述捶心肝的情境應該或多或少都有發生過，遇到一次就夠你受的了。有沒有想過，如果能直接搜尋 API 請求，就能找到請求對應到的所有相關服務的監控資訊，是多棒的一件事！不必再去從一堆 Log 中找出你要的訊息，也不必在各個服務的機器中跳來跳去，只要輸入一個 **Unique Transaction ID**，就能直接定位到是哪個服務中出現的錯誤訊息。

這就是本文的重點 ------ **分散式追蹤 (Distributed Tracing)**，這邊會用到的單位有 **Trace**, **Transaction** 和 **Span**：

![img](https://docs.sentry.io/static/1ae959bb1d05b01379cf856c5dc36a01/d61c2/diagram-transaction-trace.png)
#### Trace

- 一整個你想追蹤的動作 (Operation) 的所有紀錄
- 當想追蹤的動作為跨服務時，就叫做 Distributed Trace
- 由一個或多個 Transaction 組成

#### Transaction

- 由一個服務中的所有 Spans 組成
- 有一個 Root Span，下面會有 N 個 Parent Spans 和 Child Spans

#### Span

- 一個服務最少會有一個 Span (Root Span)
- 每個 Span 都可能會有自己的 Parent Span 和 Child Span
![img](https://docs.sentry.io/static/7de716d4f17ae73fccfe287c11141be1/e67dd/diagram-transaction-spans.png)

## Use Case

看了架構圖後，應該多多少少都有一些觀念了吧！

先來看一段 Sentry 提供的 Feature Demo，看完之後你會更清楚！


<iframe src="https://www.youtube.com/embed/zoBM5mGTsh4" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
既然我們要的是追蹤一個 API Request 的生命週期，也就是 **Trace**，那我們勢必就得有個值來判斷某個 Transaction 是屬於哪個 Trace 的，我最常碰的就是 Web Application，就以這個為例，通常我們一個 Request 的流程如下：

<h4 style="text-align: center;"> 前端 → A 服務 → B 服務 → A 服務 → 前端 </h4>

這邊看起來最簡單的解法就是，在前端發送 Request 給 A 服務的時候，就透過某些 HTTP Request 的參數算出 Hash 值來當作這個 Request 的 Unique Transaction ID。

假設產生了一組 `6ecd3698bac76bb4fec4e6c61996487f`，就把他放在 Request Header 中，並在這包數據要送給 Sentry 之前一起帶上。之後 Sentry 的前端 Project 中就會有一個 Unique Transaction ID 為 `6ecd3698bac76bb4fec4e6c61996487f` 的 Request Event。

在 A 服務收到前端傳來的 Request 時，就要先看看這個 Request 的 Header 裡面有沒有 Unique Transaction ID，如果有的話就要在 A 服務將 Request 資訊送給 Sentry 前一起帶上，同理，Sentry 的 A 服務 Project 也會有一個 Unique Transaction ID 為 `6ecd3698bac76bb4fec4e6c61996487f` 的 Request Event。

剩下的 `B 服務 → A 服務 → 前端` 步驟也是依樣畫葫蘆，所以這個 API Request 在 Sentry 裡面總共會有 **5** 個 Events，今天想要查詢某個 API 有沒有報錯，然後報錯的同個 Request 下所有的 Events 有哪些，都可以透過一次撈出來。


## 總結

在這個微服務架構當道的時代，要去撈出特定的 Log 是越來越麻煩了，如果專案本來就有在使用 Sentry 來協助監控，那就可以利用 Sentry 來搭上順風車，輕輕鬆鬆完成跨服務間的 Request 監控，不管是要 Debug 或是 Optimize，都可以跳脫以往只能單服務單 Event 來 Trace 的形式，用真實用戶的角度來看整個流程慢在哪個環節或是錯在哪個服務。

## 參考

[Sentry Distributed Tracing Docs](https://docs.sentry.io/product/performance/distributed-tracing/)<br>
[Distributed Tracing with Sentry: How to Find the Root Cause of Errors Across Applications](https://www.youtube.com/watch?v=zoBM5mGTsh4)<br>
[A Quick Introduction to Distributed Tracing](https://newrelic.com/resources/ebooks/quick-introduction-distributed-tracing)<br>
[一文搞懂基於zipkin的分散式追蹤系統原理與實現](https://www.mdeditor.tw/pl/2H50/zh-tw)<br>