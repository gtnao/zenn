---
title: "Aggregate / GROUP BY（一人自作RDBMS Advent Calendar 2025 18日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」18日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day18)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day17 day18
```

## 今日のゴール

昨日JOINを実装し、複数テーブルを結合できるようになりました。今日は**Aggregate（集計）** を実装し、GROUP BYやCOUNT/SUM/AVG/MIN/MAXなどの集計クエリを実行できるようにします。

## 集計処理の考え方

集計処理を素朴に実装するなら、全データをメモリに読み込んでから計算する方法が考えられます。しかしこれでは100万行のデータを集計するために100万行分のメモリが必要です。

そこで、データを1行ずつ読みながら集計状態を更新していく方法を採用します。

```
1. タプルを1行ずつ読む
2. 集計状態を更新
3. 全タプル処理後、最終結果を計算
```

メモリ使用量はO(グループ数)になります。100万行でも1000グループなら、1000個分の集計状態だけで済みます。

実装では、この集計状態を保持する構造体を`Accumulator`（累積器）と呼びます。各集計関数は以下のような状態を持ち、行ごとに更新していきます。

| 集計関数 | 保持する状態         | accumulate            | finalize    |
| -------- | -------------------- | --------------------- | ----------- |
| COUNT    | count: i64           | NULL以外なら+1        | count       |
| SUM      | sum: i64             | sum += value          | sum         |
| AVG      | sum: i64, count: i64 | sum += value, count++ | sum / count |
| MIN      | min: Option          | 小さければ更新        | min         |
| MAX      | max: Option          | 大きければ更新        | max         |

ポイントは、**入力値を保持しない**ことです。例えばAVGで値`[10, 5, 3]`を処理する場合：

```
accumulate(10) → sum=10, count=1
accumulate(5)  → sum=15, count=2
accumulate(3)  → sum=18, count=3
finalize()     → 18/3 = 6
```

入力値`[10, 5, 3]`自体は保持せず、`sum`と`count`だけを更新していきます。

### GROUP BYの処理

GROUP BYがある場合、グループごとに`Accumulator`のセットを用意します。グループキー（GROUP BY列の値）をHashMapのキーとして管理します。

```
HashMap<グループキー, Accumulatorセット>

┌─────────────────────────────────────┐
│ "apple"  → [COUNT: 3, SUM: 18]      │
│ "banana" → [COUNT: 3, SUM: 27]      │
└─────────────────────────────────────┘
```

同じグループキーを持つタプルは同じAccumulatorセットを共有し、最後にグループごとにfinalizeして結果を出力します。

### WHERE vs HAVING

SQLを学ぶとWHEREとHAVINGの違いに戸惑うことがあります。しかしExecutorの構造で考えると明快です。

```
┌──────┐    ┌──────────────┐    ┌───────────┐    ┌───────────────┐
│ Scan │ →  │ Filter(WHERE)│ →  │ Aggregate │ →  │ Filter(HAVING)│ → 出力
└──────┘    └──────────────┘    └───────────┘    └───────────────┘
                  ↑                   ↑
            個々の行を            集計結果を
            フィルタ              フィルタ
```

WHEREはAggregateの**前**、HAVINGはAggregateの**後**に位置します。

```sql
SELECT product, SUM(quantity)
FROM sales
WHERE quantity > 5        -- 先にquantity > 5の行だけ抽出
GROUP BY product
HAVING SUM(quantity) > 10 -- 集計後、合計が10超のグループだけ残す
```

## 実装

### Aggregate Executor

AggregateExecutorは、グループごとの集計状態をHashMapで管理します。

```
HashMap<グループキー, GroupState>

GroupState {
    group_values: グループキーの値
    accumulators: 各集計関数のAccumulator
}
```

実行フローは以下の通りです。

```
1. 子Executorからタプルを1行ずつ取得
2. グループキーを計算し、HashMapから該当グループを検索
   - なければ新規作成
