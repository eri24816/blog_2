---
title: 用 Jenkins 部屬 Docker 應用程式
date: 2024-11-18
authors:
  - eri24816
image: /Pasted image 20241116182611.png
draft: true
tags:
  - tech
categories: tech
series: 
summary:
---
我用 Vue 和 Python 做了 AI 生成音樂的應用程式。為了讓每個 commit 自動部屬到 server，我學了 Jenkins 怎麼用。

在 git repo 裡面新增 Dockerfile 和 Jenkinsfile

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
Jenkinsfile 做三件事: build image、刪掉舊 container、跑新 container。
```pipeline
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