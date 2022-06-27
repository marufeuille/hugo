---
title: "CloudBuild内でビルドするコンテナ内のpoetryからプライベートリポジトリを参照する方法"
date: 2022-06-26T06:23:54+09:00
draft: false
tags: [cloudbuild, python, docker]
---
# 注意事項

その後の検証で `--ssh` オプションでも同等に動作することの検証が取れたので[こちら](https://blog.marufeuille.dev/posts/docker_build_with_ssh/)を参照してください。

# はじめに
長いタイトルになってしまいましたが、表題のとおりです笑

cloudbuild内でプライベートリポジトリを参照する方法も、`docker build`内でプライベートリポジトリを参照する方法もどちらも色々と記事があったのですが、併せると中々なかったのでメモしておきます

# やったこと
## 結論

ざっくりいうと、以下のようにすれば動きます。

ポイントは
- private repositoryの秘密鍵はSecretManager経由で渡す
- `docker build`はBUILDKITを有効化した上で、秘密鍵を--secretオプションでマウントする
- `Dockerfile`内では`--mount`オプションで秘密鍵を利用する


```yaml
steps:
- name: 'gcr.io/cloud-builders/git'
  secretEnv: ['SSH_KEY']
  entrypoint: 'bash'
  args:
  - -c
  - |
    echo "$$SSH_KEY" >> /root/.ssh/id_rsa
    chmod 400 /root/.ssh/id_rsa
    ssh-keyscan -t rsa github.com >  /root/.ssh/known_hosts
  volumes:
  - name: 'ssh'
    path: /root/.ssh

- name: gcr.io/cloud-builders/docker:latest
  dir: k8s_pipelines
  entrypoint: bash
  args:
  - -c
  - |-
    set -eux
    docker build --secret id=ssh,src=/root/.ssh/id_rsa -f Dockerfile -t tag-name:latest .
  env:
    - "DOCKER_BUILDKIT=1"
  volumes:
  - name: 'ssh'
    path: /root/.ssh

availableSecrets:
  secretManager:
  - versionName: projects/project_id/secrets/SSH_KEY/versions/latest
    env: 'SSH_KEY'
timeout: 3600s
```

```Dockerfile
FROM python:3.10

RUN pip install poetry

COPY pyproject.toml /workspace/
COPY poetry.lock /workspace/
WORKDIR /workspace

RUN mkdir -m 700 ~/.ssh
RUN ssh-keyscan -t rsa github.com > ~/.ssh/known_hosts
RUN --mount=type=secret,id=ssh,target=/root/.ssh/id_rsa poetry config virtualenvs.create true \
    && poetry config virtualenvs.in-project true \
    && poetry install
```

## CloudBuild内でシークレットを利用してプライベートリポジトリを参照する

こちらに関しては公式ドキュメントが参考になるので[こちら](https://cloud.google.com/build/docs/access-github-from-build)を確認してください。

## `docker build`内の`poetry`からプライベートリポジトリを参照できるようにする

~~これはネット上に色々転がっているのですが、よく出てくる`--ssh`でマウントする方法だと私のローカルではうまくいったものの、cloudbuild側ではうまくいきませんでした。~~

2022/06/28追記: できました。[こちら](https://cloud.google.com/build/docs/access-github-from-build)を確認してください。以下はSecretをマウントする方式で実現する場合の手順となります。

そこで、同様にBuildkitのドキュメント内を調べていくと[こちら](https://docs.docker.jp/develop/develop-images/build_enhancements.html#new-docker-build-secret-information)にシークレットのマウントのやり方に関する記載がありましたので、これを利用するようにしました。

具体的にはまずcloudbuild側(実行側)で

```bash
docker build --secret id=ssh,src=/root/.ssh/id_rsa -f Dockerfile -t tag-name:latest .
```

このようにすることで、`id=ssh`というシークレットにローカルの`/root/.ssh/id_rsa/`を割り当てています。
そして、Dockerfile内で

```Dockerfile
RUN --mount=type=secret,id=ssh,target=/root/.ssh/id_rsa poetry config virtualenvs.create true \
    && poetry config virtualenvs.in-project true \
```

このようにすると、この`RUN`中だけシークレットが参照できるようになります。