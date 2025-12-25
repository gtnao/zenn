---
title: "B+Tree（一人自作RDBMS Advent Calendar 2025 19日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」19日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day19)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day18 day19
```

## 今日のゴール

昨日Aggregate/GROUP BYを実装しました。今日からは**Index**の実装に着手します。Indexの実装は複雑なため、複数日に分けて進めます。

今日はIndexのデータ構造である**B+Tree**の基盤部分を実装します。具体的には：

- ページ構造の整理（HeapPage、LeafNode、InternalNode）
- B+Treeの挿入（Insert）と範囲検索（Range Scan）
- ユニットテストによる動作確認

## HeapPageへの名称変更

B+Tree用のページ（LeafNode、InternalNode）が追加されるため、これまで「Page」と呼んでいたテーブルデータ用のページを`HeapPage`に改名しました。Heap（ヒープ）とは、タプルを順序なく格納する方式のことで、[PostgreSQLでも同様の用語が使われています](https://www.interdb.jp/pg/pgsql01/03.html)。

## B+Treeの構造

Indexの実装にはHash Indexなど様々な方式がありますが、今回は最も一般的なB+Treeを実装します。B+Treeの詳しい説明は他に譲りますが、実装はなかなか複雑です。

```
                    [30]              ← InternalNode（ルート）
                   /    \
            [10|20]      [40|50]      ← InternalNode
           /   |   \    /   |   \
         [5] [15] [25] [35] [45] [55] ← LeafNode（リーフ）
          ↔   ↔   ↔   ↔   ↔   ↔
              リーフ間リンク
```

- **InternalNode**: キーと子ページへのポインタを持つ。どの子ノードに進むかを決める
- **LeafNode**: 実際のデータ（キーとRID）を格納し、隣接リーフへのリンクを持つ

## ページレイアウト

### LeafNode

```
┌─────────────────────────────────────────────────────────────┐
│ Header (16 bytes)                                           │
│   [0]      node_type = 0 (leaf)                             │
│   [1..3]   key_count                                        │
│   [3..5]   free_space_offset                                │
│   [5..9]   prev_leaf_page_id                                │
│   [9..13]  next_leaf_page_id                                │
├─────────────────────────────────────────────────────────────┤
│ Slot Array (grows →)     │ Free Space │ Entries (← grows)   │
│ [off|len][off|len]...    │            │ [key|rid][key|rid]  │
└─────────────────────────────────────────────────────────────┘
```

HeapPageと同様に、スロット配列は前方へ、エントリは後方へ伸びます。

リーフノードは隣接リーフへのポインタ（prev/next）を持つため、範囲検索時にリーフ間を効率的に移動できます。

### InternalNode

```
┌─────────────────────────────────────────────────────────────┐
│ Header (16 bytes)                                           │
│   [0]      node_type = 1 (internal)                         │
│   [1..3]   key_count                                        │
│   [3..5]   free_space_offset                                │
│   [5..9]   rightmost_child                                  │
├─────────────────────────────────────────────────────────────┤
│ Slot Array (grows →)     │ Free Space │ Entries (← grows)   │
│ [off|len][off|len]...    │            │ [key|child]...      │
└─────────────────────────────────────────────────────────────┘
```

内部ノードはキーと子ページIDを持ちます。n個のキーはn+1個の子ページへの振り分けに使われます。各エントリは`(key, child)`のペアで「このキー**未満**ならこの子へ」を表しますが、最後のキー**以上**の場合の行き先が必要です。それがヘッダーに格納する`rightmost_child`です。

```
キー: [10, 20, 30]
子:   [P0, P1, P2, rightmost=P3]

検索キー k に対して:
  k < 10   → P0 へ
  10 <= k < 20 → P1 へ
  20 <= k < 30 → P2 へ
  k >= 30  → P3（rightmost）へ
