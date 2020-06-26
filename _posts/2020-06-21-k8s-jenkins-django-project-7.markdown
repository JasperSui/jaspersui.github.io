---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (7) - Jenkins"
subtitle:   ""
date:       2020-06-21 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
    - GKE
---
## 前言

為了生出這篇文，卡了兩三天才做出來，遇到了好多問題但也都一一解決了，如果沒有同事 Nick 和 Jing 兩位大神每次都在我卡關的時候給我一些想法和建議，我可能要花兩倍甚至更久的時間來踩雷，除了感謝沒有其他詞可以用來表達我的心情了！

## 正文

在上一次，我們將 Ingress 部署起來，已經可以看到我們的網站在 Ingress 被派發的 IP 位址上被瀏覽了，其實只要這樣子就是一個高可用性的網站，一路做到這裡，真的要給自己一個大大的鼓勵。

但好還要更好，假設目前我們有一個新的版本要部署到網站上，我們就會需要以下步驟：

```
1. Push 改動到 Git Repository
2. Build Docker image
3. Push Docker image
4. Apply all yaml in k8s-yaml (By running `k8s-yaml/scripts/update_k8s.sh`)
5. Rollout Restart 相關的 Deployment
```

這樣看起來是不是步驟很繁瑣？而且每個步驟都還要下好幾個指令，如果能做到 Git Repository 每次被更新的時候就自動去執行接下來的步驟，那就會省去我們輸入指令的麻煩，除此之外，如果有個介面能夠讓我們管理每一次的部署，控管每一次的部署狀況，就可以讓部署的可靠性大幅提升。

