# pymysql を Lambda レイヤーにする方法

* Lambda 関数から MySQL へ接続する際に、pymysql を使用する場合、以下の手順で pymysql を Lambda レイヤーとして作成できます。

* マネジメントコンソールの CloudShell からでも実行できます。
    - 以下の手順は CloudShell での実行を前提にしています。

## 手順

### 1. ローカルでレイヤー用のパッケージを準備

```cmd
mkdir python
pip install pymysql -t python/
```

`python/` フォルダの中に pymysql がインストールされます。Lambda はレイヤーの中の `python/` ディレクトリを自動的にパスに追加するため、このフォルダ名は固定です。

### 2. ZIP ファイルに圧縮

**Linux / macOS:**
```bash
zip -r pymysql-layer.zip python/
```

### 3. AWS CLI でレイヤーを公開

```cmd
aws lambda publish-layer-version \
  --layer-name pymysql-layer \
  --zip-file fileb://pymysql-layer.zip \
  --compatible-runtimes python3.14 python3.13 python3.12 python3.11 python3.10 \
  --description "pymysql for RDS connection"
```

### 4. Lambda 関数にレイヤーをアタッチ


### マネジメントコンソールから行う場合

1. 上記の手順 1〜2 で ZIP を作成
2. Lambda コンソール → 「レイヤー」→「レイヤーの作成」
3. ZIP をアップロードし、互換ランタイム（Python 3.12 等）を選択
4. Lambda 関数の設定画面 →「レイヤー」→「レイヤーの追加」で紐付け

---

## Lambda 関数側のコード例

レイヤーを追加すれば、通常通り `import pymysql` できます。

```python
import pymysql

def lambda_handler(event, context):
    connection = pymysql.connect(
        host='RDSエンドポイント',
        user='admin',
        password='パスワード',
        database='データベース名',
        cursorclass=pymysql.cursors.DictCursor
    )

    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM users LIMIT 10")
        results = cursor.fetchall()

    connection.close()
    return results
```

---

## ポイント

- フォルダ構造は必ず `python/pymysql/...` の形にする（`python/` が最上位）
- ZIP のサイズ上限は 50MB（解凍後 250MB）。pymysql は軽量なので問題なし
- RDS に接続する場合、Lambda を VPC 内に配置し、セキュリティグループで MySQL ポート（3306）を許可する必要があります


---

# Python Lambda から PostgreSQL に接続するライブラリ

主に以下の選択肢があります。

| ライブラリ | 特徴 |
|-----------|------|
| **psycopg2** | 最も広く使われる PostgreSQL アダプタ。C拡張に依存するため、Lambda レイヤーには `psycopg2-binary` を使うのが簡単 |
| **psycopg (psycopg3)** | psycopg2 の後継。非同期対応、型ヒント充実。Pure Python モード (`psycopg[binary]`) もあり Lambda 向き |
| **pg8000** | Pure Python 実装。C拡張不要なのでレイヤー作成が最も簡単。パフォーマンスは psycopg2 より劣る |
| **asyncpg** | 非同期専用（asyncio）。高速だが Lambda の同期ハンドラとは相性が悪い |
| **SQLAlchemy** | ORM / SQL ツールキット。内部で psycopg2 や pg8000 をドライバとして使う |

---

## Lambda レイヤーとしての使いやすさ


### 最も手軽：pg8000

Pure Python なので、どの環境でも `pip install` するだけでレイヤーにできます。

```cmd
mkdir python
pip install pg8000 -t python/
zip -r pg8000-layer.zip python/
```

```cmd
aws lambda publish-layer-version \
  --layer-name pg8000-layer \
  --zip-file fileb://pg8000-layer.zip \
  --compatible-runtimes python3.14 python3.13 python3.12 python3.11 python3.10 \
  --description "pg8000 for RDS connection"
```

---

## 使い分けの目安

- **シンプルなクエリ実行** → `psycopg2-binary` または `pg8000`
- **C拡張のビルドを避けたい** → `pg8000`（Pure Python）
- **ORM を使いたい** → `SQLAlchemy` + `pg8000`（ビルド不要の組み合わせ）
- **最新機能・非同期対応** → `psycopg3`

RDS PostgreSQL に接続する場合も、pymysql のときと同様に Lambda を VPC 内に配置し、セキュリティグループでポート 5432 を許可する必要があります。
