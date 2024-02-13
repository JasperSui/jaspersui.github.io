---
layout:     post
title:      "用 Python 實作 LRU Cache 機制"
subtitle:   ""
date:       2020-09-27 02:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
    - Cache
    - LRU
---
## 前言

一陣子沒有寫文章了，最近沒有什麼靈感，前陣子在看一些 System Design 的文章時，看到題目中有提到 LRU Cache 機制，覺得挺有意思的就想著要來用 Python 來實作一個簡單的 LRU Cache 機制。

因為要有效率地運用電腦有限的空間，不能把所有要記錄的東西都存在某個儲存空間上，所以必須要有個機制來汰換資料，而汰換資料的方法大略分為下列四種：

1. FIFO (First In, First Out)
2. NMRU (Not Most Recently Used)
3. LFU (Least Frequently Used)
4. LRU (Least Recently Used)

在 CPU 快取的角度來說，當 CPU 發出 RAM 存取請求時，會先查看 Cache 內是否有請求資料。如果存在（命中），則不透過存取 RAM 的過程而直接返回資料。如果不存在（失效），則要先把 RAM 中的資料先存到 Cache 內，再把資料丟給 CPU。

上述第三種和第四種其實規則非常相似，而 LRU 就是我們今天要講的主角，在一個有限的空間內，若是資料已經達到空間上限時，就要進行所謂的「汰換」，而根據 LRU 的定義，我們就需要淘汰最近最少使用的資料，這邊我們用 Python 來實作。

## LRU 是什麼

用生活上的例子來說明 LRU Cache 機制是最簡單不過的了，之前有看到有人用衣服放在衣櫃裡的情境來形容，這邊借用一下圖片。

![img](https://i.imgur.com/dFPNFYl.jpeg)

可以想想自己衣櫃平常是怎麼被使用的，常穿的衣服一定都是衣櫃靠近前面那幾件，要用的時候拿出來穿，穿完之後洗乾淨再掛回衣櫃的前面，一直沒有在穿的衣服就被堆到衣櫃的後方，直到哪一天突然想到一件很久沒穿的衣服，拿出來穿完洗乾淨之後再放回衣櫃的前面。

那衣櫃的空間不可能無限地成長啊！所以這時候媽媽（限制）說話了：「是時候把你衣櫃整理整理了！」這時候的你一定會苦惱要丟那些衣服，套用 LRU Cache 機制的話，這時候你就會選擇把衣櫃最後方最不常穿的衣服丟掉。

### 實作

要實作 LRU Cache 有好幾種方法，這邊簡單提一下：

1. LinkedHashMap
2. Hash Map + Double Linked List

在 Python 中，我們很間單就能夠使用第二個方法提到的兩個資料結構，分別是 Dictionary (Hash Map) 及 Deque (Double Linked List)，在 **Get** 和 **Set** 的時間複雜度都會是 O(1)，非常適合用來實作這個機制，在下面我們會將把衣服放回至衣櫃的最前面叫做 **Refresh** 。

#### Get(Key)

回傳 Cache 中 Key 的 Value 並將 **Refresh** 該 Key，若 Key 不存在則傳回 None。

#### Set(Key, Value)
分為兩種情況，用 Key 在不在 Cache 內來分：

1. 若 Key 在 Cache 裡：代表空間一定是夠的，直接 Refresh 該 Key 就好
2. 若 Key 不在 Cache 裡：如果空間已經滿了，就把最尾端的資料 Pop 掉，再把 Key 放進隊伍的頭；如果空間還沒滿，就直接把 Key 放進隊伍的頭。

構思了兩個簡單的 Function，接下來就來實作吧！

```python
from collections import deque

class LRUCache:
    def __init__(self, size):
        self.size = size
        self.hash_map = dict()
        self.queue = deque()
    
    def get(self, key):
        if key in self.hash_map:
            # 將 Key 從 Queue 中拿出並放到最前面
            
            self.queue.remove(key)
            self.queue.appendleft(key)
            return self.hash_map[key] # 回傳該 Key 的值

    
    def set(self, key, value):
        if key not in self.hash_map:
            if len(self.queue) == self.size:
                deleted_key = self.queue.pop() # 將最後一個順位的 Key pop 掉

                self.queue.appendleft(key)
                self.hash_map[key] = value
                del self.hash_map[deleted_key] # 清除 Dict 裡面不必要的 Key

            else:
                self.hash_map[key] = value
                self.queue.appendleft(key)
        else:
            # 若 Key 已經在 HashMap 裡面就直接 Refresh 該 Key 即可

            self.queue.remove(key)
            self.queue.appendleft(key)
            self.hash_map[key] = value

cache = LRUCache(3)
cache.set("A", 1) # cache.deque = ["A"]

cache.set("B", 2) # cache.deque = ["B", "A"]

cache.set("C", 3) # cache.deque = ["C", "B", "A"]


cache.get("A") # 1, cache.deque = ["A", "C", "B"], A is refreshed

cache.get("D") # None

cache.set("D", 4) # cache.deque = ["D", "A", "C"], B is poped

cache.set("C", 5) # cache.deque = ["C", "D", "A"], C is refreshed

```

## 總結

實作過一遍之後就會對概念更加清楚，強烈建議有興趣的讀者自己也實作一遍，理解衣服與衣櫃的概念之後，透過自己的方式去構思，試看看自己寫出 Get 和 Set 的 Function，之後對於一些 System Design 的題目要求也會有一些基本的概念！

## 參考

[LRU & LFU 緩存機制的原理及實現](https://zhuanlan.zhihu.com/p/120423040)<br>
[如何使用 Python 實現 LRU Cache 快取置換機制](https://blog.techbridge.cc/2019/04/06/how-to-use-python-implement-least-recently-used/)<br>
[記憶體延遲：因與果](https://www.jishuwen.com/d/2EVz/zh-tw)<br>
[快取文件置換機制](https://zh.wikipedia.org/wiki/%E5%BF%AB%E5%8F%96%E6%96%87%E4%BB%B6%E7%BD%AE%E6%8F%9B%E6%A9%9F%E5%88%B6)<br>
