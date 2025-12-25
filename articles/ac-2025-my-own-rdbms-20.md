---
title: "IndexScan（一人自作RDBMS Advent Calendar 2025 20日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」20日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day20)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day19 day20
```

## 今日のゴール

昨日B+Treeの基盤を実装しました。今日はこれをRDBMS全体に統合し、SQLから利用できるようにします。

- CREATE INDEX構文のサポート
- IndexScanExecutorの実装
- INSERT/UPDATE時のインデックス更新
- 簡易オプティマイザによるIndex Scan選択

## メタページ

昨日の実装では、B+Treeのルートページは`BTree`構造体が直接保持していました。しかしこれには問題があります。

```
ルート分割前:
  pg_index: root_page_id = 5
  B-Tree: [5]

ルート分割後:
  pg_index: root_page_id = ??? ← 更新が必要
  B-Tree: [7] ← 新ルート
         /  \
       [5]  [6]
```

ルートが分割されるたびにカタログ側である`pg_index`を更新すると、実装が複雑になります。

[PostgreSQLでも同様の構造が採用されており](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/README)、メタページ（metapage）がルートページへの参照を保持しています。これにより、B+Tree自身がルートを管理できるようになります。

```
pg_index: meta_page_id = 10 (メタページ、固定)
               ↓
         [MetaPage: root=7]
               ↓
              [7] ← ルート（分割で変わる）
             /  \
           [5]  [6]
```

### メタページのレイアウト

かなり単純ですが、以下のようなレイアウトになっています。

```
┌─────────────────────────────────────────────────────────────┐
│ [0]      node_type = 2 (meta)                               │
│ [1..5]   root_page_id                                       │
│ [5..]    reserved                                           │
└─────────────────────────────────────────────────────────────┘
```

B+Treeの作成時にメタページと初期ルート（空のリーフ）を作成し、ルート分割時にはメタページを更新します。

```rust
pub fn create_empty(&mut self) -> Result<()> {
    // メタページを作成
    let (meta_page_id, meta_page) = bpm.new_page()?;
    MetaNode::init(&mut meta_page.write().unwrap().data);

    // 初期ルート（リーフ）を作成
    let (root_page_id, root_page) = bpm.new_page()?;
    LeafNode::init(&mut root_page.write().unwrap().data);

    // メタページにルートを記録
    MetaNode::set_root_page_id(&mut meta_page.write().unwrap().data, Some(root_page_id));

    self.meta_page_id = Some(meta_page_id);
    Ok(())
}
```

## システムカタログ

インデックスのメタデータを管理するために`pg_index`システムテーブルを追加します。

| カラム       | 型      | 説明                               |
| ------------ | ------- | ---------------------------------- |
| index_id     | Int     | インデックスID                     |
| index_name   | Varchar | インデックス名                     |
| table_id     | Int     | 対象テーブルID                     |
| column_ids   | Varchar | カラムインデックス（カンマ区切り） |
| meta_page_id | Int     | B+Treeメタページ                   |

`pg_class`や`pg_attribute`と同様に、起動時にブートストラップ処理で作成されます。

## CREATE INDEX

`CREATE INDEX idx_users_id ON users (id)` のような構文をサポートします。

### 処理の流れ

1. 新しいインデックスIDを採番
2. B+Treeを作成（メタページ + 初期ルート）
3. 既存のテーブルデータをスキャンしてインデックスに投入
4. `pg_index`にメタデータを登録

既存データの投入部分を見てみましょう。

```rust
// テーブルの全ページを走査
while current_page_id != NO_NEXT_PAGE {
    let page = bpm.fetch_page(current_page_id)?;

    for slot_id in 0..tuple_count {
        if let Some(tuple_data) = page.get_tuple(slot_id) {
            let (_, _, values) = deserialize_tuple_mvcc(tuple_data, &schema)?;

            let key_values: Vec<Value> = column_ids
                .iter()
                .map(|&col_idx| values[col_idx].clone())
                .collect();
            let key = IndexKey::new(key_values);
            let rid = Rid { page_id: current_page_id, slot_id };
            btree.insert(&key, rid)?;
        }
    }

    current_page_id = page.next_page_id();
}
```

可視性に関係なく全タプルをインデックスに追加しています。タプルが見えるかどうかの判定はクエリ時にIndexScanExecutorが行います。

## IndexScanExecutor

```rust
fn open(&mut self) -> Result<()> {
    // B+Treeから範囲検索でRID一覧を取得
    let mut btree = BTree::new(Arc::clone(&self.bpm), key_schema);
    btree.set_meta_page_id(self.index_def.meta_page_id);

    let results = btree.range_scan(
        self.start_key.as_ref(),
        self.end_key.as_ref(),
    )?;

    // RIDを保存
    self.rids = results.into_iter().map(|(_, rid)| rid).collect();
    Ok(())
}

