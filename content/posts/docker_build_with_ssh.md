---
title: "CloudBuild内でビルドするコンテナ内からプライベートリポジトリを参照する方法"
date: 2022-06-27T21:51:40+09:00
draft: false
tags: [cloudbuild, docker]
---

## 結論
- [先日](https://blog.marufeuille.dev/posts/poetry_private_repository_on_docker_on_cloudbuild/)、cloudbuild上で`docker build`するときに`--ssh`を渡す方法だとうまく動作しないと書きましたが、再検証したら動いたのでメモします。
- おそらく、known_hostsを追加していなかったとかそんな感じだと思われます・・・ださい。

- 追検証の体なので、そういう形式で書いていますので、結論だけ知りたい方は最下部に行ってください。

## 検証ログ
### on Local

適当なプライベートリポジトリ `marufeuille/privare_repository_test` を用意しローカル環境で以下を実行すると問題なくクローンできます（公開鍵は設定済みです）

```
# git clone git@github.com:marufeuille/privare_repository_test.git
Cloning into 'privare_repository_test'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
```

では、このRepositoryを `docker build` 時にcloneすることを考えます。

次のようなDockerfileを考えます。

```
FROM ubuntu:22.04

RUN apt update && apt install -y openssh-client git
RUN git clone git@github.com:marufeuille/privare_repository_test.git
```

そしてこれをbuildすると当然落ちます。

```
# docker build -t test .
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM ubuntu:22.04
 ---> 27941809078c
Step 2/3 : RUN apt update && apt install -y openssh-client git
 ---> Running in 23982a11b5e4

(中略)

Removing intermediate container 23982a11b5e4
 ---> 48d7bfc074b6
Step 3/3 : RUN git clone git@github.com:marufeuille/privare_repository_test.git
 ---> Running in b056b674da78
Cloning into 'privare_repository_test'...
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
The command '/bin/sh -c git clone git@github.com:marufeuille/privare_repository_test.git' returned a non-zero code: 128
```

なので、Dockerfileを次のように書き換え

```
FROM ubuntu:22.04
RUN apt update && apt install -y openssh-client git
RUN mkdir -m 700 ~/.ssh && ssh-keyscan -t rsa github.com > ~/.ssh/known_hosts
RUN --mount=type=ssh git clone git@github.com:marufeuille/privare_repository_test.git
```

次にssh-agentをたちあげ、その後適切なオプションを渡すことでコンテナ側に不要な秘密鍵を渡すことなくビルドすることができます。

```
# eval $(ssh-agent)
Agent pid 9334
# ssh-add /home/marufeuille/.ssh/id_rsa
Identity added: /home/marufeuille/.ssh/id_rsa (marufeuille@penguin)
# ssh-add -l
3072 SHA256:yYyoN155dKbZRmZEZm0rDWG/Az9tVi/32DWbxdtJHV8 marufeuille@penguin (RSA)
# DOCKER_BUILDKIT=1 docker build --ssh default -t test .
[+] Building 3.3s (6/7)                                                                                                                
[+] Building 6.8s (8/8) FINISHED                                                                                                       
 => [internal] load build definition from Dockerfile                                                                              0.2s
 => => transferring dockerfile: 279B                                                                                              0.0s
 => [internal] load .dockerignore                                                                                                 0.3s
 => => transferring context: 2B                                                                                                   0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04                                                                   0.0s
 => [1/4] FROM docker.io/library/ubuntu:22.04                                                                                     0.0s
 => CACHED [2/4] RUN apt update && apt install -y openssh-client git                                                              0.0s
 => [3/4] RUN mkdir -m 700 ~/.ssh && ssh-keyscan -t rsa github.com > ~/.ssh/known_hosts                                           1.5s 
 => [4/4] RUN --mount=type=ssh git clone git@github.com:marufeuille/privare_repository_test.git                                   3.3s
 => exporting to image                                                                                                            1.5s
 => => exporting layers                                                                                                           1.4s
 => => writing image sha256:0a4b9e970ed98155faac957cd472bee06bdcf6dc8defeae380e98834ee3eb894                                      0.0s
 => => naming to docker.io/library/test   
```

なんですが、これをそのままcloudbuildに持ち込むと動きませんでした。

### on cloudbuild

次に追加でこんな `cloudbuild.yaml` を用意します。

```
steps:
- name: 'gcr.io/cloud-builders/git'
  secretEnv: ['SSH_KEY']
  entrypoint: 'bash'
  args:
  - -c
  - |
    echo "$$SSH_KEY" >> /root/.ssh/id_rsa
    chmod 400 /root/.ssh/id_rsa
    ssh-keyscan -t rsa github.com >> /root/.ssh/known_hosts
  volumes:
  - name: 'ssh'
    path: /root/.ssh

- name: 'gcr.io/cloud-builders/docker'
  entrypoint: bash
  args:
  - -c
  - |-
  - set -eux
  - eval $(ssh-agent)
  - ssh-add /root/.ssh/id_rsa
  - docker build --ssh default -t test .
  env:
  - "DOCKER_BUILDKIT=1"
  volumes:
  - name: 'ssh'
    path: /root/.ssh

availableSecrets:
  secretManager:
  - versionName: projects/PROJECT_ID/secrets/secret-name/versions/latest
    env: 'SSH_KEY'
```

そして `gcloud builds submit .` をすれば動きますが、シークレットの設定やcloudbuildのSAにシークレットアクセサの付与をしてから実施してください。
