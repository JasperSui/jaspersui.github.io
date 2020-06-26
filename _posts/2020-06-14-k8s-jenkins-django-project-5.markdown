---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx"
subtitle:   ""
date:       2020-06-14 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 正文

這篇單獨拿出來講的原因，是因為在前面都把所有 Deployment 執行起來，用 `kubectl port-forward` 直接連進 Pod  裡面看一下專案有沒有正常執行後，發現所有的 Static 檔案都沒有被正確映像到指定的路徑，但原本就有將 `static-volume` 用 `emptyDir` 的方式來分享靜態檔，而 `emptyDir` 的特性如 [K8s Document](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) 所說：

> An emptyDir volume is first created when a Pod is assigned to a Node, and exists as long as that Pod is running on that node. As the name says, it is initially empty.

因此沒辦法在還沒產生這個 Volume 也就是 Pod 生成並指派給 Node 前就先將資料寫入，於是我就改寫了一下 `jaspershop-api.yaml` 裡面的 `static-volume`，希望能和 `jaspershop-db` 裡面一樣，是用 PV 和 PVC 來儲存 靜態檔，方便 nginx 都能直接讀取，但後來測試了一陣子，發現就算用 PV 和 PVC 的方式來儲存，還是沒辦法將正確被解密的檔案或是圖片指給 nginx，就只能放棄這個方法。

後來，決定還是用 `emptyDir` 的 Volume 來解決這次的問題，前面有提到沒辦法在 Pod 生成前就將檔案寫入，那如果用一般 `python manage.py collectstatic` 的思維來做，和一般做專案的時候一樣，如果有需要更新 Static 檔案的時候再去跑這個指令，產生新的靜態檔，那問題應該就可以解決了！

於是我將在這次的 [Commit](https://github.com/JasperSui/k8s-jenkins-django-jasper-shop/commit/ca16b92eaf04fe2d391ff0bd0871cd4c01eca230) 中將 Container 的名字順手改名為比較好 Debug 的長度 (輸入指令的時候不用打那麼長 XD)。

原先的 `command` 就是 `["/bin/sh","-c"]`，意思是接下來的指令以腳本的模式執行，之後再將 `api` Container 底下的 `args` 變動一下，原先就有做以下兩件事：

```shell
1. uwsgi --ini uwsgi.ini
2. python manage.py collectstatic
```

接下來為了要在 Pod 產生後才去對生成靜態檔，就先執行 `python manage.py collectstatic` 後，取得生成出的靜態檔，再將檔案複製到 `static-volume` 裡，完成靜態檔的寫入，這樣就可以達到檔案在 Volume 被初始化完後才寫入的需求：

```shell
1. mkdir volume/static/ -p # 新建一個路徑為 volume/static/ 的資料夾
2. python manage.py collectstatic --noinput
3. cp -R ./static/ ./volume/ # 將 /static/ 資料夾複製到 /volume/ 下
4. uwsgi --ini uwsgi.ini 
```

這邊要注意的是，**uwsgi** 的指令開始做了之後就會直接進到 **Log** 模式，所以必須要在做完 `/static/` 資料夾的複製以後才執行 `uwsgi --ini uwsgi.ini`，否則在 uwsgi 後的指令都不會有作用。

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress]({% post_url 2020-06-16-k8s-jenkins-django-project-6 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (7) - Jenkins]({% post_url 2020-06-21-k8s-jenkins-django-project-7 %})
<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (8) - 結語]({% post_url 2020-06-26-k8s-jenkins-django-project-8 %})

## 參考

[Kubernetes PV and PVC Life CycleDjango Channels 2](https://wiki.shileizcc.com/confluence/display/KUB/Kubernetes+PV+and+PVC+Life+Cycle)