所以，我們需要 [Jenkins](https://www.jenkins.io/) 來做我們自動化建置、部署的工具！

![img](https://www.jenkins.io/images/gsoc/jenkins-gsoc-transparent.png)

這次選用同事推薦的方式來安裝 Jenkins 到我們的 Cluster 上－`Helm Chart`，

需要先安裝 Helm，如果已經有安裝 Homebrew 的話可以用 `brew install helm` 來進行安裝。

接下來照著 [Helm Charts](https://github.com/helm/charts) 的安裝步驟來走就可以了，

#### 1. 把 `Helm Stable Charts` 加到本地的 Helm 裡
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

#### 2. 安裝 `stable/jenkins` chart
```
helm install my-release stable/jenkins
```

安裝完之後，就會跳出以下提示：

![img](https://i.imgur.com/9dAaDrX.png)

之後可以先執行 1. 的 Command，之後會看到一串英數組成的密碼，這個就是等等 Jenkins Admin 登入的密碼
```
printf $(kubectl get secret --namespace staging my-release-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

再搭配 [[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress]({% post_url 2020-06-16-k8s-jenkins-django-project-6 %}) 服用，將 Jenkins 的 Ingress 也架上去，就可以直接連上 Jenkins 了！

![img](https://i.imgur.com/OOZAwY3.png)

輸入帳號密碼後，進入到 Jenkins Admin 的主頁，接下來就直接進入到正題，我們要做的事情只有下列三項：

1. Github Repository Webhook 設定
2. Build/Push Docker image Job
3. Rollout Restart Job

接下來就一項一項慢慢完成吧！

---
### Github Repository Webhook 設定

進到你的 Github Repository 頁面，**Settings** → **Webhooks** → **Add Webhooks**

![img](https://i.imgur.com/0v947NX.png)

![img](https://i.imgur.com/54Cq7WF.png)

![img](https://i.imgur.com/3yGplz0.png)

這邊需要注意的只有 **Payload URL**，只需要輸入能連線到 Jenkins 的網址再加上 `/github-webhook/` 即可。

![img](https://i.imgur.com/kwjFZTx.png)

所以我這邊要帶的是 `https://34.120.139.98/github-webhook/` 就可以了！

**※ 這邊只帶到 Github 部分的 Webhook 設定，其他的設定大同小異，Jenkins 那邊有 Plugin 支援的應該都沒甚麼問題！**

<br>

### Build/Push Docker image Job

接著回到 Jenkins Admin 這邊，要用 Docker 來操作的地方，例如：Build Image、Push Image，就選擇用 Pipeline 來作。

先選 New Item，輸入 Item 名字後再選擇 Pipeline。

![img](https://i.imgur.com/l2IToZP.png)

![img](https://i.imgur.com/qCkHx3Q.png)

接著在 `Build Triggers` 區塊中勾選 `GitHub hook trigger for GITScm polling` 選項。

![img](https://i.imgur.com/NmV6Muj.png)

勾選完之後，只要 Github Repository 被 Push 後，就會自動觸發 Webhook 去 Post Jenkins，進而觸發這個 Job 的開始。

這塊就剩下實際的 Pipeline 腳本需要撰寫了，就直接貼上來邊看邊解釋吧！

```
pipeline {
    environment {
        registry = "suiyang03/k8s-jenkins-django-jasper-shop_api:latest" // Docker Image 的名字
        registryCredential = 'docker-hub-credential' // Docker Hub 的 Credential，下方會介紹
    }

    agent {
        // Jenkins Agent 的設定，這邊使用 Docker Image 來作我們的 Agent
        kubernetes {
            label "jenkins-docker"
            defaultContainer "docker"
                yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-release-jenkins
  containers:
    - name: docker
      image: docker:stable
      tty: true
      securityContext:
          runAsUser: 0
          privileged: true
      volumeMounts: 
        - mountPath: /var/run/docker.sock
          name: docker-sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
        }
    }
    stages{
        stage('GitHub Source'){
            steps{
                // Git Plugin 在 pipeline 中的用法
                git branch: 'master', url: 'https://github.com/JasperSui/k8s-jenkins-django-jasper-shop'
            }
        }

        stage('Build Image') {
            steps {
                // Docker Plugin 在 pipeline build image 的寫法
                script{
                    dockerImage = docker.build(registry, "JasperShop/.")
                }
            }
        }

        stage('Push Image') {
            steps {
                // Docker Plugin 在 pipeline push image 的寫法
                script{
                    docker.withRegistry('', registryCredential){
                        dockerImage.push()
                    }
                }
            }
        }
    }
}
```

比較特別的一個地方，就是這邊用了 **DinD (Docker in Docker)** 的方式來作這次的 Job，理由是我們要透過 Jenkins 來替我們完成 Build 和 Push Docker Image 的任務，所以 Jenkins 的機器一定要有 Docker 才能執行相關的指令，但是因為我們的 Jenkins 牽扯到 K8s 和 Docker，背後分派 Job 的邏輯又稍微複雜了一些，簡單來說可以用這張簡單的架構圖一言以蔽之：

![img](https://www.kubernetes.org.cn/img/2017/03/20170320212443.jpg)

剛部署好 Jenkins 時我們只會有 Master 的 Jenkins，只有在收到任務的時候才會由 Jenkins Master 去根據自身配置來產生 Slave Pod，並分派到節點中，所以想要在這種情境下用到 Docker，依照我試了各種方法的經驗下，還是用 Pipeline + DinD 的方式來作會最快。

接下來就分成三個 **Stage**：

- Stage 1: 取得並初始化我們的 Github Repository
- Stage 2: 透過 Clone 下來的 Github Repository 裡面的 Dockerfile 來 Build 出最新的 Docker Image
- Stage 3: 將 Build 出來的 Docker Image Push 到 Docker Hub 上

那聰明的讀者們應該會發現，`registryCredential = 'docker-hub-credential'` 這個是從哪裡來的，

會需要這個的原因是因為我們在 Push Image 到 Docker Hub 上的時候，一定會需要 Crendential 才能夠 Push，在本機的時候我們一定也是要先 `Docker login` 並輸入自己的帳號密碼後才能夠對自己的 Repository 進行一些操作。

而在 Jenkins Pipeline 中我們使用了 `Docker Plugin`，把這些繁瑣的事情都替我們包裝好了，我們只需要帶上正確的 Credentail，認證的事情就能夠替我們解決。

所以現在先到 [Docker Hub](https://hub.docker.com/) 去處理我們的憑證吧！

登入後，選擇 **Account Settings** → **Security** → **New Access Token**

![img](https://i.imgur.com/ED3DQYP.png)

接著替你的 Access Token 隨便取一個你認得出來的名字，並將 Token 記好！

![img](https://i.imgur.com/wLfMvac.png)

回到 Jenkins Admin，新建一個 Credential，並將資料填入即可。

![img](https://i.imgur.com/GjklTVe.png)

Username 就是在 Docker Hub 的帳號，Password 則是剛剛新增的 Access Token，ID 就是在 Pipeline 中的 `registryCredential = 'docker-hub-credential'`，新增完之後就能讓 Jenkins 在執行 Pipeline 時能帶入儲存好的 Credential。

### Rollout Restart Job

最後一步驟了！

一樣是選 New Item，輸入 Item 名字，但這次是選擇 Freestyle project，之後在 **Source Code Management** 的區塊需要設定 Git，和 Pipeline 那邊的 Stage 1 做的事情一樣，先將 Repository Clone 下來，原因是之後我們才能同步到最新的 `k8s-yaml/` 資料夾中的設定檔。

![img](https://i.imgur.com/xDorxzG.png)

還得設定 **Build Triggers** 要接在上一步驟的 Job 執行成功後，才能更新 K8s 的 yaml 檔。

![img](https://i.imgur.com/PyVa4vk.png)

最後就是設定這個 Job 的建置步驟：

![img](https://i.imgur.com/Uk1Soc6.png)

##### 第一個建置步驟

主要目的就是在 Agent 中安裝 `kubectl`，讓之後可以使用 kubectl 來和 K8s API 溝通。

```sh
apt-get update
apt-get install curl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

##### 第二個建置步驟

將所有的 yaml 檔案都透過一個 Shell Script 更新一遍。

```sh
sh k8s-yaml/scripts/update_k8s.sh
```

```sh

# k8s-yaml/scripts/update_k8s.sh

parent_path="$( cd "$(dirname "$0")" ; pwd -P )"
project_path=$parent_path/..
cd $project_path

kubectl apply -f jaspershop-api.yaml
kubectl apply -f jaspershop-db.yaml
kubectl apply -f rabbitmq.yaml
kubectl apply -f basic-ingress.yaml
kubectl apply -f jenkins-ingress.yaml
```

##### 第三個建置步驟

將 Deployments 滾動更新 (Rollout Restart)。

```sh
kubectl rollout restart Deployment/jaspershop
kubectl rollout restart Deployment/jaspershop-rabbitmq
```

等這些步驟都處理完之後，就可以實際 Commit 一些改動並 Push 到 Github 上，觀察一下這兩個 Job 有沒有正常執行，如果執行有報錯的話可以進 Job 裡面的 Console Output 看一下錯誤訊息是甚麼！

這樣就完成了 K8s + Jenkins 的專案部署了！


### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize]({% post_url 2020-06-03-k8s-jenkins-django-project-2 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (4) - Google Kubernetes Engine (下)]({% post_url 2020-06-10-k8s-jenkins-django-project-4 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (5) - Django Static File With Nginx]({% post_url 2020-06-14-k8s-jenkins-django-project-5 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (6) - Ingress]({% post_url 2020-06-16-k8s-jenkins-django-project-6 %})
<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (8) - 結語]({% post_url 2020-06-26-k8s-jenkins-django-project-8 %})

## 參考

[30. Docker 與 CI/CD (下)](https://ithelp.ithome.com.tw/articles/10209866)<br>
[Jenkins on Kubernetes Engine](https://cloud.google.com/solutions/jenkins-on-kubernetes-engine?hl=zh-tw)<br>
[基于 Jenkins 的 CI/CD (二)](https://www.qikqiak.com/k8s-book/docs/37.Jenkins%20Pipeline.html)