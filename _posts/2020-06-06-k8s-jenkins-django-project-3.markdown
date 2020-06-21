---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)"
subtitle:   ""
date:       2020-06-06 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 前言

久違的第三篇終於來了，其實最近都有一直花心思在這次的專案上，但也想要呼籲一下大家，平時還是得多注意自己的心理健康，不要太過逼迫自己而不自知，學會偶爾喘口氣對於路要走得長遠是一件非常重要的事！

## 正文

大概前情提要一下，在上一篇我們簡單地 Dockerize 了舊有的專案，依照原先的專案的組成來分成以下 4 個 Container (其實還有 celery beat + worker 沒做但是有點懶 XD) 

1. nginx
2. uWSGI
3. RabbitMQ
4. MySQL

直到四個 Container 都可以由 `docker-compose up` 一次直接執行起來後，就可以開始往下一步走了，也就是今天的主題，用 **[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)** 來作為我們管理叢集 (Cluster) 的助手。

首先必須先準備一個已經有餘額能夠使用 GCP 的帳號，這裡有兩個方式，一個是透過**建立一個新的 Google 帳號加上綁定帳單資訊（信用卡）**，就會有 $300 的試用額度可以使用，而另外一個方式就是比較現代手段一點而且適合懶人——**課金**。

有帳號可以使用後，接下來就可以直接進到 GCP 的任一專案內開始操作，直接進到 **Google Kubernetes Engine** 的標籤後就可以開始建立叢集

