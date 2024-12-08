---
title: "Pythonは本当に必要か？Rustで作るデータパイプライン"
date: 2024-12-09T00:00:00+09:00
draft: false
---

## はじめに

昨今のデータ基盤の流行りはELTだと思いますが、つまりT処理をBigQueryやSnowflakeといったDWHサービス上で実行してしまうケースが多く、ELはAPIやほかサービスからデータを引っこ抜いてDWHサービスに生データに近い形で登録しておきたいといったことがだいぶ多いのではないかと思っています。

そして最近はfivetran, troccoのような便利なSaaSもありその部分の実装を行う機会はだいぶ減ったとは思うのですが、それでもSaaSだと高い、オープンソースだと要件に合わないようなことも多くあり自前Pythonで実装だ！となることもまだまだあるのかと思います。

Pythonはデータ分野で超便利な言語で、チームのケイパビリティなどを考慮しても採用しやすいと思いますが、一方でしばしばシンプルな処理なのにメモリを食いつぶしツライ・・・みたいことが起こりがちだと思っています。

今回はデータパイプラインのEL処理をRustで実装することで、

- どの程度性能面のメリットを得られる可能性があるのか？
- 実装はシンプルにできるのか？

というところに焦点を当てて検証してみようと思います。

※ Rustはこのために勉強した感じなので、色々とアレなところがあったらすみません(自衛)

## パフォーマンス比較

まずシナリオとして以下を想定して実験してみます

- ローカルにあるCSVをBigQueryにinsert
- BQ Loadではなく、メモリ上でCSVを解釈し逐次insertする
  - BQにロードするための前処理をすることが多いという想定(ロードした時刻の付与とか、UTF変換とか)

とりあえず少しまとまったデータが欲しかったので、雑にPythonに作らせます(ChatGPTに作ってもらいました)

<details>
<summary>データ生成用コード(Python)</summary>

```python
import csv
import random
import time

# Configuration
num_rows = 1_000_000
output_file = "large_data.csv"

# Define sample data
names = ["Alice", "Bob", "Charlie", "David", "Eve"]
countries = ["USA", "Canada", "UK", "Australia", "Germany", "Japan"]

# Generate CSV
start_time = time.time()
with open(output_file, mode="w", newline="") as file:
    writer = csv.writer(file)
    # Write header
    writer.writerow(["id", "name", "age", "country", "signup_date"])

    # Write rows
    for i in range(1, num_rows + 1):
        writer.writerow([
            i,  # id
            random.choice(names),  # name
            random.randint(18, 99),  # age
            random.choice(countries),  # country
            f"2024-{random.randint(1, 12):02d}-{random.randint(1, 28):02d}",  # signup_date
        ])

end_time = time.time()
print(f"Generated {num_rows} rows in {end_time - start_time:.2f} seconds. Output: {output_file}")
```

</details>

比較用のコードです。どちらも1000件ずつinsertしていくという作りにしてあります。

<details>
<summary>Python</summary>

```python
import asyncio
import concurrent.futures
import time

import pandas as pd
from google.cloud import bigquery


def insert_batch(batch, table_ref):
    client.insert_rows_json(table_ref, batch)

def insert_to_bigquery_python(csv_file, dataset_id, table_id):
    table_ref = client.dataset(dataset_id).table(table_id)

    start_time = time.time()

    df = pd.read_csv(csv_file)

    rows = df.to_dict(orient="records")

    async def process_batches():
        with concurrent.futures.ThreadPoolExecutor() as executor:
            loop = asyncio.get_event_loop()
            tasks = []
            for i in range(0, len(rows), 1000):
                batch = rows[i:i+1000]
                tasks.append(loop.run_in_executor(executor, insert_batch, batch, table_ref))
            await asyncio.gather(*tasks)

    asyncio.run(process_batches())

    print(f"Python Insert Time: {time.time() - start_time} seconds")

if __name__ == "__main__":
    client = bigquery.Client()
    insert_to_bigquery_python("large_data.csv", "sample", "insert_test")
```

</details>

<details>
<summary>Rust</summary>

```rust
use csv::Reader;
use serde_json::json;
use reqwest::Client;
use tokio::task;
use std::fs::File;
use std::io;
use std::time::Instant;
use std::env;

#[tokio::main]
async fn main() -> io::Result<()> {
    let csv_file = "large_data.csv";
    let dataset_id = "DATASET_ID";
    let table_id = "TABLE_ID";
    let bigquery_url = format!(
        "https://bigquery.googleapis.com/bigquery/v2/projects/{project_id}/datasets/{dataset_id}/tables/{table_id}/insertAll",
        project_id = "GCP_PROJECT_ID",
        dataset_id = dataset_id,
        table_id = table_id
    );

    let client = Client::new();
    let mut rdr = Reader::from_reader(File::open(csv_file)?);

    let headers = rdr.headers()?.clone(); // Clone headers to avoid borrowing `rdr`
    let mut rows = Vec::new();
    let start_time = Instant::now();

    let mut tasks = vec![];

    for result in rdr.records() {
        let record = result?;
        let mut json_record = json!({});
        for (header, value) in headers.iter().zip(record.iter()) {
            json_record[header] = json!(value);
        }
        rows.push(json_record);

        if rows.len() >= 1000 {
            let batch = rows.clone();
            rows.clear();
            let client = client.clone();
            let url = bigquery_url.clone();

            tasks.push(task::spawn(async move {
                insert_to_bigquery(&client, &url, &batch).await;
            }));
        }
    }

    if !rows.is_empty() {
        let batch = rows;
        let client = client.clone();
        let url = bigquery_url.clone();

        tasks.push(task::spawn(async move {
            insert_to_bigquery(&client, &url, &batch).await;
        }));
    }

    for task in tasks {
        task.await.unwrap();
    }

    println!("Rust Insert Time: {:?} seconds", start_time.elapsed().as_secs());
    Ok(())
}

async fn insert_to_bigquery(client: &Client, url: &str, rows: &[serde_json::Value]) {
    let access_token = env::var("GOOGLE_ACCESS_TOKEN").expect("GOOGLE_ACCESS_TOKEN environment variable is not set");

    let payload = json!({ "rows": rows.iter().map(|r| json!({ "json": r })).collect::<Vec<_>>() });

    let response = client.post(url)
        .header("Authorization", format!("Bearer {}", access_token))
        .json(&payload)
        .send()
        .await;

    match response {
        Ok(resp) if resp.status().is_success() => println!("Batch inserted successfully"),
        Ok(resp) => eprintln!("Failed to insert batch: {:?}", resp.text().await),
        Err(err) => eprintln!("Error sending request: {:?}", err),
    }
}
```

