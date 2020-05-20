---
layout:     post
title:      "[Python] 肉眼沒辦法找到的 Syntax Error"
subtitle:   ""
date:       2020-05-19 23:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
    - 技術筆記
---

### 我的開發環境
 - macOS Catalina
 - Python 3.6.2
 - Visual Studio Code

## 正文

最近開發的時候遇到了一個 `SyntaxError` 的問題，居然花了我一小時來解…不斷的檢查縮排、`for...in...`、`if...else...` 等等的有沒有語法上的錯誤，曾經解到一度懷疑人生，想說這陣子的 Python 是不是都白學的…，這種問題開口問同事又很尷尬，索性開始 Google 有沒有人遇到相關的問題。不查還好，一查又被各種文章勸去找 Code 裡面到底縮排有沒有縮好。

覺得不可能發生這種事的我就跑去找看看有沒有 Python Syntax Error Checker，結果還真的有這種線上工具，這個工具推薦大家可以加在書籤列裡，不然不知道我還要花多少時間在找這個問題：

 - [Python Syntax Error Checker](https://extendsclass.com/python-tester.html)

抱著姑且一試的心情把 Code 貼上去，發現真的有 Error！而且是沒看過的字元，丟上 Google Chrome 的網址列上只看到極小的 `BS` 字樣，有點像是 `™` 這種符號的大小，但是在 Google 直接搜尋，會被取代成 `?` 字樣。

這是我第一次遇到這種字元，想到之前 Facebook Messenger 有那種傳送某個字元導致 App crash 的案例，所以我就把他丟到 [Telegram](https://telegram.org/) 上面看看會發生甚麼事，結果字元根本就沒出現，但是可以按下送出，只是訊息會一直無法送達，重試也沒辦法。

順帶一提，這個字元連 [Bitbucket](https://bitbucket.org/product/) 的 Diff 上都沒辦法看出有這個字元，但是如果將刪除這個字元後的改動 Commit 上 Repository，Diff 則會顯示這行有新增或刪除（但看起來根本沒有差異）。

這邊我截一小段這個問題在 [Python Syntax Error Checker](https://extendsclass.com/python-tester.html) 上會出現的樣子，會以紅點的方式出現，大家可以試著丟進會報 `SyntaxError` 的 Code 進去檢查：

![img](https://i.imgur.com/nCLMMML.png)

但實際上的 Code 就看不出有這幾個紅點字元存在：

```python
else:
```

這個字元後來我再也無法打出來，後來刪掉那些字元後程式也就正常執行了，那時候真該留下這個字元的，懷疑可能是 `macOS` 的 `VSCode` 在複製貼上的時候可能會不小心帶到這個字元，又或者是可能搭配 `Command + {?}` 個鍵會產生這個字元，按遍了整個鍵盤也沒看到這個字元，可能是組合鍵才能打出來的吧…。

建議如果要把時間花在美好事物上的話，遇到 `SyntaxError` 可以直接丟到這個 Checker 上，可以避免懷疑人生的情況發生，在此對這個 Checker 的作者致上崇高的敬意。 

## 參考

[Python Syntax Error Checker](https://extendsclass.com/python-tester.html)