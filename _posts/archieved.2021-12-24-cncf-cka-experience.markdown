---
layout:     post
title:      "CNCF Certified Kubernetes Administrator (CKA) 證照心得"
subtitle:   ""
date:       2021-12-24 02:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - CNCF
    - CKA
    - Kubernetes
    - Certification
    - Experience
---
## 前言

睽違了十個月的發文，累積了一些想法都還沒有變成文章，最近會慢慢來把之前想好的主題都紀錄下來的，就先從最近考到的 Certified Kubernetes Administrator (CKA) 證照開始吧！

<br>

## 介紹

![img](https://i.imgur.com/mP5nWWv.png)

[Certified Kubernetes Administrator (CKA)](https://www.cncf.io/certification/cka/) 是由 [Cloud Native Computing foundation (CNCF)](https://www.cncf.io/) 所發行的證書，目前的報名費用是 **$375** 美元，不定時會有一些折扣碼釋出，但今年的黑色星期五就很罕見的沒有佛心折扣碼，只有一些固定打 8 折或 85 折的折扣碼，但下面會提到我是在哪邊去買這個考試資格的，總共只花了 6500 台幣，算起來大概打了 62 折，能大幅降低荷包失血的程度 XD。

CKA 提供瞭解各個 Kubernetes 基礎元件組成，以及清楚 Kubernetes 架構和運作模式的「管理者/開發者」來進行測驗，考試時除了 Terminal 分頁以外，還可以再開一個[官方 Kubernetes Docs](https://kubernetes.io/docs/home/) 的分頁來查詢，考試時間 2 個小時，我遇到的題目總共有 17 題，體感上來說題目的順序並不是照著難度來排的，考試配分的比重我覺得和官方提供的[配分表](https://www.cncf.io/certification/cka/)蠻相似的：

![img](https://i.imgur.com/ifFZxvK.png)

本篇適合粗略瞭解**容器化概念 (Containerized)**的讀者，不管你之前有沒有接觸過 Kubernetes，相信這篇文章應該能帶給你一些收穫，會和大家分享一些考試的技巧，以及我如何從只會在工作環境使用一些 Kubernetes 簡單指令到拿到 CKA 證照的！

<br>

## 考前準備

#### Udmey 課程

在這裡推薦任何不管有無經驗想要考 CKA 的讀者，花個 380 塊台幣跟著 Udemy 上的課程 [Certified Kubernetes Administrator (CKA) with Practice Tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/) 來準備。如果你完全沒有經驗的話，那這篇能夠帶你去從 Kubernetes 基礎的設計從廣而深去學習；如果你是平常就有在使用 Kubernetes 來做日常開發的工程師，那這個課程能夠更穩固你的知識，甚至是在之後需求和架構上的設計都能因爲更全面地瞭解 Kubernetes 而有更多元彈性的解決方案！

這個課程的講師 **Mumshad Mannambeth** 除了 CKA 以外也有提供 CKAD 的課程，整個課程上完的感覺優點如下：

1. 價格便宜，Udemy 時常會有折扣價所以大約 10 美元左右就可以買到了
2. 上課節奏是很平穩的，講師口音不明顯，語速偶爾會稍快但也可以練習一些英聽
3. 對於一些前置的基礎概念包括 Docker, Networking, etc. 都會有小節來幫你補足先備知識
4. 學習完各個元件、知識後都有 **Practice Tests** 可以練習，最後還有三次的 Mock Exam 可以做
5. **提供免費的 Web 環境 (KodeKloud) 讓你無限次地去做 Practice Tests 來熟悉操作**


![img](https://i.imgur.com/JNFCnOZ.png)


最後一點是我認為最重要也是這個課程最優質的一點，因為有一個環境能夠讓你方便快速的實戰練習各個小節學到的知識和技巧是很重要的。CKA 在你大概懂 Kubernetes 的架構之後，考點就剩下你對 `Terminal`、`Kubernertes CLI` 的熟練度了，如果你能夠在測試環境去完成各個元件的操作 (`Pod`, `Service`, `Deployment` ...)，那在真正上考場的時候也就不需要擔心了，因為接下來會提供一份之前考試時使用的書籤資料夾，裡面已經有各種常見且分好類的文檔連結，這樣在看到類似的題目時可以直接用書籤進到文檔頁面，會省去很多時間。

<br>

#### 準備時間

我大概是在 8 月下旬的時候買上面的課程決定要考 CKA 的，每天用的時間就是平日中午吃飯從配 Youtube 換成配 Udemy，一天取決於小節長度，大約都可以看 2 ~ 4 個小節左右，假日則是看個 1 ~ 2 個小時的課程，再邊補齊平日看完但沒做完 Practice Tests 的地方，最後在 11 月中旬的時候就順利考到證照啦！

其實我準備的時間算是拉得很長的，主要是因為不想要學得太趕，把每個地方都學透徹了慢慢學才繼續走下一步，加上某些不熟悉的地方會練習比較多次，像是 **ETCD 備份**、**Cluster 升級** 之類的，平常如果你和我一樣接觸到的都是 Cloud Platform (`GKE`, `EKS`, `AKS`) 幫你管理 Kubernetes 平台的話，那這些東西對你來說會相較於使用 `Pod`、`Deployment`、`Service`、`Ingress` 來得陌生，而這些考題的比分佔比也是相較簡單的題目多蠻多的，加上這幾塊因為流程其實雖然繁瑣，但最根本的那幾個指令和步驟都是在文檔上面清清楚楚的寫出來了，所以我認為這些都是很好拿分的考點。

在 11 月初的時候因為已經安排考試時間了，所以我就是有意識地去不斷重作一些自己認為不熟的章節，包括 Mock Exam 三題也是刷了好幾遍，盡可能讓自己去熟悉實際操作，考試當下看到題目的時候才不會因為緊張而忘記指令。


## 實作

#### 1. 拿到專案 Service Account 憑證檔案

首先你要先有一個 GCP 的帳號和專案，再進入 [Service Account Create Page](https://console.cloud.google.com/apis/credentials/serviceaccountkey) 創一個 SA，並拿到他的 JSON 檔，並命名為 `sa.json`。

<img src="https://i.imgur.com/QHRsfle.png" style="width:450px">

#### 2. 建立基本環境
```bash
# 路徑結構
Docker-Python-Cloud-PubSub/
    -- Dockerfile-Pub # Build Pub Container 用的 Dockerfile
    -- Dockerfile-Sub # Build Sub Container 用的 Dockerfile
    -- docker-compose.yml
    -- sa.json
    -- pub.py
    -- sub.py
```

Clone 我的 Github Repo ([Docker-Python-Cloud-PubSub](https://github.com/JasperSui/Docker-Python-Cloud-PubSub)) 後，將你自己的 `sa.json` 檔案丟進去。

接著來組 Puber 和 Suber 的 Dockerfile，這兩個的 Dockerfile 幾乎是一模一樣，只差在最後 CMD run 的 python file 不同而已！

```dockerfile
# Dockerfile-Pub
FROM python:3.8-slim
WORKDIR /usr/src/app
RUN pip install google-cloud-pubsub==2.3.0
COPY . .

# 指定憑證的路徑
ENV GOOGLE_APPLICATION_CREDENTIALS=/usr/src/app/sa.json

# 用 --build-arg 帶入 GCP Project ID
ARG GCP_PROJECT_ID
ENV GCP_PROJECT_ID=${GCP_PROJECT_ID:-supple-student-304807}

CMD ["python", "./pub.py"]
```

```dockerfile
# Dockerfile-Sub
FROM python:3.8-slim
WORKDIR /usr/src/app
RUN pip install google-cloud-pubsub==2.3.0
COPY . .

# 指定憑證的路徑
ENV GOOGLE_APPLICATION_CREDENTIALS=/usr/src/app/sa.json

# 用 --build-arg 帶入 GCP Project ID
ARG GCP_PROJECT_ID
ENV GCP_PROJECT_ID=${GCP_PROJECT_ID:-supple-student-304807}

CMD ["python", "-i", "./sub.py"]
```

---

組好後就剩下主角 `pub.py` 和 `sub.py` 啦！

在 `pub.py` 我們只做一件事，就是每 5 秒就去推送一則訊息到 `channel` 這個 Topic，

```python
# pub.py

import os, time, json, random, datetime
from google.cloud import pubsub_v1
from google.api_core.exceptions import AlreadyExists

project_id = os.environ.get('GCP_PROJECT_ID') # Google Project Id

topic_id = "channel" # Topic Id

topic_path = f"projects/{project_id}/topics/{topic_id}"
publisher = pubsub_v1.PublisherClient()
msg_list = [{"type":"show_channel_info", "id": 22},
            {"type":"show_channel_info", "id": 23},
            {"type":"show_channel_info", "id": 52}]

# 新建一個 Topic
try:
    publisher.create_topic(name=topic_path)
except AlreadyExists:
    print(f"{datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
          f" [WARNING] Topic \"{topic_id}\" already exists.")

# 定時每 5 秒 Publish 一個隨機 message 到 channel 這個 Topic
while 1:
    message = json.dumps(random.choice(msg_list)).encode()
    publisher.publish(topic_path, message)
    print(f"{datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')} [Info] message sent, message: {message}")
    time.sleep(5)
```

而 `sub.py` 的目的也只有一個，就是收 Message 並印出 Channel info。
```python
# sub.py

import os, uuid, time, json, random, datetime
from google.cloud import pubsub_v1
from google.api_core.exceptions import AlreadyExists

CHANNEL_MAP = {
    22: "Cartoon",
    23: "YoYo TV",
    52: "News"
}

project_id = os.environ.get('GCP_PROJECT_ID') # Google Project Id

topic_id = "channel" # Topic Id

topic_path = f"projects/{project_id}/topics/{topic_id}"
container_name = os.environ.get('HOSTNAME')
sub = f"sub-{container_name}-{uuid.uuid4().hex}"
sub_path = f"projects/{project_id}/subscriptions/{sub}"
subscriber = pubsub_v1.SubscriberClient()

# 新建一個 Subscription

try:
    subscriber.create_subscription(name=sub_path, topic=topic_path)
except AlreadyExists:
    print(f"{datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
          f' [WARNING] Subscription already exists, sub_path: {sub_path}')

def callback(message):
    data = json.loads(message.data.decode())
    if type := data.get('type'):
        if type == 'show_channel_info':
            print(f"{datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
                  f" [INFO] Message received, the channel is {CHANNEL_MAP.get(data.get('id'))}")
    message.ack()

future = subscriber.subscribe(sub_path, callback)
```

---

還有最後的最後，就是如果你是自己要用你自己的 GCP 專案來 Run 的話，那務必要修改 `docker-compose.yml` 裡面的參數喔！

```yaml
version: '2'
services:

  # Suber 1
  suber-1:
    tty: true
    build:
      context: .
      dockerfile: Dockerfile-Sub

      # 如果你要用自己的 Google Cloud Project 的話，這邊 ID 要換
      args:
        GCP_PROJECT_ID: supple-student-304807

  # Suber 2
  suber-2:
    tty: true
    build:
      context: .
      dockerfile: Dockerfile-Sub
      args:
        GCP_PROJECT_ID: supple-student-304807

  # Puber
  puber:
    tty: true
    build:
      context: .
      dockerfile: Dockerfile-Pub

      args:
        GCP_PROJECT_ID: supple-student-304807
```

#### 3. 成果時間

等你確定好你的 `sa.json` 檔案和 `docker-compose.yml` 裡面的 `GCP_PROJECT_ID` 都換成你的了之後，就可以一行指令開始 Run 啦！

```sh
docker-compose up -d
```

跑起來的結果大概會像這樣子，左邊是 Puber，中間是 Suber 1，右邊是 Suber 2。
![img](https://i.imgur.com/xwz7gQX.gif)

## 總結

自己操作過一次之後，就會發現要實作起來一點都不難，我覺得難的點反而是要想一下什麼情境會需要使用到。這次剛好有機會可以在產品上直接實作一次，也踩了許多坑，所以強烈推薦大家自己去看一下 [google-cloud-pubsub](https://googleapis.dev/python/pubsub/latest/index.html) 的文檔，裡面對 `PublisherClient` 和 `SubscriberClient` 都有更詳細的介紹，也有更多 API 可以使用。

像這次就有搭配 `SubscriberClient` 的 `delete_subscription` 來使用，避免每次部署都新建一堆 Suber 導致 Topic 下面塞滿一堆不必要的 Suber。

## 參考

[Google Cloud Pub/Sub](https://cloud.google.com/pubsub?hl=zh-tw)<br>
[google-cloud-pubsub](https://googleapis.dev/python/pubsub/latest/index.html)<br>