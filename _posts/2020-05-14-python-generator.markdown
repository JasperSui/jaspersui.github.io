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

- 更新自身 (iterator) 狀態，將自己指到下一個位址，如果下一個位址沒有值，狀態就會變為 `StopIteration`
- 回傳目前的結果（類似於 pop），回傳完之後值就會從 iterator 中消失，都回傳後就變成空的 Generator 容器 (not None)

直接進到場景可能會直觀一點，假設今天有個需求是要讀取一個 `data.csv` 檔案，並列出這個檔案的行數：

```python
# 產生 data.csv 的 Function，可以用來直接產生測試檔案

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

在行數小的情況下，這樣做沒什麼太大的問題，但如果今天如果 `data.csv` 的行數很多，像是現在的 10 億行，就有可能會產生 `MemoryError` 的例外狀況，這時候如果我們將 `file_reader()` 改用 `Generator` 的邏輯來寫：

```python
def file_reader(file_name):
    with open(file_name) as f:
        for row in f:
            yield row
```

只要一個 Function 有用到 `yield`，那去呼叫這個 Function 的時候，就會回傳一個 `<generator object>`，如下圖：

![img](https://i.imgur.com/UUdwEuF.png)

那這個又得先提到 `for...in...` 迭代的條件：`for...in...` 的對象必須是 **可迭代的 (iterable)**。

如同前面提到的名詞解釋，這個對象必須實現 `iterator.__iter__()` 這個方法，而透過呼叫這個可迭代對象的 `__iter__()` 方法，就可以得到這個對象所對應的迭代器，再透過 for 迴圈不斷地調用 `__next__()` 來去得到所有的值，直到迭代器拋出 `StopIteration` 的例外狀況，也就代表這個迭代器已經沒有下一個值，for 迴圈就會自動去處理這個例外狀況並跳出迴圈。

在這邊我們改寫了 `file_reader()`，讓這個 Function 回傳一個 `<generator object>`，並指給 `file_generator` 這個變數：

```python
file_generator = file_reader('data.csv')
row_count = 0

for row in file_generator:
    row_count += 1
```

同時，將 `file_generator` 用 `for...in...` 的方式不斷去調用 `__next__()` 來取得下一次的元素，在每一次的取得元素的時候再去指派記憶體空間給它就可以了，這麼做可以大幅度地減少浪費不必要的記憶體，而不需要像原先用 `return` 的方法，在進入到 for 迴圈的時候，就必須要先指派給所有資料記憶體空間，就有可能因為資料量太大而導致 `MemoryError` 的例外狀況發生。

---

順便提供一個費氏數列 (Fibonacci) 原先的實作方式及用 `Generator` 的比較：

#### 一般作法

```python
def fibonacci(times):
    n, a, b = 0, 0, 1
    while n < times:
        print(f'[No.{n+1}] Result: {b}')
        a, b = b, a + b
        n = n + 1
    return True

#>>> fibonacci(10)

#[No.1] Result: 1

#[No.2] Result: 1

#[No.3] Result: 2

#[No.4] Result: 3

#[No.5] Result: 5

#[No.6] Result: 8

#[No.7] Result: 13

#[No.8] Result: 21

#[No.9] Result: 34

#[No.10] Result: 55
```


#### Generator 作法

```python
def fibonacci(times):
    n, a, b = 0, 0, 1
    while n < times:
        yield (f'[No.{n+1}] Result: {b}')
        a, b = b, a + b
        n = n + 1

results = fibonacci(10)
for result in results:
    print(result)

#[No.1] Result: 1

#[No.2] Result: 1

#[No.3] Result: 2

#[No.4] Result: 3

#[No.5] Result: 5

#[No.6] Result: 8

#[No.7] Result: 13

#[No.8] Result: 21

#[No.9] Result: 34

#[No.10] Result: 55
```

## 參考

[Python Generator](https://lotabout.me/2017/Python-Generator/)