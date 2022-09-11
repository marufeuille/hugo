---
title: "Fivetran を触ってみて思ったことたち"
date: 2022-09-10T21:00:00+09:00
draft: false
tags: [fivetran]
---

# はじめに
前回に引き続きELツールである[Fivetran](https://www.fivetran.com/)を試しており、ちょいちょい触って気づいたことを書き留めておきます。

一般的な話は[ドキュメント](https://fivetran.com/docs/getting-started/core-concepts)によく書かれているので、さわる中で気になった点にフォーカスして書いてみます。

# 所感

## Transformations(dbt連携)

今回はfivetranのdbt連携を使い、オープンデータから稲作に関するデータと天気情報を持ってきてゴニョろうとする、みたいなデモを作ってみました。

[Github](https://github.com/marufeuille/dbt-with-fivetran-sample)

生データをlakeにいれ、少し加工したものをwarehouse、分析に使う段をmartとわけてます。

fivetran側でconnector/destinationの設定をして、githubにdbtのコードをアップロード、fivetran側で連携の設定を入れればあとは簡単に動作するようになります。

dbtを連携させるとこんな感じでソースからmartまでいい感じに可視化できました。
(文字等はあとから書き込んでいます)

![lineage](/images/lineage.png)

手軽に使えるのはとても良いですね。

### ログについて

ログは以下のようにWeb UI上から確認できます。

![logs](/images/logs.png)

また、ログの転送設定も可能で、例えば成功時のログは以下のようにjsonで構造化されてcloudlogginから確認できます。

dbtのログさえちゃんと構造化されているので、サクッとメトリックを作ったり通知するのもやりやすそうに見えます。

```
{
  "insertId": "9g7n20g1atrvd6",
  "jsonPayload": {
    "event": "dbt_run_succeeded",
    "created": "2022-09-10T07:14:44.435Z",
    "data": {
      "startTime": "2022-09-10T07:14:17.900Z",
      "result": {
        "stepResults": [
          {
            "step": {
              "processBuilderCommand": [
                "dbt",
                "test",
                "--models",
                "+sunshine_time",
                "+temperature"
              ],
              "name": "Scheduled: Test"
            },
            "failedModelRuns": 0,
            "successfulModelRuns": 0,
            "success": true,
            "commandResult": {
              "error": "",
              "exitCode": 0,
              "output": "07:14:39  Running with dbt=1.2.0\n07:14:39  Partial parse save file not found. Starting full parse.\n07:14:40  Found 2 models, 0 tests, 0 snapshots, 0 analyses, 285 macros, 0 operations, 0 seed files, 1 source, 0 exposures, 0 metrics\n07:14:40  \n07:14:40  [WARNING]: Nothing to do. Try checking your model configs and model specification args"
            },
            "startTime": "2022-09-10T07:14:34.035Z",
            "endTime": "2022-09-10T07:14:41.385Z"
          }
        ],
        "description": "Steps: successful 1, failed 0"
      },
      "dbtJobId": "lyre_stolen",
      "dbtJobName": "TEST_MODELS:sunshine_time_temperature",
      "startupDetails": {
        "jobId": 24518629,
        "type": "integrated_scheduler"
      },
      "endTime": "2022-09-10T07:14:41.614Z"
    }
  },
  "resource": {
    "type": "service_account",
    "labels": {
      "unique_id": "",
      "project_id": "*******",
      "email_id": "g-******@fivetran-production.iam.gserviceaccount.com"
    }
  },
  "timestamp": "2022-09-10T07:14:44.441Z",
  "severity": "INFO",
  "labels": {
    "levelValue": "801",
    "levelName": "INFO"
  },
  "logName": "projects/*******/logs/******-Warehouse-dbt-TEST_MODELS_sunshine_time_temperature",
  "receiveTimestamp": "2022-09-10T07:14:45.364961774Z"
}
```

このあたりは大変ありがたい。。。

## まとめ

とりあえずELツールとしては十分そうでした。

また、dbtの連携はベータながら今後が楽しみになる感じでした。料金とかどうなるんだろう。

今回触った範囲で気になったポイントもいくつかあって、このあたりは改善してくれると嬉しいですね。

- dbt連携やログ連携等、terraform管理できない部分がちょっとある
- connectorのスケジュールが更新間隔しか選べない気がする。それでもいいのだが、dbtのスケジュールは時刻指定可能なので思想でもなさそうなのが気になった。
- CSV取り込み時に日本語ヘッダー行があると謎のアルファベットに変換されて何が何だか分からない事になる(マッピングさせてほしい)
