---
layout:     post
title:      "用 Python + Docker 實作簡易 Cloud Pub/Sub Service"
subtitle:   ""
date:       2021-02-14 02:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
    - Docker
    - Google Cloud
    - Pub/Sub
    - Messaging Service
---
## 前言

前陣子在解決某次情境的問題時，因為要考慮將 Compile time 的參數拉到 Run time，和同事討論後就決定繼續使用和團隊相依性很高的 GCP (Google Cloud Platform) 的產品，也就是本文主角 -- Cloud Pub/Sub 來實現。

加上 Google 什麼都幫你包好，流量不大的話通通免費，Library 也很簡單就能使用，以及文件寫得不錯，除了有些細節的地方還是得去看一下 SDK 的 Source Code 是小缺點以外，算是雞蛋裡挑骨頭了！

專案部分可以直接參考我的 Github Repo ([Docker-Python-Cloud-PubSub](https://github.com/JasperSui/Docker-Python-Cloud-PubSub))。

## 介紹

Cloud Pub/Sub 是一個異步的 Messaging Service，將 Publisher 和 Subscriber 解耦，讓兩者不再相依。
除此之外他們還提供訊息保存 (但是要收費)，會用到 Google 服務的不外乎就是看上他們的穩定、快速，還有低流量的用戶友善 (還有 Cloud Function ...之類的)。

之前會用到這個服務是因為要解決特殊情境，以我的例子來說，Django 的 A 套件會在 Run Time 時去拿一次 B 變數的值用作未來的參數，原先我們是先將這個值放在環境變數中，透過 Yaml 檔在部署的時候能夠帶這個環境變數到 Container 裡提供給 A 套件使用。

這樣問題就來了，當這個參數需要修改的時候，我們都只能修改 Yaml 檔裡的值然後重新部署，搞了幾次下來發現實在是太麻煩了，於是就將那個值改從 Database 去拿，再引入 Cloud Pub/Sub。讓每次服務啟動時都自帶 Publisher 和 Subscriber，只要收到 Database trigger 的 signal 之後推訊息到其他各個需要的 Container，再也不需要重新進版控、等部署，只需要改個 DB 資料就能完成動態的參數設定！

但一定不是每個人的使用情境都相同，所以今天就著重在 Cloud Pub/Sub 服務本身而不是介紹上述情境。

### 組成

Cloud Pub/Sub 的組成很單純，其實就和他的名字一樣，由 Pub 和 Sub 來分為四個元素：

1. **Topic**

    就是一個 Topic，和 Subscription 是一對一或一對多的關係。
<br><br>

2. **Publisher (簡稱 Puber)**

    推送訊息的角色，一個 Publisher 會和一個 Topic 綁定，專門推訊息給該 Topic。
<br><br>

3. **Subscription**

    一個 Subscription 只能隸屬於一個 Topic 之下，用來作為接收該 Topic 訊息的角色，收到資料之後，就會指定一個 Suber 來回覆訊息。
<br><br>

4. **Subscriber (簡稱 Suber)**

    一個 Suber 可以綁定一個以上的 Subscription，用來回訊息給 Subscription。
<br><br>

簡而言之，Puber 和 Topic 是**一對一**的關係，Suber 和 Subscription 則是**一對一或一對多**的關係。

看起來雖然複雜，但用 Google 的服務一定 i配 [Document](https://cloud.google.com/pubsub/docs/overview?hl=zh-tw)，搭配流程圖來看就能快速上手了！

![img](https://cloud.google.com/pubsub/images/pub_sub_flow.svg?hl=zh-tw)

Puber 推一個訊息到 Topic 之後，Topic 會將這個訊息廣播 (Broadcast) 到他底下的所有 Subscriptions，再由 Suber 去回答 (ACK/NACK)。以上就是一個基本的訊息推送流程，無論要做成 `One-to-Many (Fan in)`、`Many-to-One (Fan Out)`，或是 `Many-to-Many`，都可以很彈性地透過這幾個元素組合出實際需要的模式。

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