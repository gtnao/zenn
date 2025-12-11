---
title: "トランザクション(UNDO) （一人自作RDBMS Advent Calendar 2025 9日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」9日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day09)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day08 day09
```

## 今日のゴール

**BEGIN/COMMIT/ROLLBACK**を実装し、トランザクションの基本機能を動かせるようにします。ROLLBACKで変更を取り消すために、**Undo Log**を導入します。

今回の実装は簡易版で、クラッシュリカバリには対応していません（WALは後日実装）。まずは「ROLLBACKしたら変更が戻る」という基本動作を実現します。

## ACIDとAtomicity

トランザクションの特性として**ACID**（Atomicity、Consistency、Isolation、Durability）がよく知られています。ただし、ACIDの各特性の定義は曖昧な部分があり、『[データ指向アプリケーションデザイン](https://www.oreilly.co.jp/books/9784873118703/)』では「ほとんどマーケティング用語」とまで言われています。特にConsistencyはアプリケーション側の責務であり、DBMSの関心外です。

とはいえ馴染みのある用語なので、これを使って今回の実装の位置づけを説明すると:

- **Atomicity**: 今回実装するUndo LogとROLLBACKが担う
- **Isolation**: 今回は未実装（ロックやMVCCで後日対応）
- **Durability**: 今回は未実装（WALで後日対応）

今日はAtomicityの基本部分、「ROLLBACKしたら変更が取り消される」を実現します。

## Undo Logとは

ROLLBACKで変更を取り消すには、「何をどう変更したか」を記録しておく必要があります。これがUndo Logです。

各操作に対して、それを取り消すために必要な情報を記録します:

| 操作   | 記録する情報               | ROLLBACKでの処理               |
| ------ | -------------------------- | ------------------------------ |
| INSERT | 挿入した行のRID            | そのRIDの行を削除              |
| DELETE | 削除した行のRIDと元データ  | 元データを復元                 |
| UPDATE | DELETE + INSERTの2エントリ | INSERT分を削除、DELETE分を復元 |

UPDATEはday08で実装したようにDELETE + INSERTで実現しているため、Undo Logも2つのエントリになります。

```rust
pub enum UndoLogEntry {
    Insert { rid: Rid },
    Delete { rid: Rid, data: Vec<u8> },
}
```

## ROLLBACKの仕組み

ROLLBACKでは、Undo Logを**逆順**に適用します。逆順にする理由は、操作の依存関係を正しく巻き戻すためです。

例えばUPDATEの場合を考えます:

1. 実行時: DELETE(old) → INSERT(new) の順で実行
2. Undo Log: `[Delete{old_rid, old_data}, Insert{new_rid}]`
3. ROLLBACK: 逆順に処理 → INSERT分を削除 → DELETE分を復元

```rust
for entry in undo_log.into_iter().rev() {
    match entry {
        UndoLogEntry::Insert { rid } => {
            // INSERTの取り消し = その行を削除
            page_guard.delete(rid.slot_id)?;
        }
        UndoLogEntry::Delete { rid, data } => {
            // DELETEの取り消し = 元データを復元
            page_guard.restore(rid.slot_id, &data)?;
        }
    }
}
```

## ソフトデリートとの連携

day08で実装したソフトデリート（スロットの`length`を0にマーク）がここで活きてきます。

DELETEは行を物理的に削除するのではなく、スロットの`length`を0にマークするだけでした。つまり、データ自体はページ上に残っています。これにより、ROLLBACKでは`length`を元に戻すだけで復元できます。

```rust
pub fn restore(&mut self, slot_id: u16, data: &[u8]) -> Result<()> {
    let (offset, length) = self.get_slot(slot_id);
    if length != 0 {
        bail!("slot {} is not deleted", slot_id);
    }
    // データを書き戻してlengthを復元
    self.data[offset as usize..(offset + data.len()) as usize].copy_from_slice(data);
    self.set_slot(slot_id, offset, data.len() as u16);
    Ok(())
}
```

## Undo Logの記録タイミング

トランザクションがアクティブな場合、各ExecutorがUndo Logを記録します。例えばDeleteExecutorでは：

```rust
// Delete all targets and record undo log
for (rid, data) in targets {
    page_guard.delete(rid.slot_id)?;

    // Record undo log if in transaction
    if let Some(ref mut txn) = self.txn {
        if txn.is_active() {
            txn.add_undo_entry(UndoLogEntry::Delete { rid, data });
        }
    }
}
```

InsertExecutorも同様に、挿入成功後にUndo Logを記録します。UpdateExecutorはDELETE + INSERTで実装しているため、`Delete`と`Insert`の2エントリを記録します。

## Transactionの状態管理

コネクションごとにトランザクションの状態とUndo Logを管理します。

```rust
pub struct Transaction {
    pub state: TransactionState,  // Inactive or Active
    pub undo_log: Vec<UndoLogEntry>,
}
```

BEGIN/COMMIT/ROLLBACKは通常のSQLとは別に、トランザクション制御文として特別に処理します。

```rust
match &stmt {
    Statement::Begin => {
        txn.begin();  // stateをActiveに、Undo Logをクリア
        return Ok(ExecuteResult::Begin);
    }
    Statement::Commit => {
        txn.commit();  // Undo Logをクリア（変更確定）、stateをInactiveに
        return Ok(ExecuteResult::Commit);
    }
    Statement::Rollback => {
        let undo_log = txn.take_undo_log();
        ExecutionEngine::perform_rollback(bpm, undo_log)?;  // Undo Logを逆順適用
        return Ok(ExecuteResult::Rollback);
    }
    _ => {}
}
```

トランザクションがActiveな間、INSERT/DELETE/UPDATE実行時にUndo Logを記録します。COMMITするとUndo Logは不要になるのでクリアされます。

## コネクション切断時の処理

トランザクションがアクティブなままコネクションが切断された場合、自動的にROLLBACKします。これにより、中途半端な状態でデータが残ることを防ぎます。

```rust
if txn.is_active() {
    let undo_log = txn.take_undo_log();
    ExecutionEngine::perform_rollback(&bpm, undo_log)?;
}
```

## 動作確認

トランザクション内でINSERT/DELETE/UPDATEを実行し、ROLLBACKで取り消します。

```
-- 初期データ
INSERT INTO users VALUES (1, 'Alice');
INSERT INTO users VALUES (2, 'Bob');

