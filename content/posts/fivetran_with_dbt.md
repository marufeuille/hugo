---
title: "Fivetran を触ってみて思ったことたち"
date: 2022-09-10T21:00:00+09:00
draft: false
tags: [fivetran]
---

# はじめに
前回に引き続きELツールである[Fivetran](https://www.fivetran.com/)を試しており、ちょいちょい触って気づいたことを書き留めておきます

# 所感
## 通知

エラー時の通知などは見た限りはメールのみに対応しています。Slack等には直接は送れなさそうです。この点はUIからだけの操作を考えるとツライかも。

今どきメールで拾いたいっていうニーズのほうが少なそうだが、逆にライトに使う層だとそれでも良い気がするので、そういうことなのだろうか。

## ログの転送
fivetran自体のログはWebUIから確認も可能ですが、Datadog, GCP Cloudloggingのような一般的なログ管理ソリューションに転送可能です。

先程の通知のところでメールのみと書きましたが、他のログ管理ソリューションに転送できればあとは大体やりたい放題ですね！

## dbt連携

まだベータですが個人的には結構良い感触を持ちました。

特にどちらも比較的価格は安いソリューションでありつつ、インフラを意識しないので最少人数で分析基盤を作っていくみたいなところでは結構良さそうです。

今回はオープンデータから稲作に関するデータと天気情報を持ってきてゴニョろうとする、みたいなデモを作ってみました。


生データをlakeにいれ、少し加工したものをwarehouse、分析に使う段をmartとわけてます。

### fivetran内の転送タスクに合わせて実行ができてワークフローが一定不要

上流のコネクタの実行に合わせて勝手に後続のdbtのタスクを流すことができます。
つまり、凝ったことをしなければワークフローの管理が不要ということですね。楽や・・・

もちろん、必要に応じてスケジュールの設定もできます。

### リネージュがソースからdbt内まで追える
こんな感じでソースからmartまでいい感じに可視化できました。
(文字等はあとから書き込んでいます)

![lineage](/images/lineage.png)

ineが稲作データですが、面倒になったのでwarehouse部分はすっ飛ばしてるのはご愛嬌。

fivetran内で全体のリネージュが出てくるので良いですね。

### dbtのログもちゃんと拾える

(転送設定の上で)cloudlogging上で確認したのですが、例えば成功時のログは以下のようにjsonで構造化されてcloudlogginから確認できます。

ちゃんと構造化されているので、サクッとメトリックを作ったり通知するのもやりやすそうに見えます。

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

dbt cloudとか使ってると、ログを取り出す方法が多分ない気がする(ちゃんと調べ尽くしてはいない)のでこのあたりもありがたい。。。

### dbtの実行はfivetranが担保してくれる

逆に言うとカスタマイズの部分で苦しむ可能性もあるかもですが、比較的小規模に使いたいケースだととても楽そうに見えます。

課金は気になる。

## まとめ

とりあえずこれでいい感あった。

今後改善してほしいなーという部分として

- dbt連携やログ連携等、terraform管理できない部分がちょっとある
- connectorのスケジュールが更新間隔しか選べない気がする。それでもいいのだが、dbtのスケジュールは時刻指定可能なので思想でもなさそうなのが気になった。
- CSV取り込み時に日本語ヘッダー行があると謎のアルファベットに変換されて何が何だか分からない事になる(マッピングさせてほしい)
