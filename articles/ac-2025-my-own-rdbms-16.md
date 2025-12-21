---
title: "システムカタログ/CRETE TABLE（一人自作RDBMS Advent Calendar 2025 16日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」16日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day16)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day15 day16
```

## 今日のゴール

これまでの実装では`users`テーブルがハードコードされており、複数のテーブルを扱うことができませんでした。今日は**システムカタログ**を実装し、`CREATE TABLE`で動的にテーブルを作成できるようにします。

## これまでの問題点

複数テーブルを扱うには、以下の2つの問題を解決する必要があります。

1. **テーブル定義がハードコード**: `CREATE TABLE`で動的にテーブルを作成し、その定義を永続化する仕組みがない
2. **ページの所属テーブルが不明**: PostgreSQLではテーブルごとに別ファイルを作成しますが、今回は簡易的に1つのファイル（`table.db`）で全テーブルを管理するため、各ページがどのテーブルに属するかを追跡する仕組みが必要

## 設計

### ページの連結リスト

1つのファイルに複数テーブルのページが混在するため、同じテーブルのページをリンクリストで連結します。

```
Table A: Page 0 → Page 3 → Page 5 → END
Table B: Page 1 → Page 4 → END
Table C: Page 2 → END
```

各テーブルは`first_page_id`（最初のページID）を持ち、そこから`next_page_id`を辿ることで全ページにアクセスできます。ページヘッダーを拡張して`next_page_id`を追加します。

```
Page Header (12 bytes):
┌──────────────┬──────────────┬──────────────┬──────────────────┐
│ page_id (4)  │next_page_id(4)│tuple_count(2)│free_space_off(2)│
└──────────────┴──────────────┴──────────────┴──────────────────┘
```

`next_page_id`が`u32::MAX`の場合は「次のページなし」（チェーンの終端）を意味します。

### システムカタログ

テーブル定義のハードコード問題を解決するため、PostgreSQLに倣って**システムカタログ**を導入します。システムカタログはテーブル定義をデータベース内に保存する仕組みで、システムカタログ自体もテーブルとして実装します。

| テーブル名   | table_id | page_id | 用途         |
| ------------ | -------- | ------- | ------------ |
| pg_class     | 0        | 0       | テーブル一覧 |
| pg_attribute | 1        | 1       | カラム定義   |

#### pg_classのスキーマ

テーブルの基本情報を格納します。

| カラム名      | 型      | 説明           |
| ------------- | ------- | -------------- |
| table_id      | INT     | テーブルID     |
| name          | VARCHAR | テーブル名     |
| first_page_id | INT     | 最初のページID |

#### pg_attributeのスキーマ

各テーブルのカラム定義を格納します。

| カラム名         | 型      | 説明                                 |
| ---------------- | ------- | ------------------------------------ |
| table_id         | INT     | 所属するテーブルID                   |
| column_name      | VARCHAR | カラム名                             |
| data_type        | INT     | データ型（0=INT, 1=VARCHAR, 2=BOOL） |
| nullable         | BOOL    | NULL許容か                           |
| ordinal_position | INT     | カラムの順序                         |

システムカタログ自身の定義もpg_class/pg_attributeに格納されます（自己記述的）。Bootstrap時に以下のデータが挿入されます。

**pg_class（テーブル一覧）:**

| table_id | name         | first_page_id |
| -------- | ------------ | ------------- |
| 0        | pg_class     | 0             |
| 1        | pg_attribute | 1             |

**pg_attribute（カラム定義）:**

| table_id | column_name      | data_type   | nullable | ordinal_position |
| -------- | ---------------- | ----------- | -------- | ---------------- |
| 0        | table_id         | 0 (INT)     | false    | 0                |
| 0        | name             | 1 (VARCHAR) | false    | 1                |
| 0        | first_page_id    | 0 (INT)     | false    | 2                |
| 1        | table_id         | 0 (INT)     | false    | 0                |
| 1        | column_name      | 1 (VARCHAR) | false    | 1                |
| 1        | data_type        | 0 (INT)     | false    | 2                |
| 1        | nullable         | 2 (BOOL)    | false    | 3                |
| 1        | ordinal_position | 0 (INT)     | false    | 4                |

## 実装

### 初期化処理

データベース初期化時（`--init`）に、システムカタログを作成します。

ここで細かいですが実装上の問題があります。[Day14](https://zenn.dev/primenumber/articles/ac-2025-my-own-rdbms-14)で実装したMVCCでは、すべてのタプルにxmin/xmaxが必要です。しかし初期化時点ではまだ通常のトランザクションは開始できないため、`txn_id=1`を初期化専用として予約し使用します。

```rust
pub const SYSTEM_TXN_ID: u64 = 1;

