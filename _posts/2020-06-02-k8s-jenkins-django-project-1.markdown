---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言"
subtitle:   ""
date:       2020-06-02 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
---
## 正文

好一陣子沒更新 Blog 了，可能是因為前一陣子工作忙碌導致才剛開始沒多久就有點怠惰，今天讀了各式各樣的文章，大到新技術的實作相關，小到技術人該有的心態，又重拾了那種戰戰兢兢的感覺，說到這個，沒多久前才對這張線圖很有感。

![img](https://miro.medium.com/max/1400/1*WjNKcO_mklTPrBTuWyZB3g.jpeg)

達克曲線，相信大家或多或少有看過。前陣子以為自己正在從谷底慢慢往上爬了，但隨著進入新的工作環境，該學習的技術越來越多，碰得越多，越清楚自己的不足，也才意識到自己其實在往上爬的旅途還有一大段路要走。

最近向同事 Nick 詢問了一下 k8s 和 Jenkins 的一些問題，他給了我一些建議，也很謝謝他帶我入門 k8s。雖然工作上會碰到，但輸入指令並不代表真正的「會用」，剛好先前有做了一個用 [Django + Nginx + uWSGI 實作的蝦皮購物網站](https://github.com/JasperSui/Django-Nginx-uWSGI-High-Performance-JasperShop) ，趁著這次機會來把以前的 Side Project 翻新一下，順便紀錄一下過程。

![img](https://titangene.github.io/images/cover/gcp.jpg)

預計全程會在 [Google Cloud Platform (GCP)](https://console.cloud.google.com/?hl=zh-TW) 搭配 [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) 來完成此次專案，給自己一個月的時間來盡力做。

到時候系列文連結將會同步在下方，請大家拭目以待。

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})

## 參考

[The remarkable reason why so many presentations are boring.](http://refusetobeboring.com/remarkable-reason-many-presentations-boring/)