---
title: "Slurmジョブが正常終了しない"
date: 2025-04-17T05:00:00+09:00
draft: false
---

# はじめに

最近お仕事で Slurm というスパコンでよく使われているらしいクラスタのジョブスケジューラを使っています。

これまではなんだか少し不安定だなー、使いにくいなーと思っていたのですが、大規模にコードを回していると特定のコードで不安定さが顕著な気がしました。

特に私のところで発生していた問題として

- ログを見る限り完了しているジョブなのに Slurm のジョブとしては完了せずに Running になっている
- 仕方ないので scancel すると高確率で `comp` ステータスになる

というものがありました。

これについて再現ができたのでまとめておこうと思います。

# 結局なんだったのか(結論)

結論を先に書くと以下のようでした。

- そのコードは内部で Python のスレッドを使用していたが、バックグラウンドで動いていた Thread が正常に終了していなくて、Job 的には完了ステータスにならなかった

- ファイルのロックを開放せずに取得しており、結果終了処理が長引いていた

# 再現実験

## 環境の用意

さすがに本番環境を使うわけに持ってことで、docker 上でクラスタを作れる [slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster) を用います

また、以下再現用のコードは Anthropic Claude 3.7 Sonnet に書いてもらってます。

## Slurm ジョブが終了しない件の再現

**コード**

```python
import os
import time
import logging
import multiprocessing
from multiprocessing import Pool
import threading

# わかりやすいようにログを出しておく
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger(__name__)

# データを処理する関数（ワーカープロセスで実行）
def process_data(item):
    pid = os.getpid()
    logger.info(f"プロセス {pid}: アイテム {item} の処理を開始")
    time.sleep(2)  # 処理時間をシミュレート
    logger.info(f"プロセス {pid}: アイテム {item} の処理を完了")
    return item * 2

# 明示的に終了させない限り終わらない関数
def background_monitor():
    while True:
        logger.info("バックグラウンドモニタリング実行中...")
        time.sleep(5)


def main():
    logger.info("メインプログラム開始")

    # 問題: 非デーモンスレッドの作成
    monitor_thread = threading.Thread(target=background_monitor)
    monitor_thread.start()
    logger.info("モニタリングスレッド開始")

    # 実処理しているふうの部分
    data = list(range(10))
    with Pool(processes=4) as pool:
        results = pool.map(process_data, data)
        logger.info(f"処理結果: {results}")

    # ここまで実行されるとスクリプトは「完了した」ように見える
    logger.info("すべての処理が完了しました")
    logger.info("プログラム終了")
    # しかし、非デーモンスレッドはまだ実行中のため
    # Pythonプログラムは終了せず、Slurmジョブも終了しない

if __name__ == "__main__":
    main()
```

**出力**

```
[root@slurmctld problem]# tail -f slurm-14.out
2025-04-16 12:51:47 - INFO - プロセス 451: アイテム 8 の処理を開始
2025-04-16 12:51:47 - INFO - プロセス 453: アイテム 9 の処理を開始
2025-04-16 12:51:47 - INFO - プロセス 452: アイテム 5 の処理を完了
2025-04-16 12:51:47 - INFO - プロセス 450: アイテム 6 の処理を完了
2025-04-16 12:51:48 - INFO - バックグラウンドモニタリング実行中...
2025-04-16 12:51:49 - INFO - プロセス 451: アイテム 8 の処理を完了
2025-04-16 12:51:49 - INFO - プロセス 453: アイテム 9 の処理を完了
2025-04-16 12:51:49 - INFO - 処理結果: [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
2025-04-16 12:51:49 - INFO - すべての処理が完了しました
2025-04-16 12:51:49 - INFO - プログラム終了
^C
[root@slurmctld problem]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      1  alloc c1
normal*      up 5-00:00:00      1   idle c2
[root@slurmctld problem]# squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                14    normal block.py     root  R       0:15      1 c1
```

はい、このように残ってしまいます。

いちいち scancel しないといけないので、逐次 Job を回しているケースなんかだとツラいですね...

## comp ステータスになる件の再現

**コード**