</details>

3回ほど実行してみました。
環境はそれぞれDockerコンテナで固めて以下で計算しています。

- 実行時間はプログラムからの出力
- メモリはdocker statsで取得し、実行期間中の最大値 - 開始時

|       | Python実行時間(sec) | Pythonメモリ使用量最大値(MB) | Rust実行時間(sec) | Rustメモリ使用量最大値(MB) | 
| ----- | ------------------- | ---------------------------- | ----------------- | -------------------------- | 
| 1回目 | 57.13560914993286   | 379                          | 8                 | 221.79                     | 
| 2回目 | 58.565985918045044  | 379                          | 7                 | 211.259                    | 
| 3回目 | 58.839012145996094  | 379.5                        | 7                 | 204.959                    | 
| 平均  | 58.1802024          | 379.1666667                  | 7.333333333       | 212.6693333                | 


ちなみに並列化しないときはほぼ変わらない結果(どちらも大体400秒程度)だったのですが、非同期処理の効率の高さがかなり際立っていそうです。

実際のデータ量がもっと多かったり並列数が増えたりしてくるとこの差が顕著になってきそうな気もしますね。

## 実装のシンプルさ

見た目の話であれば慣れや好みの問題も多分に含んでしまうので、ここではRustで得られるメリットの1つである型安全性からもたらされるシンプルさについて書いてみようと思います。

Pythonはduck typingです。

個人的にはそれも悪くないと思っているのですが、プロダクションに乗せるコードだともう少しいい感じにしたいと思うときが多くあります。

例えば、以下のコードは本当にageがint値であることの保証はどこにもなく、例えば入力がユーザの作ったスプレッドシートのようなものを想定すると変な値が入ることは十分にあり、このコードが実行可能な保証はありません。

```python
import pandas as pd

df = pd.read_json("data.json")
print(df["age"] + 1)  # エラーの可能性がある
```

これを回避しようとすると、一つずつ型のチェックをするコードを書いてOKの場合処理すると言った形で、非常に見慣れたとてつもなく冗長なコードになると思います。

これがrustだと以下のように書けます。
(動かす場合、JSON文字列にあるコメントは削除してください)

```rust
use serde::Deserialize;
use serde_json::Value;

#[derive(Deserialize, Debug)]
struct Record {
    age: u32,
    name: String,
}

fn main() {
    let json_data = r#"
        [
            {"age": 25, "name": "Alice"},
            {"age": "invalid", "name": "Bob"}, // 壊れたデータ
            {"age": 30, "name": "Charlie"}
        ]
    "#;

    let records: Vec<Result<Record, serde_json::Error>> =
        serde_json::from_str::<Vec<Value>>(json_data)
            .unwrap()
            .into_iter()
            .map(|item| serde_json::from_value::<Record>(item))
            .collect();

    for record in records {
        match record {
            Ok(valid_record) => println!("Valid: {:?}", valid_record),
            Err(e) => eprintln!("Invalid record: {}", e),
        }
    }
}
```

まず入力されるデータは型としてしっかり定義し、読み取った結果が正常か異常かによって処理を簡単に分けられるので、実行時にクラッシュする可能性を大きく下げられます。
また型として明確なので、何らかの集計処理などの演算を行いたい場合も型に基づいた処理が行われるので、コードの意味も自明になるのが良いですね。

## まとめ

データ界隈のエンジニアの共通スキルセットがPythonと言えそうな現状を考えると、Rustの導入は慎重になるべきというのは前提としてあると思っています。
何よりPythonはデータに関して優れたエコシステムがあるため、特に作りはじめの段階においてはそれを手放してまで初手Rustというのは中々選択しにくいでしょう。

ただ、Rustを採用することで高速かつ安全なものが作れそうというのは実装を通して十分わかりました。

個人的には

- 処理時間が伸びていてなんとか短縮したい
- CloudRunのようなCPUやメモリ課金の従量課金部分を削減したい
- データパイプラインの安定性が特に求められる

のような、ある程度規模が大きかったり、データ活用が進んでいってクリティカルな利用が増えたシーンであれば学習コストを割いてでも採用してみても良さそう、とい感じかなと思いました。


ちなみに私はRustの学習にあたって以下の本を読みました。

- [Rustプログラミング完全ガイド 他言語との比較で違いが分かる！](https://book.impress.co.jp/books/1121101129)

Web上のリソースだと断片的になりがちなところが本として一通りまとまっているので、基礎力をつけるにはよさそうです。

(新しい言語を英語リソースで学ぶのは中々苦しかったですし...)

どこかで本番投入できる機会があったら是非やりたいなぁと思いながら、本記事は締めようと思います。。
