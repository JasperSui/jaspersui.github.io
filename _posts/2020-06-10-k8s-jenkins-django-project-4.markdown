---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)"
subtitle:   ""
date:       2020-06-10 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 正文

上一篇已經把 `jaspershop-api.yaml` 裡面的兩個 Container (`jaspershop-nginx`、`jaspershop-api`)都介紹完了，今天要來把 `jaspershop-db.yaml` 裡面用到的 **kind** 都稍微提一下，這邊篇幅比較小，所以直接上 yaml 檔！

```yaml
apiVersion: v1
kind: PersistentVolumeClaim # PVC
metadata:
  name: mysql-pv-claim
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce # 三種模式的其中一種，GKE 原生只支援 ReadOnlyMany, ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaspershop-db
spec:
  selector:
    matchLabels:
      app: mysql 
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers: 
        - image: mysql
          name: mysql-con
          imagePullPolicy: Always
          env: # 環境變數
            - name: MYSQL_ROOT_PASSWORD  
              value: rootroot
            - name: MYSQL_DATABASE # 初始 Schema
              value: jaspershop
            - name: MYSQL_PASSWORD 
              value: rootroot
          args: ["--default-authentication-plugin=mysql_native_password"]
          ports:
            - containerPort: 3306 
              name: mysql 
          volumeMounts:
            - name: mysql-persistent-storage 
              mountPath: /var/lib/mysql # 綁定 PVC 內的資料到 /var/lib/mysql，達成永久化儲存 
      volumes:
        - name: mysql-persistent-storage # 依賴上面宣告的 PVC 所成的 Volume
          persistentVolumeClaim:
            claimName: mysql-pv-claim # 上面宣告的 PVC 的 name
```

用 `kubectl apply -f jaspershop-db.yaml` 將各自的元件都建好之後，接下來使用 `kubectl cp` 指令將原先的 Dump 檔案丟進 Container 裡去匯入資料，`jaspershop-api` 這個 Pod 就可以拿到資料庫的資料了！

而其中最值得一提的就是 `PersistentVolumeClaim`，先前有提到若是使用 `emptyDir` 的 Volume 來讓 Container 之間都可以讀寫某個路徑的資料夾，雖然很方便，但是缺點就是**當 Pod 生命週期結束時，屬於 emptyDir 類型的 Volume 資料都會被清空**，其實可以想成類似是電腦的記憶體，只要關機了裡面的資料就會消失。

但是資料庫的資料不能在每一次 Pod 重啟資料都被清空啊！於是這時候我們就需要利用 `PersistentVolume (PV)` 和 `PersistentVolumeClaim (PVC)` 而成的 Volume 來協助我們永久性的儲存資料，只要 PVC 被建立了，就會去接上適合的 PV，取得 PV 的資料並分享給有使用 `volumeMounts` 來接上 PVC 的 Container，也就是說------**PVC 和 PV 是相對應的**。

![img](https://wiki.shileizcc.com/confluence/download/attachments/29982850/image2017-12-19_9-38-42.png?version=1&modificationDate=1513648636000&api=v2)

和 `emptyDir` 的 Volume 不一樣，只要不去主動刪除 PV 的話，每一次創建的 PVC 都可以去接上相對應的 PV 並取得 PV 中的資料，來達成永久化儲存資料的目的。

看到這邊可能會想說那我的 `PersistentVolume (PV)` 去哪裡了，並沒有看到 PV 在我的 yaml 檔裡啊！**沒有 PV 只有 PVC 的話確實是沒辦法正常使用的沒錯**，但如果需要細講的話，會花上太多時間，我認為只需要用到的時候再去看一下就可以了，簡單來說，產生 PV 的方法分為 **static** 和 **dynamic**：

* `static`: 先建立幾個規格的 PV 預備給未來的 PVC 連接
* `dynamic`: 如果現有 PV 沒有符合 PVC 需求的話，**就會去自動產生相對應條件的 PV**

有興趣的話可以詳細看一下 [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)，因為我們用的是 GKE，所以可以用 **dynamic** 這種模式來使用 PVC，因此不需要先去建立 PV。

### 小結

在 [Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %}) 和 [Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %}) 裡我們把 `docker-compose.yaml` 中最基本的 **nginx**、**api**、**db** 的三個 Container 都透過 yaml 檔的形式來分別處理成 K8s 的 **Service**、**Deployment** 元件並部署成功了！但是接下來要做的事情還可多了，為了避免自己也忘記，先寫下剩餘的步驟讓大家有個心理準備：

1. 解決 nginx 的 `/static/` 儲存問題
2. 用 Ingress 來處理外部來的 request
3. 搭配 Github + Jenkins 來完成專案的 CI/CD

感謝各位的閱讀，若是語意上有不通順或是講解錯誤請不吝指教！

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx]({% post_url 2020-06-14-k8s-jenkins-django-project-5 %})

## 參考

[Kubernetes PV and PVC Life CycleDjango Channels 2](https://wiki.shileizcc.com/confluence/display/KUB/Kubernetes+PV+and+PVC+Life+Cycle)