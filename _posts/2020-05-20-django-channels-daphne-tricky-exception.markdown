---
layout:     post
title:      "[Django] Channels 的 took too long to disconnect 報錯"
subtitle:   ""
date:       2020-05-20 12:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - WebSocket
    - 技術筆記
---

## 前言

今天在和同事去 Trace 為什麼 [Django Channels 2](https://channels.readthedocs.io/en/latest/) 會有 `took too long to disconnect` 和 `Exception inside application: Lock is not acquired.` 報錯的時候，覺得要重現這個問題的過程以及作者和社群的討論很有趣，想記錄一下。

---
### 我的開發環境
 - macOS Catalina
 - Python 3.6.2

## 正文

在遇到這個問題的時候，一開始是發現 WebServer（我們是用 [Daphne](https://github.com/django/daphne)） 來接收瀏覽器在建立連結後的 Channel 送訊息的時候，可能會報這個錯。

但無論我們怎麼送，去看一下 `Consumers.py` 裡面的 `receive()` 函數需要接收甚麼樣格式的資料，刻意修改成錯誤的格式。發現 [Channels](https://channels.readthedocs.io/en/latest/) 其實都將這些例外狀況 Handle 得很好，因為故意讓 [Channels](https://channels.readthedocs.io/en/latest/) 接收錯誤格式的資料並沒有發生甚麼奇怪的例外，會發生的錯誤只是資料處理的錯誤，例如 `Dict` 的 `KeyError` 之類的，和我們想要重現的例外沒有任何關係。

再加上後來有去找到一篇作者有在回應的 [Github Issue](https://github.com/django/channels_redis/issues/81)，其中 [Django Channels](https://channels.readthedocs.io/en/latest/) 的作者 [andrewgodwin](https://github.com/andrewgodwin)，同時也是 [Django](https://www.djangoproject.com/) 的核心開發者，有回應了這麼一段：

![img](https://i.imgur.com/NSpfnnV.png)

於是我們開始往 Django 和 Worker 的方向去找，讓 `receive()` 函數在接收到資料時不要馬上回傳給 Client 端，而是 `sleep()` 之後在做一些操作，由於我們 Channels 是用 `Async` 來實作的，所以要必須用到 `asyncio` 的 `sleep()`。

```python
import asyncio

async def receive():
    # Start...
    await asyncio.sleep(30)
    # End...
```

之後我們模擬了 Client 連線後 `Send()` 資料給 Django 的情境，用簡單的 `ping` 和 `pong` 來測試，打 `ping()` 過去給 Django，但因為這邊有 `await asyncio.sleep(30)` 所以 Django 在 `receive()` 的時候會睡 30 秒。

起初睡了 30 秒還是可以正常收到 Django 回傳的 `pong`，所以就想說如果在 Django sleep() 的時候，直接將 [Daphne](https://github.com/django/daphne) Restart 的話會發生甚麼事，結果就這麼剛好 Django 就報了 `took too long to disconnect` 的例外。

當下以為找到重現情境的辦法了，所以當下又馬上重做了一模一樣的步驟，但是這次我們將 `await asyncio.sleep(30)` 的 30 秒改為 5 秒，並在 Django 睡著的期間同樣去重啟 Worker，並同時一直 `Send(ping)` 給 Django，原本想說等 Worker 重啟完之後 Django 就會收到這個例外報錯了，但是意外的是這次沒有報錯了，我們不死心又試了很多次，結果都是沒報錯。

後來我們又將 `sleep(5)` 改為 30 秒，起初也是沒有和上一次的重現一樣沒有收到任何的例外報錯，不斷地重複以上的步驟：

`Send(ping) to Django -> Django sleep -> Restart worker (Daphne)`

直到不知道第幾次的時候，`took too long to disconnect` 例外又出現了，因此可以得知，**其實 `sleep()` 的秒數不是導致這個例外報錯的主因。**

---

於是只好再去拜讀這個 [Channels_redis Issue #81](https://github.com/django/channels_redis/issues/81)，讀了也同樣遇到這個報錯的留言後，發現幾個蠻重要的 Comments：

![img](https://i.imgur.com/envJBOR.png)

![img](https://i.imgur.com/HRx0PJy.png)

從以上兩個留言可以得知一個線索，**`ASGI_THREADS` 這個參數如果設高可能會延緩發生這個例外的時間。**

最後，看到這個留言之後問題就一切明朗了：

![img](https://i.imgur.com/DgeXfAO.png)

> ...(略)<br>
> At some point every request to the daphne instance will start getting these errors. We're working around it (kind of) by running a number of daphne instances across multiple servers and restarting daily. Would be good to get to the bottom of it though.

從這邊可以發現，透過兩個方法可以解決這個問題：

1. 啟用更多的 Daphne
2. 每隔一段時間重啟 Daphne

而在我們重現出這個情境的步驟中，可以知道，在 Daphne 剛重啟的時候，並不會發生這個例外報錯，而是在我們不斷的去 `Send(ping)` 後才會出現 `took too long to disconnect` 的問題，而 [Channels](https://channels.readthedocs.io/en/latest/) 是搭配 [Redis](https://redis.io/) 來實作的，可以合理猜測：

**Client 每次送訊息給 Daphne 之後，都會產生類似暫存的東西存在 Redis 裡，並且會因為暫存多寡而影響到速度。**

因此，想要解決這個問題的話，其實還是要看到 Channels 背後是如何去處理這塊的，目前就只能透過上述提到的兩種方式來避免這個報錯。

如內容有錯請在下方留言告知，非常樂意討論！

## 參考

[Django Channels 2](https://channels.readthedocs.io/en/latest/)<br>
[Daphne](https://github.com/django/daphne)<br>
[Channels_redis Issue #81](https://github.com/django/channels_redis/issues/81)