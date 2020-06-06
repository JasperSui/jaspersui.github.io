---
layout:     post
title:      "[K8s + Jenkins] 將舊有 Django 專案翻新 (2) - Dockerize"
subtitle:   ""
date:       2020-06-03 04:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - k8s
    - Jenkins
---
## 正文

今天將 [Django + Nginx + uWSGI 實作的蝦皮購物網站](https://github.com/JasperSui/Django-Nginx-uWSGI-High-Performance-JasperShop) 這一份專案 Dockerize 了，也開了一個 [k8s-jenkins-django-jasper-shop](https://github.com/JasperSui/k8s-jenkins-django-jasper-shop) 的 GitHub Repository 給這一次的專案，未來在每一篇文章中都會帶上對應的 Commits，方便對照做了哪些事。

要使用 K8s 來部署 Django 專案，先決條件是要有一個 [Dockerized](https://docs.docker.com/engine/examples/) 的專案，這邊就不對細節多作介紹，主要是把實作的過程紀錄下來，要將一個傳統的 Django (Nginx + uWSGI) 專案 Dockerize 會需要以下幾個步驟：

1. 建立 Django project 的 Dockerfile (在 [k8s-jenkins-django-jasper-shop](https://github.com/JasperSui/k8s-jenkins-django-jasper-shop) 的路徑為 `/JasperShop/`)
    <br><br>
    ```
    FROM python:3.6.2

    WORKDIR /usr/src/app
    EXPOSE 8000

    ENV PYTHONUNBUFFERED 1

    RUN apt-get update && \
        apt-get install vim gettext -y \
        unzip

    COPY requirements.txt requirements.txt
    RUN pip install -r requirements.txt

    # Update the repository
    # "/usr/src/app/" is added in the directory to make sure that will always be in that location
    COPY . /usr/src/app/
    ```
    <br>
2. 建立 nginx 的 Dockerfile (`/nginx/`)
    <br><br>
    ```
    FROM nginx:latest

    COPY nginx.conf /etc/nginx/nginx.conf
    COPY my_nginx.conf /etc/nginx/sites-available/

    RUN mkdir -p /etc/nginx/sites-enabled/\
        && ln -s /etc/nginx/sites-available/my_nginx.conf /etc/nginx/sites-enabled/

    CMD ["nginx", "-g", "daemon off;"]
    ```
    <br>
3. 建立 docker-compose.yaml 檔
    <br><br>
    ```yaml
    version: '3'
    services:
        # MySQL db
        jaspershop-db:
            restart: always
            image: mysql
            environment:
                # these variables for db should be matched in settings.py
                MYSQL_ROOT_PASSWORD: rootroot 
                MYSQL_DATABASE: jaspershop
                MYSQL_USER: root
                MYSQL_PASSWORD: rootroot
            command: --default-authentication-plugin=mysql_native_password
            volumes:
                - ./mysql-data:/docker-entrypoint-initdb.d
            networks:
                - japsershop-network
            ports:
                - "3307:3306"

        # nginx
        nginx:
            build: ./nginx
            restart: always
            ports: 
                - "8000:80"
            volumes:
                - ./JasperShop:/usr/src/app/
                - ./log:/var/log/nginx

        # Django + uWSGI
        jaspershop:
            restart: always
            image: jaspershop:latest
            environment:
                - DJANGO_SETTINGS_MODULE=JasperShop.k8s_settings.local
            build: ./JasperShop
            depends_on:
                - jaspershop-db
            networks:
                - japsershop-network
            volumes:
                - ./JasperShop:/usr/src/app/
            command: uwsgi --ini uwsgi.ini

        # RabbitMQ
        jaspershop-rabbitmq:
            restart: always
            hostname: jaspershop-rabbitmq
            image: rabbitmq:management
            ports:
                - 15672:15672
                - 5672:5672
            volumes:
                - ./rabbitmq_data:/var/lib/rabbitmq

    networks:
        japsershop-network:
            driver: bridge

    volumes:
        mysql-data:  # db
    ```

之後只要照著文檔安裝 [Docker Compose](https://docs.docker.com/compose/install/) 後，用簡單的一行 `docker-compose up` 就能夠把這些在 `docker-compose.yaml` 裡設定的 Container 都執行起來。

接下來就可以去 [127.0.0.0:8000](http://127.0.0.1:8000) 看看服務有沒有正常運行，接下來要停止 Container 也只需要一組快捷鍵即可 (`Ctrl/Command + c`)，確認正常運行後今天的任務就結束了！

### 相關文章
[[K8s + Jenkins] 將舊有 Django 專案翻新 (1) - 前言]({% post_url 2020-06-02-k8s-jenkins-django-project-1 %})<br>
[[K8s + Jenkins] 將舊有 Django 專案翻新 (3) - Google Kubernetes Engine (上)]({% post_url 2020-06-06-k8s-jenkins-django-project-3 %})