---
title: "FivetranをTerraformで構成する方法"
date: 2022-09-08T22:00:00+09:00
draft: false
tags: [fivetran, terraform]
---

## はじめに
ELツールである[Fivetran](https://www.fivetran.com/)を試す機会があり、terraformも使えるということでどこまで使えるのか試してみました

## 構成

データの転送は `CloudSQL(GCP) -> Fivetran -> BigQuery` という経路で行うことにしました。

## CloudSQLの準備
CloudSQLはコンソール上から適当に作成しています。
その上でデータは面倒だったので[この手順](https://cloud.google.com/sql/docs/mysql/connect-instance-cloud-shell?hl=ja)で用意し、その上で以下のようにしてユーザを作成します。

```
CREATE USER 'fivetran'@'%' IDENTIFIED BY 'ぱすわーど'
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'fivetran'@'%'
```

## Terraform

- provider.tf
  - api_key, api_secretはWebUI上から発行しておいてください
```
terraform {
  required_providers {
    fivetran = {
      source = "fivetran/fivetran"
      version = "0.6.3"
    }
  }
}


provider "fivetran" {
  api_key = "APIKEY"
  api_secret = "APISECRET"
}
```

- main.tf
```terraform
resource "fivetran_group" "group" {
    name = "Terraform"
}

resource "fivetran_connector" "mysql" {
    group_id = fivetran_group.group.id
    service = "google_cloud_mysql"
    sync_frequency = 60
    paused = false
    pause_after_trial = false

    destination_schema {
        prefix = "test2_"
        name = "test2"
    }

    config {
        host = "ホスト名 or Global IP"
        port = 3306
        database = "guestbook"
        user = "fivetran"
        password = "パスワード"
    }
}

resource "fivetran_destination" "dest" {
    group_id = fivetran_group.group.id
    service = "big_query"
    time_zone_offset = "+9"
    region = "GCP_US_EAST4"
    trust_certificates = "true"
    trust_fingerprints = "true"
    run_setup_tests = "true"

    config {
        project_id = "GCPのプロジェクトID"
        data_set_location = "US"
    }
}
```

これをapplyすればリソースは作れるはずです

## apply後のUI操作
このままでは転送は開始されないので、一旦UIに戻って以下2つの作業をする必要があります

1. BigQuery用のサービスアカウントの紐付け

今回はfivetranが提供してくれるSAを自GCPプロジェクトのIAMに紐付けました。
BigQueryユーザをつければOKです。

2. CloudSQLの証明書の確認

WebUIにログインすると、CloudSQL周りの接続がおかしいという警告が出ているので再度コネクション編集画面に入り、SAVE&TESTを押下すると以下のような画面が出ますのでCONFIRMする必要があります。

![cloudsql](images/fivetran.png)


これであとは転送がかってに始まります！
簡単で良いですね！！