```

## 実装

### Insert

```rust
pub fn insert(&mut self, key: &IndexKey, rid: Rid) -> Result<()> {
    if self.root_page_id.is_none() {
        self.create_empty()?;
    }

    let root_id = self.root_page_id.unwrap();
    let leaf_page_id = self.find_leaf(root_id, key)?;

    // リーフに挿入、分割が発生したらsplit_keyと新ページIDが返る
    let split_result = self.insert_into_leaf(leaf_page_id, key, rid)?;

    // 分割が発生したら親に伝播
    if let Some((split_key, new_page_id)) = split_result {
        self.insert_into_parent(leaf_page_id, split_key, new_page_id)?;
    }

    Ok(())
}
```

`insert_into_leaf`ではリーフが満杯なら分割します。

```rust
fn insert_into_leaf(&mut self, page_id: u32, key: &IndexKey, rid: Rid)
    -> Result<Option<(IndexKey, u32)>>
{
    // ... ページ取得 ...

    if LeafNode::need_split(&page_guard.data, key) {
        let (new_page_id, new_page) = bpm.new_page()?;

        // 分割してsplit_keyを取得
        let split_key = LeafNode::split(&mut page_guard.data, &mut new_page_guard.data);

        // リーフ間リンクを設定
        LeafNode::set_next_leaf(&mut page_guard.data, Some(new_page_id));
        LeafNode::set_prev_leaf(&mut new_page_guard.data, Some(page_id));

        // split_keyより小さければ元のリーフ、そうでなければ新リーフに挿入
        if *key < split_key {
            LeafNode::insert(&mut page_guard.data, key, rid)?;
        } else {
            LeafNode::insert(&mut new_page_guard.data, key, rid)?;
        }

        return Ok(Some((split_key, new_page_id)));
    }

    LeafNode::insert(&mut page_guard.data, key, rid)?;
    Ok(None)
}
```

分割が発生するとsplit_keyを親に挿入します。親も満杯なら再帰的に分割し、ルートまで達したら新しいルートを作成します。

#### リーフの分割

```
分割前:
   [...| 60 |...]
         │
         ↓
  [10|20|30|40|50] (満杯、<60)

分割後:
   [...| 30 | 60 |...]
         │    │
         ↓    ↓
      [10|20]  [30|40|50]
      (<30)    (<60)
```

#### 内部ノードの分割

内部ノードの場合、中央キーは元のノードに残らず親に「押し上げ」られます。

```
分割前:
   [...| 60 |...]
         │
         ↓
  [10|20|30|40|50] (満杯、<60)

分割後:
   [...| 30 | 60 |...]
         │    │
         ↓    ↓
      [10|20]  [40|50]
      (<30)    (<60)
```

### Range Scan

```rust
pub fn range_scan(&self, start: Option<&IndexKey>, end: Option<&IndexKey>)
    -> Result<Vec<(IndexKey, Rid)>>
{
    let root_id = match self.root_page_id {
        Some(id) => id,
        None => return Ok(vec![]),
    };

    // 開始リーフを特定
    let start_leaf = match start {
        Some(key) => self.find_leaf(root_id, key)?,
        None => self.find_leftmost_leaf(root_id)?,
    };

    let mut results = Vec::new();
    let mut current_leaf = Some(start_leaf);

    // リーフチェーンを辿る
    while let Some(leaf_id) = current_leaf {
        // ... ページ取得 ...

        for i in 0..count {
            let key = LeafNode::get_key(&page_guard.data, i);
            if let Some(e) = end {
                if key >= *e {
                    return Ok(results);  // 終了条件
                }
            }
            let rid = LeafNode::get_rid(&page_guard.data, i);
            results.push((key, rid));
        }

        current_leaf = LeafNode::get_next_leaf(&page_guard.data);
    }

    Ok(results)
}
```

リーフ間リンク（next_leaf）を辿るため、内部ノードを経由せずに効率的にスキャンできます。

## 重複キーの扱い

同じキーを持つ複数のエントリが存在する場合、エントリを`(key, rid)`のペアで一意にソートします。

```rust
// キーが同じならRIDで比較
let cmp = match mid_key.cmp(key) {
    Ordering::Equal => mid_rid.cmp(rid),
    other => other,
};
```

これをやらないとどうなるか。キーだけで比較すると、重複キーの挿入位置が定まりません。

```
既存: [100, 100, 100]
新規: 100 を挿入 → どこに入れる？
```

二分探索の結果が不定になり、同じ操作をしても異なるツリー構造になる可能性があります。RIDを含めて比較することで、挿入位置が一意に決まります。

## テスト

これまでの記事では毎日実行結果を確認していましたが、今回はB+Treeの基盤部分のみでSQLからはまだ使えません。実装が複雑なこともあり、ユニットテストで動作確認しています。

- **基本操作**: 空ツリーの作成、キーの挿入と検索、IndexKeyのシリアライズ/デシリアライズ
- **分割**: 100件の順次挿入による分割、逆順挿入、500件挿入による多段分割
- **範囲検索**: 境界条件（開始が全エントリより前、終了が全エントリより後）、空範囲（開始=終了）、全件スキャン（None, None）、マッチなし
- **重複キー**: 同一キー10件、分割を伴う200件、1000件の大量重複、重複と一意の混在、RID順の確認
- **エッジケース**: 空ツリーでの検索/範囲検索、単一エントリ操作
- **異なる型**: NULLキー（NULL < 他の値の確認）、Varcharキーの辞書順、複合キー（Int, Varchar）

## 次回予告

今日でB+Treeの基盤部分を実装しました。

次回はこのB+Treeを使って、実際に**CREATE INDEX**や**IndexScanExecutor**を実装し、SQLクエリでIndexを活用できるようにします。