3. そのグループのAccumulatorをaccumulate
4. 全タプル処理後、各グループをfinalizeして結果を返す
```

```rust
struct GroupState {
    group_values: Vec<Value>,
    accumulators: Vec<AggregateAccumulator>,
}

impl AggregateExecutor {
    fn process(&mut self) -> Result<()> {
        while let Some(tuple) = self.child.next()? {
            // グループキーを計算
            let key = self.compute_group_key(&tuple)?;

            // グループを取得または作成
            let group = self.groups.entry(key.clone()).or_insert_with(|| {
                GroupState {
                    group_values: key,
                    accumulators: self.aggregates.iter()
                        .map(|agg| AggregateAccumulator::new(&agg.func))
                        .collect(),
                }
            });

            // 各Accumulatorを更新
            for (i, agg) in self.aggregates.iter().enumerate() {
                let val = Self::evaluate_expr(&agg.arg, &tuple)?;
                group.accumulators[i].accumulate(&val);
            }
        }

        // 全グループをfinalizeして結果を構築
        for group in self.groups.values() {
            let result = self.build_result_tuple(group)?;
            self.results.push(result);
        }
        Ok(())
    }
}
```

### HAVING句の処理

HAVINGはAggregateExecutorの**出力**に対するフィルタです。既存のFilterExecutorを再利用します。

```
入力 → AggregateExecutor → FilterExecutor(HAVING) → 出力
```

## 動作確認

```bash
cargo run -- --init
psql -h localhost -p 5433
```

### テストデータの準備

```sql
CREATE TABLE sales (region VARCHAR, product VARCHAR, quantity INT, price INT);

INSERT INTO sales VALUES ('east', 'apple', 10, 100);
INSERT INTO sales VALUES ('east', 'apple', 5, 100);
INSERT INTO sales VALUES ('east', 'banana', 8, 50);
INSERT INTO sales VALUES ('west', 'apple', 3, 100);
INSERT INTO sales VALUES ('west', 'banana', 12, 50);
INSERT INTO sales VALUES ('west', 'banana', 7, 50);
```

### COUNT(\*)

```sql
SELECT COUNT(*) FROM sales;
```

```
 count
-------
     6
(1 row)
```

GROUP BYなしの場合、全行が1つのグループとして扱われます。

### SUM / AVG / MIN / MAX

```sql
SELECT SUM(quantity), AVG(quantity), MIN(quantity), MAX(quantity) FROM sales;
```

```
 sum | avg | min | max
-----+-----+-----+-----
  45 |   7 |   3 |  12
(1 row)
```

AVGは整数除算です（45 / 6 = 7）。

### GROUP BY（単一カラム）

```sql
SELECT product, COUNT(*), SUM(quantity) FROM sales GROUP BY product;
```

```
 product | count | sum
---------+-------+-----
 apple   |     3 |  18
 banana  |     3 |  27
(2 rows)
```

productごとにグループ化され、それぞれのグループで集計されます。

### GROUP BY（複数カラム）

```sql
SELECT region, product, SUM(quantity) FROM sales GROUP BY region, product;
```

```
 region | product | sum
--------+---------+-----
 east   | banana  |   8
 west   | apple   |   3
 west   | banana  |  19
 east   | apple   |  15
(4 rows)
```

(region, product)の組み合わせがグループキーになります。

### HAVING

```sql
SELECT product, SUM(quantity) FROM sales GROUP BY product HAVING SUM(quantity) > 10;
```

```
 product | sum
---------+-----
 apple   |  18
 banana  |  27
(2 rows)
```

apple(18)とbanana(27)はどちらもSUM > 10を満たすので両方残ります。

## 次回予告

今日でAggregate/GROUP BYを実装し、集計クエリを実行できるようになりました。

次回は**Index**を実装し、テーブルのフルスキャンを回避できるようにします。