fn next(&mut self) -> Result<Option<Tuple>> {
    while self.current_idx < self.rids.len() {
        let rid = self.rids[self.current_idx];
        self.current_idx += 1;

        // RIDからヒープページを取得
        let page = self.bpm.fetch_page(rid.page_id)?;
        let tuple_data = page.get_tuple(rid.slot_id)?;

        let (xmin, xmax, values) = deserialize_tuple_mvcc(&tuple_data, &schema)?;

        // MVCC可視性チェック
        if is_tuple_visible(xmin, xmax, &self.snapshot, self.txn_manager) {
            return Ok(Some(Tuple { values, rid: Some(rid) }));
        }
    }
    Ok(None)
}
```

`open()`でB+Treeからマッチする全RIDを取得し、`next()`で1件ずつヒープからタプルを読み取ります。MVCCの可視性チェックを行うことで、他のトランザクションがコミットしていない変更は見えないようにしています。

## INSERT/UPDATE時のインデックス更新

テーブルにデータが追加・更新されたとき、インデックスも同期する必要があります。

### INSERT時

タプル挿入後、そのテーブルに存在する全インデックスに対してエントリを追加します。

```rust
// タプル挿入後
let indexes = self.catalog.get_indexes_for_table(self.table_id);
for index_def in indexes {
    let mut btree = BTree::new(Arc::clone(&self.bpm), key_schema);
    btree.set_meta_page_id(index_def.meta_page_id);

    let key_values: Vec<Value> = index_def.column_ids
        .iter()
        .map(|&col_idx| self.values[col_idx].clone())
        .collect();
    let key = IndexKey::new(key_values);
    btree.insert(&key, rid)?;
}
```

### UPDATE時

MVCCでは更新は「旧タプルの論理削除 + 新タプルの挿入」として実装されています。旧タプルはxmaxが設定されるため、IndexScan時のMVCC可視性チェックで自動的に除外されます。そのため、新タプルのインデックスエントリを追加するだけで済みます。

ただし、旧タプルを指すインデックスエントリは残ったままになります。本来はVACUUM時にヒープとインデックスの両方から不要なエントリを削除する必要がありますが、今回は実装していません。

## 簡易オプティマイザ

本来はPlanner/Optimizerを実装してクエリプランを生成しますが、今回は一旦Executor構築時のロジックに簡易的なものを入れています。WHERE句を解析し、使えるインデックスがあればIndex Scanを選択します。現時点では`column = literal`形式の等価条件のみをサポートしています。

```rust
fn try_find_index_for_where(
    catalog: &Catalog,
    table_id: u32,
    where_clause: &Option<AnalyzedExpr>,
) -> Option<(IndexDef, IndexKey)> {
    let where_expr = where_clause.as_ref()?;

    // column = literal の形式から (column_index, value) を抽出
    let (column_index, value) = extract_equality_condition(where_expr)?;

    // そのカラムにインデックスがあるか確認
    for index_def in catalog.get_indexes_for_table(table_id) {
        if index_def.column_ids == [column_index] {
            let key = IndexKey::new(vec![value.clone()]);
            return Some((index_def, key));
        }
    }
    None
}
```

インデックスが使われていることを確認できるよう、ログに出力しておきます。

```
[Optimizer] Using IndexScan with index 'idx_users_id'
```

## 実行例

```sql
-- テーブル作成
> CREATE TABLE users (id INT, name VARCHAR);
1 row affected

-- データ挿入
> INSERT INTO users VALUES (1, 'alice');
> INSERT INTO users VALUES (2, 'bob');
> INSERT INTO users VALUES (3, 'charlie');

-- インデックス作成
> CREATE INDEX idx_users_id ON users (id);
1 row affected

-- インデックスを使った検索
> SELECT * FROM users WHERE id = 2;
[Optimizer] Using IndexScan with index 'idx_users_id'
+----+-----+
| id | name|
+----+-----+
| 2  | bob |
+----+-----+
```

## 現状の課題

今日の実装にはいくつかの問題が残っています。

**WALが未対応**: インデックスの変更がWALに記録されていないため、クラッシュするとインデックスが失われます。

**並行制御が未対応**: 複数のトランザクションが同時にインデックスを操作した場合の問題があります。

- Split中に別のトランザクションがReadを実行したら？
- CREATE INDEX中にInsertやUpdateが走ったら？

B+Treeの並行制御には[Latch Crabbing](https://15445.courses.cs.cmu.edu/fall2023/notes/09-indexconcurrency.pdf)と呼ばれる手法があります。これは親ノードから子ノードへ移動する際、子のラッチを取得してから親のラッチを解放することで、ツリー構造が途中で変わらないことを保証します。さらに「安全な」ノード（分割やマージが起きない）であれば先祖のラッチを早期に解放でき、並行性を高められます。

## 次回予告

次回はこれらの課題に対処し、インデックスのWAL対応と並行制御を実装します。
