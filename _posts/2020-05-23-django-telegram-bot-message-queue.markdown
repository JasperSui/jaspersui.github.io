---
layout:     post
title:      "[Django] python-telegram-bot 的 MessageQueue 與 Django 難以兼容的問題"
subtitle:   ""
date:       2020-05-23 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - Python
    - Telegram
---
## 前言

閱讀前可以先服用 `python-telegram-bot` 的 [Github](https://github.com/python-telegram-bot/python-telegram-bot)，可以先看一下他們是如何在 Python 實作這個 Telegram Bot 的。


### 我的開發環境
 - macOS Catalina
 - Python 3.6.2

## 正文

因為想用 [Telegram Bot](https://core.telegram.org/bots/api) 搭配 Django + Celery 的 Perioic Task 來執行每分鐘發送提醒的功能，但是發了超過 **20** 則後發現都會收到已經超過送出上限的 Response，去查了一下才發現 Bot 透過 API 來傳送訊息到群組有以下兩個限制：

1. 20 則訊息/分鐘
2. 30 則訊息/秒

只好找看看有沒有現成去針對 Telegram Bot 的情境而設計的 Message Queue 可以直接使用，結果發現 `python-telegram-bot` 就有提供擴充 Library，而裡面就正好有針對限制而設計好的 [MessageQueue](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.messagequeue.html)。

這邊引用一下 `python-telegram-bot` [Wiki](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Avoiding-flood-limits) 中的資料，，來看看他們怎麼去設計這個 MQ 的：

![img](https://cloud.githubusercontent.com/assets/16870636/23493181/82244f60-ff12-11e6-8fd7-679fdbd16b04.png)

![img](https://user-images.githubusercontent.com/16870636/28248541-e7cfca4a-6a4e-11e7-84e8-ad1992e21fd4.png)

其實流程很簡單，就是先去檢查要發送的 `bot.send_message(chat_id, text)` 的 `chat_id` 是否為一個 `群組 (Group)`，不是的話就只需要看看現在有沒有達到「1 秒內最多送出 30 則訊息」的限制，之後就可以直接送出。如果是群組的話，就需要再多檢查一個有沒有已經在「1 分鐘內最多送出 20 則訊息」。

而 `telegram.ext.messagequeue` 背後實作的方式詳細可以直接讀 [Source Code](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/telegram/ext/messagequeue.py)，行數不多而且脈絡清楚，看懂不需要花太多時間，這邊就只簡單帶過有關這篇的重點概念：

#### 1. 每一個 `MessageQueue` 會使用到 **2** 個 `DelayQueue`，一個用於所有訊息，另一個只用於群組訊息

在使用 `queuedmessage.MessageQueue()` 來實體化一個 `MessageQueue` 時，如果不帶任何參數，那 `auto_start` 就會是預設的 True，底下的 `_all_delayq` 和 `_group_delayq` 被實體化時也就會同時調用 `threading.Thread.start()` 來開始執行，這邊等等還會細講，**同時這也是導致 python-telegram-bot 的 MQ 沒辦法和 Django 兼容的原因。**

---

#### 2. DelayQueue 繼承於 threading.Thread，同時也是 Non-daemon Thread

> However, you should be aware that callbacks always run in non-main (DelayQueue) thread. That's not a problem for Python-Telegram-Bot lib itself, but in rare cases you may still need to provide additional locking etc to thread-shared resources. So, just keep that fact in mind.

上面是在 [Wiki](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Avoiding-flood-limits) 中提到的一段描述，在 `telegram/ext/messagequeue.py` 的第 100 行，可以看到 DelayQueue 的 `__init__()` 中就已經設定：

```python
self.daemon = False
```

可以看看[這篇](https://myapollo.com.tw/zh-tw/python-daemon-thread/)，會對 Non-daemon Thread 有更深的瞭解，既然 `DelayQueue` 是 Non-daemon 的，在主程式被 Kill 掉之後，只要這個 Thread 該做的事還沒做完，就沒辦法結束。

那 `DelayQueue` 該做的是什麼呢？**不間斷地去監控要進出這個 Queue 的 Telegram Mesaage 來確保訊息正常送出不遺失**，因此，只要開始運行後就不會去主動 `stop()`。

而我們的主程式，也就是 Django，除了在運行伺服器的時候會去讀專案的 `.py` 檔，還有兩個情況也會：

1. `python manage.py migrate`
2. `python manage.py makemigrations`

在進行這兩個動作都會去把和 Django 有關聯的 `.py` 檔都 Load 出來，那好了，如果你的程式碼裡面有用到`queuedmessage.MessageQueue()` 來實體化一個 `MessageQueue`，在動作執行完的時候，MainThread 會被 Kill，但 MQ 底下的兩個 `DelayQueue (Thread)` 不會伴隨著 MainThread 的生命週期一起被終止，而是繼續著他們的任務。

這樣會導致每次要 `migrate` 和 `makemigrations` 的時候，都會因為 Non-Daemon Thread 還在繼續運行而無法照著正常流程來完成，在原本的任務結束後都要手動去強制 Kill 這些 Thread 才能去完成 `migrate` 或 `makemigrations`，若是專案有引入 CI/CD 的話，這樣會造成很大的困擾。

但或許可以在 `migrate` 和 `makemigrations` 結束時，用 `threading.enumerate()` 來找出所有的 `DelayQueue`，並將他們一一 `stop()`，應該就能夠避免這個問題，有興趣的人可以參考 Django 官方文檔的 [post_migrate](https://docs.djangoproject.com/en/3.0/ref/signals/#post-migrate) Signal 去實作看看。

## 總結

主要還是因為會影響 CI/CD 的流程，所以還是棄用了這個完整度很高的 Message Queue，有點可惜，而我這邊則是改用快取的方式來自行實作訊息發送的 Queue，花了些時間但是至少能確保訊息能最有效率地被送出而不遺失。

如果有更好的辦法請在下方留言告訴我！謝謝你讀完這篇文章，希望能帶給你一些收穫！

## 參考

[python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot)<br>
[Avoiding flood limits](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Avoiding-flood-limits)<br>
[Python daemon thead 解說](https://myapollo.com.tw/zh-tw/python-daemon-thread/)<br>
[Django Signals](https://docs.djangoproject.com/en/3.0/ref/signals/#post-migrate)