-- トランザクション開始
BEGIN;
BEGIN

DELETE FROM users WHERE id = 1;
DELETE 1

INSERT INTO users VALUES (3, 'Charlie');
INSERT 0 1

UPDATE users SET name = 'Bob Updated' WHERE id = 2;
UPDATE 1

SELECT * FROM users;
 id |    name
----+-------------
  2 | Bob Updated
  3 | Charlie
(2 rows)

-- ROLLBACKで全て取り消し
ROLLBACK;
ROLLBACK

SELECT * FROM users;
 id | name
----+-------
  1 | Alice
  2 | Bob
(2 rows)
```

DELETE/INSERT/UPDATEの全ての変更がROLLBACKで取り消され、元の状態に戻っています。

COMMITした場合は変更が確定し、Undo Logはクリアされます。

## 現時点の制限

今回の実装は、ACIDで言えばAtomicityの一部のみです:

- **Durability未対応**: サーバがクラッシュするとUndo Logも失われます。WALを実装することで、クラッシュ後も復旧可能になります。
- **Isolation未対応**: 他のトランザクションから変更が丸見えです。例えばDirty Read（未コミット変更の読み取り）が起きます。

2つのコネクションで確認してみましょう:

```
-- Txn1: UPDATEしてまだCOMMITしていない
BEGIN;
UPDATE users SET name = 'DIRTY VALUE' WHERE id = 1;

-- Txn2: 別のコネクションからSELECT
SELECT * FROM users;
 id |    name
----+-------------
  2 | Bob
  1 | DIRTY VALUE   -- Txn1の未コミット変更が見えてしまう！
(2 rows)

-- Txn1: ROLLBACK
ROLLBACK;

-- Txn2: 再度SELECT
SELECT * FROM users;
 id | name
----+-------
  1 | Alice   -- 元に戻っている
  2 | Bob
(2 rows)
```

Txn2はTxn1がROLLBACKした「DIRTY VALUE」を読んでしまいました。

## 次回予告

明日は**2PLによる古典的なロック**を実装し、トランザクション間の分離を実現します。
