---
title: Continuously Deploy Docker Applications with Jenkins
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
  - automation
categories: tech
series: 
summary: Use Jenkins to automatically pull each commit from GitHub and deploy it to the server. It's very convenient!
---
I created an AI-generated music application using Vue and Python, and I used Jenkins to automatically pull each commit from GitHub and deploy it to the server. It's very convenient!

The initial setup involves installing Docker and Jenkins on the server. However, someone helped me with the installation, so I won’t go into the details here. (I don’t know how to install them either.)
# Add Jenkinsfile and Dockerfile
First, add Jenkinsfile and Dockerfile to your git repo.

The Jenkinsfile is used to define three tasks to perform whenever there’s a new commit: build the image, remove the old container, then run the new container.

```javascript
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

The content of the Dockerfile is not the focus of this discussion and may not always look like this, but I put it here FYI anyway:
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

# Set up the pipeline on Jenkins

First, install the GitHub plugin in Jenkins.

Then, create a new pipeline.

![](https://i.imgur.com/oSjajxr.png)
![](https://i.imgur.com/zt6vmdb.png)

Enter pipeline configuration, and specify the git repo and the build trigger
![](https://i.imgur.com/nKd12Lb.png)
![](https://i.imgur.com/lvVA2t3.png)

Finally, add a webhook to your GitHub repository, pointing to `<your jenkins url>/github-webhook/`. This way, GitHub will notify Jenkins to deploy whenever you push changes.
![](https://i.imgur.com/x8EonGV.png)

And that’s it! The automatic deployment setup is complete. You can try making a commit to test it—Jenkins should trigger the pipeline and start/update your application on the server.