![img](https://i.imgur.com/YZCbPtM.png)

基本上只要把 Cluster 的名字改成自己喜歡的，甚至是什麼都不改，直接按下建立，就完成了。

接下來要安裝 gcloud 的 sdk，可以參考[這篇](https://cloud.google.com/sdk/docs/downloads-interactive)依照自己的系統來輸入對應的指令安裝，值得注意的是，安裝完後**務必輸入 `gcloud init` 並依照說明來完成初始化 gcloud 的動作**，其中會需要輸入帳號 (上面使用的 Google 帳號)，這個要對應到否則會無法對 GCP 的專案進行任何操作。

接著可以輸入 `gcloud components install kubectl` 來安裝 `kubectl` 套件，接下來會一直透過它來完成許多事。

安裝完後，回到 GKE 的頁面，找到自己剛剛新建的 Cluster，最右邊有一個連結的按鈕

![img](https://i.imgur.com/1q6lopq.png)

點進去後會有 GKE 幫你產生的一行指令，是讓你的 `kubectl` 可以直接連到這個 Cluster，複製貼上到你的 Terminal 就可以準備開始寫我們部署的 `.yaml` 檔了

（註：後來才聽到同事們對於常常在碰 k8s 的工程師為 `yaml engineer`，起初還沒辦法完全體會，過了一陣子突然想起之後才苦笑一下 XD）

![img](https://i.imgur.com/yT3gbLg.png)

接下來就搭配這次的 Commit 來服用，每個步驟會將有連結對應到被修改的檔案並大略說明一下，至於 K8s 的各個組成元件礙於篇幅的關係請自行搭配 [K8s Documentation](https://kubernetes.io/docs/home/) 來比對：

第一步要做的事我選擇將上面的 `nginx` 和 `uWSGI` 這兩個 Container 先合併成一個 Deployment，以 [jaspershop-api.yaml](https://github.com/JasperSui/k8s-jenkins-django-jasper-shop/blob/72aab66d54f7f2d83054b2b9c10d545f3c9d951b/k8s-yaml/jaspershop-api.yaml) 來規範這一個 Deployment，下面以註解的方式來介紹重點部分，

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: staging
  name: jaspershop-api
  labels:
    app: jaspershop
spec:
  replicas: 3 # Pod 要維持的數量就是由這裡來設定的
  selector:
    matchLabels:
      app: jaspershop-api
  template:
    metadata:
      annotations:
        version: latest
      labels:
        app: jaspershop-api
    spec:
      containers:

      - name: jaspershop-nginx
        image: nginx:stable
        ports:
          - containerPort: 80
        volumeMounts:
          - name: nginx-conf-volume # 這個非常重要，會對應到下方 volume，用來初始這個 container 的 nginx.conf 設定
            mountPath: /etc/nginx/conf.d
          - name: sock-volume # 用來取得 jaspershop-api(uWSGI) 所產生的 .sock 檔，為 nginx 和 uWSGI 的溝通手段之一
            mountPath: /usr/src/app/ # 指定 jaspershop-nginx 的 /usr/src/app/ 作為 sock-volume 的路徑
          - name: static-files # 用來取得 jaspershop-api 指令產生的 /static/ 資料夾
            mountPath: /usr/src/app/static/ # 指定 jaspershop-nginx 的 /usr/src/app/static/ 作為 static-files 的路徑

      - name: jaspershop-api
        image: suiyang03/k8s-jenkins-django-jasper-shop_api:latest # 我放在 Docker Hub 裡自己打包好的 Django Image
        imagePullPolicy: Always
        env:
          - name: DJANGO_SETTINGS_MODULE # 初始化 Django Settings 的路徑
            value: JasperShop.k8s_settings.local
        command: ["/bin/sh","-c"]
        args: ["uwsgi --ini uwsgi.ini && python manage.py collectstatic"] # 執行 uWSGI 的同時產生 /static/ 資料夾
        volumeMounts:
          - name: static-files # 對應到 nginx 需要的 /static/ 的 volume
            mountPath: /usr/src/app/static/ # 指定 jaspershop-api 的 /usr/src/app/static/ 作為 static-files  的路徑
          - name: sock-volume # 對應到 nginx 需要的 .sock 檔案的 volume
            mountPath: /usr/src/app/socket/ # 指定 jaspershop-api 的 /usr/src/app/socket/ 作為 sock-volume 的路徑
      
      volumes:
        - name: nginx-conf-volume # 用來分享自訂的 nginx config 的 volume
          configMap:
            name: nginx-conf
            items:
            - key: jaspershop-nginx.conf
              path: jaspershop-nginx.conf
        - name: sock-volume # 用來作為 container 之間分享 .sock 檔案的 volume
          emptyDir: {}
        - name: static-files # 用來作為 container 之間分享 /static/ 檔案的 volume
          emptyDir: {}
```

這邊卡比較久的坑就是 `volumes` 的部分，因為在我的 `uwsgi.ini` 和 `jaspershop-nginx.conf` 中可以看到我是使用 `.sock` 檔案來作為讓 `nginx`, `uWSGI` 彼此可以互相通訊的管道

```conf
# /k8s-yaml/env/jaspershop-nginx.conf

# Line 4

upstream uwsgi {
    # server api:8001; # use TCP

    server unix://usr/src/app/app.sock; # for a file socket

}

(略)
```

```ini
# /JasperShop/uwsgi.ini

# Line 16

socket          = /usr/src/app/socket/app.sock

```

可以看到 **uWSGI** 會產 `app.sock` 到 `/usr/src/app/socket/` 路徑下，而 **nginx** 又想要到 `/usr/src/app/` 拿到 `app.sock`。

那現在怎麼辦呢？別忘了，**K8s 的最小單位是 Pod**，我們實際上是沒辦法透過正規方式在 runtime 的時候去操作 Container 裡面的資料的，這時候我們就需要 `emptyDir` 的 `volume` 來幫我們建立一個 Container 之間都可以讀寫的地方，這樣不論是什麼資料，都能夠透過指派 `mountPath` 來將指定路徑的資料映射到這個 `volume`，之後就可以輕鬆達成我們的需求了。

以 `sock-volume` 為例，在 `jaspershop-nginx` 中因為我們需要取得的 `app.sock` 的路徑如下：

> /usr/src/app/app.sock

而在 `jaspershop-api` 中，我們設定 **uWSGI** 產生的 `app.sock` 檔的路徑如下：

> /usr/src/app/socket/app.sock

所以只要將 `jaspershop-nginx: /usr/src/app/` 和 `japsershop-api: /usr/src/app/socket/` 都指派給 `sock-volume`，那這樣兩邊都可以在各自預設的路徑中取得到 `app.sock` 檔案了。

同理，`static-files` 也是將 `jaspershop-api` 產生的 `/static/` 資料夾指派到 `jaspershop-nginx` 中指定讀取 `/static/` 的路徑即可。

接著我們只要執行 `kubectl apply -f jaspershop-api.yaml` 及 `kubectl get pods`

就能夠看到有三個 `jaspershop-api` 的 pods 正在運行中囉！

![img](https://i.imgur.com/btBFEMn.png)

下一篇會再將剩下的 MySQL 以及 RabbitMQ 都一起部署上去，順便講講部署 MySQL 的時候可能會遇到的坑！

謝謝你的閱讀，篇幅比較長可能會有些語意不通順的地方，歡迎隨時下方留言和我討論！

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx]({% post_url 2020-06-14-k8s-jenkins-django-project-5 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress]({% post_url 2020-06-16-k8s-jenkins-django-project-6 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (7) - Jenkins]({% post_url 2020-06-21-k8s-jenkins-django-project-7 %})

