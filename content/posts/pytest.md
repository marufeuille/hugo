---
title: "Pytest"
date: 2022-07-10T22:29:17Z
draft: false
---

# はじめに

pytestの書き方をよく忘れるのでまとめておく

なお、自分が思っている使い方的な側面もあるので必ずしもこの書き方でなくて良いし、もっと良い書き方があるかもしれない。

気づいたときに追記していきます。


# 使い方色々
## Mockオブジェクトを作成する

Pythonなので、適当なオブジェクトを作ってMockとして呼び出すということも可能だが、管理が面倒なので基本はMagicMockして作れば良い。

```
mocked = MagicMock(spec=...)
```

## 属性値のMockをする

PropertyMockを使えば良い

```
from unittest.mock import PropertyMock

from google.cloud import storage

def test_hoge():
    mocked_blob = MagicMock(spec=storage.Blob)
    suze = PropertyMock(return_value=10)
    type(mocked_blob).size = size
```

mockに対してではなく、mockの`type`に対して設定するのがミソである。


## インポートされたパッケージ内の特定のクラスをモックしたい

モックをするときは`patch`を使って基本的に以下のような形でできる

```python
from unittest.mock import patch

...

def test_hoge():
  ...
  with patch(..., autospec=True) as mocked:
    ...

```

このpatchに何を指定するかがちょっと悩みがちだが、その名前空間におけるオブジェクトを上書きするみたいな気持ちでやると良い。

例えば、`hoge/fuga/main.py`で定義されるモジュールがあったときに、以下のようにしてimport 文が書かれていたとする

```
from google.cloud import storage
```

これの `storage.Client` をモックしたいとしたときは

```python
from unittest.mock import patch

...

def test_hoge():
  ...
  with patch("hoge.fuga.main.storage.Client", autospec=True) as mocked:
    ...

```

とすればよい。これは `hoge.fuga.main`という名前空間（モジュール）にstorageをimportし、その中のClientをモックするというニュアンスである。