pub fn bootstrap(bpm: &Arc<Mutex<BufferPoolManager>>, txn_manager: &TransactionManager) -> Result<()> {
    // Page 0: pg_class を作成
    let (_, pg_class_page) = bpm.new_page()?;
    {
        let mut page = pg_class_page.write().unwrap();

        // pg_class自身のエントリ（xmin=1, xmax=0）
        let tuple = serialize_tuple_mvcc(SYSTEM_TXN_ID, 0, &[
            Value::Int(0),  // table_id
            Value::Varchar("pg_class".to_string()),
            Value::Int(0),  // first_page_id
        ]);
        page.insert(&tuple)?;

        // pg_attributeのエントリ
        let tuple = serialize_tuple_mvcc(SYSTEM_TXN_ID, 0, &[
            Value::Int(1),  // table_id
            Value::Varchar("pg_attribute".to_string()),
            Value::Int(1),  // first_page_id
        ]);
        page.insert(&tuple)?;
    }

    // Page 1: pg_attribute を作成し、カラム定義を挿入
    // ...

    // システムトランザクションをCommit済みとしてCLOGに登録
    // これがないと、可視性判定でシステムカタログが見えない
    txn_manager.set_txn_status(SYSTEM_TXN_ID, TxnStatus::Committed);

    // 次のトランザクションIDは2から開始
    txn_manager.set_next_txn_id(SYSTEM_TXN_ID + 1);

    Ok(())
}
```

システムトランザクションをCLOGにCommittedとして記録しないと、MVCCの可視性判定でシステムカタログのタプルが見えなくなってしまいます。

### Catalogの動的読み込み

通常起動時（`--init`なし）は、pg_class/pg_attributeを読み込んでテーブル定義を復元します。

```rust
impl Catalog {
    pub fn load_from_disk(bpm: &Arc<Mutex<BufferPoolManager>>) -> Result<Self> {
        // pg_classをスキャンしてテーブル一覧を取得
        // (table_id, name, first_page_id) のリスト
        let tables = Self::scan_pg_class(bpm)?;

        // pg_attributeをスキャンしてカラム定義を取得
        // table_id -> Vec<Column> のマップ
        let columns = Self::scan_pg_attribute(bpm)?;

        // 両者を結合してテーブル定義を構築
        let mut catalog = Catalog::new();
        for (table_id, name, first_page_id) in tables {
            catalog.add_table(Table {
                id: table_id,
                name,
                first_page_id,
                columns: columns.get(&table_id).cloned().unwrap_or_default(),
            });
        }

        Ok(catalog)
    }
}
```

### SeqScanExecutorの変更

これまでは固定のページをスキャンしていましたが、テーブルの`first_page_id`から開始し、`next_page_id`でページチェーンを辿るように変更します。

```rust
fn next(&mut self) -> Result<Option<Vec<Value>>> {
    loop {
        // 現在のページからタプルを取得
        if let Some(tuple) = self.get_next_tuple_from_current_page()? {
            return Ok(Some(tuple));
        }

        // 現在のページを読み終えたら次のページへ
        let next_page_id = self.current_page.read().unwrap().next_page_id();

        if next_page_id == NO_NEXT_PAGE {
            return Ok(None);  // テーブルの終端
        }

        // 次のページをフェッチ
        self.current_page_id = next_page_id;
        self.current_page = self.bpm.fetch_page(next_page_id)?;
        self.current_slot = 0;
    }
}
```

INSERT時も同様にページを辿りますが、空きスペースのあるページを見つけるために全ページをスキャンする必要があります。PostgreSQLではこの問題を**Free Space Map（FSM）** で解決しています。FSMは各ページの空き容量を記録しており、INSERT時に効率的に挿入先ページを見つけられます。今回は簡易実装のためFSMは省略しています。

### CREATE TABLE Executor

`CREATE TABLE`文を処理するExecutorを実装します。新しいテーブル用のページを割り当て、pg_class/pg_attributeに登録します。

```rust
fn execute_create_table(&self, stmt: &CreateTableStatement) -> Result<()> {
    let new_table_id = self.catalog.read().unwrap().next_table_id();

    // 新しいテーブル用のページを割り当て
    let (new_page_id, new_page) = self.bpm.new_page()?;
    new_page.write().unwrap().set_next_page_id(NO_NEXT_PAGE);

    // pg_classに登録（通常のINSERTと同じ処理）
    self.insert_into_pg_class(new_table_id, &stmt.table_name, new_page_id)?;

    // pg_attributeに各カラムを登録
    for (ordinal, col) in stmt.columns.iter().enumerate() {
        self.insert_into_pg_attribute(new_table_id, col, ordinal)?;
    }

    // メモリ上のCatalogにも追加
    self.catalog.write().unwrap().add_table(/* ... */);

    Ok(())
}
```

### AllocatePage WALレコード

テーブルにデータを挿入していくと、1つのページでは足りなくなり新しいページを割り当てる必要があります。この時、ページチェーンを更新する操作（前のページの`next_page_id`を新しいページIDに設定）もWALに記録しないと、クラッシュ後にページチェーンが壊れてしまいます。

```rust
pub enum WalRecordType {
    // ...
    AllocatePage {
        page_id: u32,      // 新しく割り当てたページID
        table_id: u32,     // 所属するテーブルID
        prev_page_id: u32, // チェーンの前のページ（そのnext_page_idを更新）
    },
}
```

リカバリ時には、このレコードを再生して`prev_page_id`の`next_page_id`を`page_id`に設定します。これにより、クラッシュ後もページチェーンが正しく復元されます。

## 動作確認

```bash
cargo run -- --init
psql -h localhost -p 5433
```

### Test 1: CREATE TABLEとINSERT

```sql
CREATE TABLE users (id INT, name VARCHAR);
INSERT INTO users VALUES (1, 'Alice');
INSERT INTO users VALUES (2, 'Bob');
SELECT * FROM users;
```

```
 id | name
----+-------
  1 | Alice
  2 | Bob
(2 rows)
```

### Test 2: 複数テーブル

```sql
CREATE TABLE products (id INT, price INT);
INSERT INTO products VALUES (100, 500);
SELECT * FROM products;
```

```
 id  | price
-----+-------
 100 | 500
(1 row)
```

## 次回予告

今日でシステムカタログを実装し、CREATE TABLEで動的にテーブルを作成できるようになりました。

複数テーブルを扱えるようになったので、次回は**JOIN**を実装します。
