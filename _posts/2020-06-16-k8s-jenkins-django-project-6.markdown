---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress"
subtitle:   ""
date:       2020-06-16 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 正文

現在已經把所有會用到的 **Service**、**Deployment** 都部署完成，但是總不能每次都只能用 Port-Forwarding 的方式來連到網站，所以我們需要 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 來當作我們的 LoadBalancer，可以用來依照我們的規則轉發請求給對應的 Service，流程圖如下：

![img](https://miro.medium.com/max/958/1*K27cGIItdm_OBLLB8i0gAA.png)

因為目前只有一個主要的 Service，這次也只是拿來讓外部使用者可以透過 GCP 派發給 Ingress 的外部 IP 來存取我們的網站，所以可能看不出來 Ingress 的優點，有興趣的讀者可以去看一下其他 Ingress 可以實作的功能，這邊就不多提！

這次部署好了兩個 Ingress，分別是 `ingress` 和 `jenkins-ingress`。

### <ins>ingress<ins>
```yaml
# k8s-yaml/basic-ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  namespace: staging
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: jaspershop-public # 將 / 下的所有路徑指給 jaspershop-public 這個 NodePort
          servicePort: jaspershop-port # jaspershop-public 底下的 ports
```

```yaml
# k8s-yaml/jaspershop-api.yaml

kind: Service
apiVersion: v1
metadata:
  namespace: staging
  name: jaspershop-public
  labels:
    app: jaspershop
spec:
  type: NodePort
  ports:
  - name: jaspershop-port
    protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: jaspershop
```

分別將兩個 yaml 檔設置好之後就可以 apply 上 Cluster 了，接著需要等待幾分鐘，等到 Ingress 設定都配置好，接著可以輸入 `kubectl get ing ingress` 來取得 `ingress` 的資訊。

![img](https://i.imgur.com/AUoXkHd.png)

**34.120.102.166** 就是這個 Ingress 被派發到的外部 IP，接著就可以直接連上去看看有沒有被正確轉發 Request 了。

這邊要特別注意的是，因為 Ingress 的轉發條件，會需要 Ingress 下指定的 Service 所有 [Health Check](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress#health_checks) 都通過測試，確認狀態是 `HEALTHY` 之後才會進行轉發。

照上面的 `basic-ingress.yaml` 檔的設定，總共會產生兩個後端服務和兩個 Health Check，而這兩個 Health Check 需要做的是打通指定的路徑，通了之後才算是 `HEALTHY`，一個是檢查 `/healthz`，另外一個則是檢查 `/`。

在部署這個 Ingress 的時候，檢查 `/` 的那個 Health Check 的狀態不是 `UNHEALTHY` 就是 `UNKNOWN`，結果也就理所當然的沒辦法正常轉發 Request，檢查了好多次 Ingress 的設定有沒有問題，不斷地重新部署，都還是一樣，直接連 Ingress 的外部 IP 也只出現 Server Error，如下圖。

![img](https://i.stack.imgur.com/HVlD1.png)

在試到不知道還能怎麼改的情況下，剛好跑去看了 `k8s_settings/local.py` 的設定檔，改了 `ALLOW_HOST` 的值：

![img](https://i.imgur.com/nkRbcCc.png)

之後重新 Push `japsershop_api` 的 Image 到 Docker Hub，刪除所有 `jaspershop` 的 Pod 並讓 Deployment 自動重新建立起來後，檢查 `/` 的那個 Health Check 狀態就正常了，也就可以轉發了。

### <ins>jenkins-ingress<ins>

[Jenkins](https://www.jenkins.io/) 相關的 **Service**、**Deployment** 建立步驟會在下一篇文章詳述，這邊就只帶到 Ingress 的部分。

在 Jenkins 中，會需要使用 Jenkins 的 UI 來協助設定一些流水線或是專案，所以我們同樣需要一個 Ingress 來轉發對應的 Port (在這裡是 **8080**) 來指向 Jenkins UI。

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: staging
  name: jenkins-public
  labels:
    app.kubernetes.io/instance: my-release
spec:
  type: NodePort
  ports:
  - name: jenkins-port
    protocol: TCP
    port: 80
    targetPort: 8080
  selector:
    app.kubernetes.io/instance: my-release
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: staging
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: jenkins-public
          servicePort: jenkins-port 
```

大致上都和 `ingress` 的設定差不多，只差在了 `targetPort` 這邊要注意一下，必須對應到 Jenkins Service 指定服務的 Port，一樣 apply 上 Cluster 後，就可以透過 `kubectl get ing jenkins-ingress` 來取得 `ingress` 的資訊。

![img](https://i.imgur.com/ek0k64u.png)

**34.120.139.98** 就是給 Jenkins UI 的 IP 位址，可以直接在瀏覽器輸入，就可以看到 Jenkins 的登入畫面了。

![img](https://i.imgur.com/lsDV8Be.png)

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx]({% post_url 2020-06-14-k8s-jenkins-django-project-5 %})

## 參考

[Kubernetes — Ingress Setup with Nginx](https://medium.com/@rajan.ramesh/kubernetes-ingress-setup-with-nginx-8373f8492c6a)<br>
[GKE Ingress for HTTP(S) load balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress#health_checks)<br>
[Kubernetes Ingress (GCE) keeps returning 502 error](https://stackoverflow.com/questions/42967763/kubernetes-ingress-gce-keeps-returning-502-error)