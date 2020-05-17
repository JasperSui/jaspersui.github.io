---
layout:     post
title:      "[Python] Generator 特性及實作場景"
subtitle:   "Talk is cheap, show you the code."
date:       2020-05-13 22:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
    - 技術筆記
---

## 前言

因為想要了解更多有關 Python 的底層實作機制，還有更 Pythonic 的寫法，最近在讀 [Fluent Python](https://www.amazon.com/Fluent-Python-Concise-Effective-Programming/dp/1491946008) 和 [Effective Python](https://effectivepython.com/)，內容都有提到 Generator，自己實務上比較少運用到，想要隨手紀錄起來順便理一理思緒。

---

### 我的開發環境
> - macOS Catalina
> - Python 3.7.7

## 正文

目前只有想到兩個場景可能會比較常用到 Generator 的特性：

- 讀取大量資料的檔案
- 迴圈因為需要迭代且回傳同一變數而耗費大量記憶體

這邊有兩個名詞要分清楚：

1. 可迭代的 iterable (Adj.)
2. 迭代器 iterator (Noun.)

要能夠 iterable，必須要實作 iterator 的 `iterator.__iter__()`，這個 function 會回傳一個 iterator，用來一次回傳一個成員的物件。有些 iterable 的物件 (e.g. List) 是將值存在記憶體中，而其他就是像 iterator，沒有把所有的值存在記憶體，只有正在執行的時候才會產生值。

iterator 除了 `iterator.__iter__()` 要實作之外，還需要 `iterator.__next__()`，當調用 `next()` 的時候，會做幾件事：

- 更新自身 (iterator) 狀態，將自己指到下一個位址，如果下一個位址沒有值，狀態就會變為 StopIteration
- 回傳目前的結果（類似於 pop），回傳完之後值就會從 iterator 中消失，都回傳後就變成空的 Generator 容器 (not None)

直接進到場景可能會直觀一點，假設今天有個需求是要讀取一個 `data.csv` 檔案，並列出這個檔案的行數：

```python
import csv
def new_test_csv():
    with open('data.csv', 'w', newline='') as f:
        writer = csv.writer(f)
        for i in range(1000000000):
            writer.writerow([i])
```

```python
def file_reader(file_name):
    with open(file_name) as f:
        return f.read().split('\n')

file_generator = file_reader('data.csv')
row_count = 0

for row in file_generator:
    row_count += 1

print(f"Total row count: {row_count}")
```

在數量小的情況下，這樣做沒什麼太大的問題，但如果今天如果 `data.csv` 的行數很多，就有可能會產生 `MemoryError` 的例外狀況，這時候如果我們將 `file_reader()` 改寫一下：

```python
def file_reader(file_name):
    with open(file_name) as f:
        for row in f:
            yield row
```

在記憶體內就會需要去儲存迴圈內每一次加入後的結果，但實際上我們要的只是最後的那個 result，並不想要短時間內被佔用太大的記憶體空間。

#TODO 產生CSV的FUNCTION
#TODO MEMORY ERROR 的範例
#TODO 費氏數列
#TODO FOR...IN... GENERATOR



## 參考

[Python Generator](https://lotabout.me/2017/Python-Generator/)