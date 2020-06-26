---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (8) - 結語"
subtitle:   ""
date:       2020-06-26 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 正文

終於將這次的小專案在距離第一篇文章的 18 天後做完了，其實當時給自己一個月的時間來做算是有點久，但主要就是想讓自己壓力不要太大，但是還是必須要持續不斷學習和成長。

前陣子在公司有試圖想要揪人一起參與寫 Blog 的活動，互相督促並提出建議，並給了每個禮拜寫兩篇文章的要求，但很可惜沒有辦法能和公司同事們一起討論，所以後來就變成自己一樣限制自己每個禮拜至少要寫兩篇文，只能等之後有其他機會再去挖掘其他人一起來響應這個活動。

在還沒自己親手來將一個專案從傳統的 Django Project Dockerized，然後再部署上 GKE，最後結合 Jenkins 來做 CI/CD 之前，對這些東西我都是一知半解的，因為平常在碰的東西也就差不多是這些，但之前我只會看了同事用了什麼招，Google 一下之後記下來，我甚至連 Container 的概念都不是很清楚。

隨著自己慢慢摸，慢慢把腦海中模糊的地方一塊一塊變清晰，偶爾會遇到像是強力污漬般的東西阻礙我，但是在我嘗試了很多辦法花了很多時間，甚至最後是直接和同事討教，就通常能夠慢慢地將不懂的地方弄清楚，而不是只會輸入指令卻不知道自己在幹嘛，同時又再一次體會到環境的重要性，也很高興我能夠在這個環境繼續學習。

說到環境，最功不可沒的就是我的兩位大前輩同事，分別是 **Nick** 和 **Jing**。

Nick 是公司的 DevOps，曾經在旁邊看著他敲了敲鍵盤就在短短時間內複製了一個公司現有完整的環境來提供其他同事作為測試用，實力絕對是不容質疑的，會有想要將我的模仿蝦皮專案來用 K8s + Jenkins 來做的原因絕大部分都是 Nick 給我的靈感，如果沒有他的幫助和提示我是沒辦法在這麼短的時間內將專案完成的。

Jing 是我們的 Sr. Backend Engineer，從談吐到實際操作的舉手投足間，給我的感覺真的就只有「果然是 Senior」，在這次專案也幫助了我很多地方，包括 Jenkins 的 DinD，那邊真的是讓我少繞了一大圈，也會在我進行到一半時來聽聽我目前的進度以及想法，看看我是不是走在正確的路上。

真的要謝謝兩位大神不斷提點和建議，能夠從他們那邊學到一點精華就是非常難得的了，而且兩位都不藏私，盡所能地把自己會的東西都教給我，遇到貴人真的是非常幸運，也希望未來能夠和他們學習更多東西，結合自己的所學能夠回饋給他們那是再好不過的了！

也謝謝讀者的耐心，雖然文章語意並不是那麼通順，但也是期望自己未來能夠更隨心所欲地將心中所想的用淺顯易懂的方式呈現出來，寫出更好的文章！


### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx]({% post_url 2020-06-14-k8s-jenkins-django-project-5 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress]({% post_url 2020-06-16-k8s-jenkins-django-project-6 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (7) - Jenkins]({% post_url 2020-06-21-k8s-jenkins-django-project-7 %})
