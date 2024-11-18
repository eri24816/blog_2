---
title: 用 Jenkins 部屬 Docker 應用程式
date: 2024-11-18
authors:
  - eri24816
image: https://i.imgur.com/QEWQGr9.png
draft: false
tags:
  - tech
  - CICD
  - Docker
  - Jenkins
categories: tech
series: 
summary:
---
我用 Vue 和 Python 做了 AI 生成音樂的應用程式，然後用 Jenkins 讓每個 commit 自動從 GitHub pull 下來並佈署到 server，非常方便。
# 安裝 Docker 和 Jenkins

前置作業是要在 server 上安裝 Docker 和 Jenkins。不過有人幫我裝好了，所以這裡不寫細節。
# 新增 Jenkinsfile 和 Dockerfile
首先要在 git repo 裡面新增 Jenkinsfile 和 Dockerfile。

Jenkinsfile 用來描述每次有新 commit 要做的三件事: build image、刪掉舊 container、跑新 container。
```Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    sh 'docker build -t midi-gen .'
                }
            }
        }

        stage('Run') {
            steps { 
                script {
                    sh 'docker ps -qa --filter "name=midi-gen" | grep -q . && docker stop midi-gen && docker rm midi-gen || true'
                    sh 'docker run --name midi-gen -e CHECKPOINT_PATH=/volume/checkpoint.pt -v /home/eri/midi-gen-volume:/volume -p 8010:8010 --gpus all --rm -d midi-gen' 
                }
            }
        }
    }
}
```

Dockerfile 的內容不是本文重點，但放在這裡供參:
```Dockerfile
FROM node:22-alpine as frontend-builder
WORKDIR /frontend

COPY frontend/package.json frontend/package-lock.json ./
RUN npm install

COPY frontend/ .
RUN npm run build-only

FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime
WORKDIR /app/backend

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git
COPY backend/pyproject.toml .

RUN pip install -e .
COPY backend/ .

WORKDIR /app
COPY --from=frontend-builder /frontend/dist ./static

CMD ["uvicorn", "backend.app.main:app", "--host", "0.0.0.0", "--port", "8010"]
```

# 在 Jenkins 上設定 pipeline

先在 Jenkins 安裝 GitHub plugin。

然後新增 pipeline

![](https://i.imgur.com/oSjajxr.png)
![](https://i.imgur.com/zt6vmdb.png)

進入 pipeline 的 configuration，然後指定 git repo 和 build trigger
![](https://i.imgur.com/nKd12Lb.png)
![](https://i.imgur.com/lvVA2t3.png)

最後在 GitHub repo 上面新增 webhook，發送到 `<你的 jenkins url>/github-webhook/`。這樣 GitHub 就會在 push 的時候通知 jenkins 佈署。
![](https://i.imgur.com/x8EonGV.png)

然後就完成自動佈署的串接了。你可以試著 commit 看看，應該能觸發 Jenkins pipeline 並讓你的應用程式再 server 上啟動/更新。