```python
import os
import sys
import time
import signal
import logging
import threading
import multiprocessing
import tempfile
import fcntl
import resource

# ロギング設定
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - PID:%(process)d - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger(__name__)

# グローバル変数
SHARED_FILE_PATH = os.path.join(tempfile.gettempdir(), 'slurm_lock_file')
EXIT_FLAG = multiprocessing.Event()
CHILD_PROCESSES = []

# 共有ファイルに対するロックを取得して保持するプロセス
def file_lock_process(lock_file_path):
    logger.info(f"ファイルロックプロセス開始")

    # ファイルロックを取得
    try:
        lock_file = open(lock_file_path, 'w')
        fcntl.flock(lock_file, fcntl.LOCK_EX | fcntl.LOCK_NB)
        logger.info(f"ファイルロック取得: {lock_file_path}")

        # 終了フラグが立つまでロックを保持
        while not EXIT_FLAG.is_set():
            time.sleep(1)
            sys.stdout.flush()  # ログをすぐに出力させる

        # 明示的に解放はしない → これがCOMPLETINGの原因になる
        logger.info("ファイルロックプロセス: 終了フラグ検知")
        # 通常はここでlock_file.close()するが、あえてしない

    except Exception as e:
        logger.error(f"ファイルロック処理でエラー: {e}")

def subprocess_spawner():
    logger.info("サブプロセス開始")

    def worker_process():
        # シグナルをブロック
        signal.signal(signal.SIGTERM, signal.SIG_IGN)
        signal.signal(signal.SIGINT, signal.SIG_IGN)

        logger.info(f"ワーカープロセス開始 PID: {os.getpid()}")

        # rlimitを最大に設定して終了しにくくする
        try:
            soft, hard = resource.getrlimit(resource.RLIMIT_NPROC)
            resource.setrlimit(resource.RLIMIT_NPROC, (hard, hard))
        except:
            pass

        # 無限ループ
        while not EXIT_FLAG.is_set():
            time.sleep(1)

        logger.info(f"ワーカープロセス: 終了フラグ検知 PID: {os.getpid()}")
        # 終了処理を遅延させる
        time.sleep(10)

    # 複数のワーカープロセスを作成
    processes = []
    for i in range(5):
        p = multiprocessing.Process(target=worker_process)
        p.daemon = False  # デーモンにしない
        p.start()
        processes.append(p)
        logger.info(f"ワーカー {i} 開始: PID {p.pid}")

    # グローバルリストに追加
    CHILD_PROCESSES.extend(processes)

    # 終了フラグが立つまで待機
    while not EXIT_FLAG.is_set():
        time.sleep(1)
        sys.stdout.flush()

    logger.info("サブプロセス: 終了フラグ検知")

    # プロセスを終了させようとするが、一部は応答しない想定
    for i, p in enumerate(processes):
        if i % 2 == 0:  # 偶数番目のプロセスのみ終了を試みる
            logger.info(f"プロセス {p.pid} の終了を試みます")
            try:
                p.terminate()
            except:
                pass

# シグナルハンドラ - 終了フラグをセットするだけで、実際には終了しない
def handle_signal(signum, frame):
    logger.info(f"シグナル {signum} を受信しましたが、すぐには終了しません")
    EXIT_FLAG.set()

    # 終了処理をするふりをする
    logger.info("終了処理中...")
    time.sleep(2)
    logger.info("まだ終了処理中...")

    # 実際には終了しない - これがCOMPLETINGの原因

def setup():
    logger.info("=== Slurm COMPLETING状態テストプログラム開始 ===")
    logger.info(f"メインプロセスPID: {os.getpid()}")

    # シグナルハンドラを設定
    signal.signal(signal.SIGTERM, handle_signal)
    signal.signal(signal.SIGINT, handle_signal)

    # 一時ファイルを作成
    with open(SHARED_FILE_PATH, 'w') as f:
        f.write('test')
    logger.info(f"共有ファイル作成: {SHARED_FILE_PATH}")

def main():
    setup()

    # 各問題プロセスを開始
    file_lock_proc = multiprocessing.Process(target=file_lock_process, args=(SHARED_FILE_PATH,))
    file_lock_proc.start()
    CHILD_PROCESSES.append(file_lock_proc)

    spawn_proc = multiprocessing.Process(target=subprocess_spawner)
    spawn_proc.start()
    CHILD_PROCESSES.append(spawn_proc)

    logger.info("すべてのプロセスを開始しました。メインループ開始")

    # メインループ
    try:
        while True:
            logger.info("メインプロセス実行中...")
            time.sleep(10)
            sys.stdout.flush()
    except KeyboardInterrupt:
        logger.info("キーボード割り込みを検知")
        EXIT_FLAG.set()

    logger.info("プログラム終了処理開始 - これは表示されるはずだが、完全には終了しない")

    # 終了処理を試みるが、すべてのプロセスが終了するわけではない
    EXIT_FLAG.set()
    for p in CHILD_PROCESSES:
        logger.info(f"プロセス {p.pid} の終了を待機中...")
        p.join(timeout=1)  # タイムアウト付きでjoin

    logger.info("メインプロセス終了 - しかしサブプロセスの一部は残る")

if __name__ == "__main__":
    main()
```

**実行結果**

```
[root@slurmctld problem]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      1  alloc c1
normal*      up 5-00:00:00      1   idle c2
[root@slurmctld problem]# squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                13    normal comp_tes     root  R       1:08      1 c1
[root@slurmctld problem]# scancel 13
[root@slurmctld problem]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      1   comp c1
normal*      up 5-00:00:00      1   idle c2
[root@slurmctld problem]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      1   comp c1
normal*      up 5-00:00:00      1   idle c2
[root@slurmctld problem]# sinfo
... (タイムアウトまで続く)
```

comp ステータスになることが確認されました。
やはりマルチスレッド環境かつファイルの出力回りが悪かったようです。

# まとめ

Slurm 上で Job をきれいに動かそうとすると少しハマりぽいんとがあることがわかりました。

避けるための私見としては

- そもそもスレッドを使うのは難しいので、Docker で固めてプロセスレベルで分散する。シングルスレッドなら難しさ自体だいぶ低減できる。
- リソースが使いっぱなしにならないように気をつける

とまあなんとも当たり前のことに帰結する感じかなと思ってます。
