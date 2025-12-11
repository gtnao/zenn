---
title: "トランザクション(S2PLによるIsolation) （一人自作RDBMS Advent Calendar 2025 10日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」10日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day10)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day09 day10
```

## 今日のゴール

昨日実装したトランザクション機能にはIsolationがありませんでした。今日は**行単位のロック**と**Strict Two-Phase Locking (S2PL)** を実装し、トランザクション間の分離を実現します。

## 並行実行における「正しさ」とは何か

データベースにおける**Isolation**とは、複数のトランザクションが並行実行されても、あたかも逐次的に実行されたかのような結果を保証する性質です。

最も単純にIsolationを実現する方法は**逐次実行**（トランザクションを一つずつ順番に実行）ですが、これではスループットが出ません。現実のDBMSでは複数のトランザクションを**並行実行**し、各トランザクションの個々の操作が時系列的に入り混じります。この操作の時系列を**スケジュール**と呼びます。

では、どのようなスケジュールなら「正しい」と言えるのでしょうか？ここで重要なのは、逐次実行が常に正しいという前提です。逐次実行であれば、各トランザクションは他のトランザクションの影響を一切受けません。したがって、**ある並行スケジュールが何らかの逐次スケジュールと「等価」であれば、それは正しい**と言えます。

この「等価」の定義の仕方によって、**Conflict Serializability**や**View Serializability**といった異なる基準が生まれます。実用上最も広く使われているのがConflict Serializabilityで、これは**競合する操作（同じデータへのアクセスで少なくとも一方がWrite）の順序関係が、ある逐次スケジュールと一致する**ことを要求します。

厳密な定義については[私の過去の記事](https://zenn.dev/primenumber/articles/dbms-isolation)で解説しています。

## Two-Phase Locking (2PL)

Conflict Serializabilityは「結果的に正しかった」という事後的な基準です。しかし現実のDBMSではトランザクションがリアルタイムに流れ込むため、**実行中から正しさを保証**する仕組みが必要です。

その代表的なプロトコルが**Two-Phase Locking (2PL)** です。

### 2PLの規則

2PLでは、トランザクションのロック操作を2つのフェーズに分けます：

1. **Growing Phase（成長相）**: ロックを取得できる。解放は禁止。
2. **Shrinking Phase（縮退相）**: ロックを解放できる。取得は禁止。

言い換えれば、「**一度でもロックを解放したら、その後は新たなロックを取得してはならない**」という制約です。

### なぜ2PLで正しさが保証されるのか

2PLを守ると、競合する操作の順序関係に矛盾が生じないことが保証されます。

直感的に説明すると：もしT1とT2の間で競合する操作があれば、その操作のためにロックを取得する必要があります。T1がT2より先にある操作をするには、T1が先にロックを取得しているはずです。2PLでは「解放してから取得」ができないので、T1→T2の順序が確定したら、それに矛盾するT2→T1の順序は発生しません。

### Shared LockとExclusive Lock

ロックには2種類あります：

- **Shared Lock（共有ロック、S）**: 読み取り時に取得。複数トランザクションが同時に保持可能。
- **Exclusive Lock（排他ロック、X）**: 書き込み時に取得。他の全てのロックと競合。

|                | S保持中 | X保持中 |
| -------------- | ------- | ------- |
| **S取得要求**  | 許可    | 待機    |
| **X取得要求**  | 待機    | 待機    |

この互換性は競合の定義と対応しています。Read-Readは競合しないのでS-S同士は共存可能、それ以外（Read-Write、Write-Write）は競合するので待機となります。

## Cascading AbortとStrict 2PL

基本的な2PLには問題があります。

1. T1がデータXを更新し、Shrinking Phaseでロックを解放
2. T2がその**未コミット**のXを読み取る
3. T1がAbort

この場合、T2が読んだXは無効な値です。T2もAbortしなければなりませんし、T2に依存するT3も...と**連鎖的なAbort（Cascading Abort）** が発生する可能性があります。

これを防ぐため、**Strict Two-Phase Locking (S2PL)** では、**Exclusive Lock（書き込みロック）をトランザクション終了時（COMMIT/ROLLBACK）まで保持**します。こうすれば未コミットのデータを他のトランザクションが読むことがなくなり、Cascading Abortを防げます。

実際には多くのDBMSは**Strong Strict 2PL (SS2PL)** を採用しており、Shared Lockも終了時まで保持します。今回の実装もSS2PLに相当します。

## LockManagerの実装

では実装に入ります。RID（Row ID）ごとのロック状態を管理する`LockManager`を導入します。

```rust
pub struct LockManager {
    lock_table: Mutex<HashMap<Rid, LockState>>,
    cond: Condvar,
    timeout: Duration,
}
```

### ロックとラッチ

ここで用語を整理しておきます。データベースの文脈では「ロック」という言葉が2つの異なる意味で使われます：

- **ロック（Lock）**: トランザクションがデータ項目（行など）に対して取得する**論理的な排他制御**。トランザクションの期間中保持され、COMMIT/ROLLBACKで解放される。今回実装しているのはこちら。
- **ラッチ（Latch）**: データ構造（ページ、ハッシュテーブルなど）への**物理的な排他制御**。OSのMutexやRwLockに相当し、クリティカルセクションの間だけ短時間保持される。day08で使った`RwLock`はこちら。

`LockManager`の`lock_table`を`Mutex`で保護しているのはラッチであり、`LockManager`が管理しているShared/Exclusive Lockはトランザクションレベルのロックです。

### Condvarによるロック待機

ロックが取得できない場合、トランザクションは待機する必要があります。単純にループで待つ（スピンロック）とCPUを無駄に消費するため、`Condvar`（条件変数）を使って効率的に待機します。

```rust
// ロック取得を試みる
pub fn lock(&self, txn_id: u64, rid: Rid, mode: LockMode) -> Result<(), LockError> {
    let mut table = self.lock_table.lock().unwrap();
    let state = table.entry(rid).or_insert_with(LockState::new);

    // すぐに取得できる場合
    if state.can_grant(txn_id, mode) {
        state.holders.insert(txn_id, mode);
        return Ok(());
    }

    // 待機キューに追加
    state.wait_queue.push_back(LockRequest { txn_id, mode });

    // Condvarでタイムアウト付き待機
    let result = self.cond.wait_timeout_while(table, self.timeout, |table| {
        // この条件がtrueの間、待機し続ける
        let state = table.get(&rid).unwrap();
        !state.holders.contains_key(&txn_id)  // まだロックを取得できていない
    }).unwrap();

    if result.1.timed_out() {
        // タイムアウト時は待機キューから削除
        let table = &mut *result.0;
        if let Some(state) = table.get_mut(&rid) {
            state.wait_queue.retain(|r| r.txn_id != txn_id);
        }
        return Err(LockError::Timeout);
    }
    Ok(())
}
```

`wait_timeout_while`のポイント：

1. **待機開始時**: Mutexのロックを**自動的に解放**し、スレッドをスリープさせる
2. **起床時**: Mutexのロックを**再取得**してから処理を続行
3. **条件チェック**: クロージャがtrueを返す間は待機を続ける

Mutexを解放してから待機するのが重要です。もしMutexを保持したまま待機すると、他のスレッドがロックテーブルにアクセスできなくなり、デッドロックが発生します。

### ロック解放と待機スレッドの起床

```rust
pub fn unlock_all(&self, txn_id: u64, held_locks: &HashSet<Rid>) {
    let mut table = self.lock_table.lock().unwrap();

    for rid in held_locks {
        if let Some(state) = table.get_mut(rid) {
            state.holders.remove(&txn_id);
            self.grant_waiting_locks(state);  // 待機中のトランザクションにロック付与
        }
    }

    self.cond.notify_all();  // 全待機スレッドを起床
}
```

`notify_all()`で全ての待機スレッドを起床させ、各スレッドは`wait_timeout_while`のクロージャで自分がロックを取得できたかを確認します。

### ロックの昇格（Shared → Exclusive）

同じトランザクションが既にShared Lockを保持している行に対してExclusive Lockが必要になる場合があります。例えば、SELECTで読んだ行をUPDATEする場合です。

```rust
fn can_grant(&self, txn_id: u64, mode: LockMode) -> bool {
    // 既にロックを保持している場合
    if let Some(held_mode) = self.holders.get(&txn_id) {
        // 既にExclusiveを保持、またはSharedで十分な場合
        if *held_mode == LockMode::Exclusive || mode == LockMode::Shared {
            return true;
        }
        // Shared → Exclusiveへの昇格
        // 自分だけがロックを保持していれば昇格可能
        if self.holders.len() == 1 {
            return true;
        }
        // 他のトランザクションもSharedを保持していると昇格できない（待機）
        return false;
    }
    // ...
}
```

昇格のルール：
- **既にExclusiveを保持** → そのままOK（より強いロックを既に持っている）
- **Sharedを保持していてSharedを要求** → そのままOK
- **Sharedを保持していてExclusiveを要求** → **自分だけがロック保持者なら昇格可能**、他のトランザクションもSharedを持っていると待機

他のトランザクションがSharedを保持している状態で昇格を許可すると、そのトランザクションが読み取り中のデータを書き換えてしまい、Non-Repeatable Readが発生するためです。

## ロック取得タイミング

### SELECTでのロック取得

```rust
fn next(&mut self) -> Result<Option<Tuple>> {
    let rid = Rid { page_id, slot_id };

    // 先にロックを取得
    if let Some(ref mut txn) = self.txn {
        if txn.is_active() {
            lock_manager.lock(txn.id, rid, LockMode::Shared)?;
            txn.add_lock(rid);
        }
    }

    // その後でタプルの存在を確認
    if let Some(tuple_data) = page_guard.get_tuple(slot_id) {
        // ...
    }
}
```

### DELETE/UPDATEでのロック昇格

DELETEやUPDATEでは、まずWHERE句の評価のためにSELECTが走り、対象行にShared Lockが取得されます。その後、実際に削除・更新する前にExclusive Lockに昇格します。

```rust
// DELETE: SELECTで特定した対象行に対してExclusiveロックに昇格
for (rid, _) in &targets {
    lock_manager.lock(txn.id, *rid, LockMode::Exclusive)?;
    txn.add_lock(*rid);  // 既にSharedで登録済みだが、ロックモードが更新される
}

// ロック取得後に削除実行
for (rid, data) in targets {
    page_guard.delete(rid.slot_id)?;
    txn.add_undo_entry(UndoLogEntry::Delete { rid, data });
}
```

## トランザクション終了時のロック解放

S2PLの核心は、**トランザクション終了まで全てのロックを保持**することです。COMMIT/ROLLBACK時に一括でロックを解放します。

```rust
Statement::Commit => {
    // COMMIT時：全ロックを解放
    let held_locks = txn.take_held_locks();
    lock_manager.unlock_all(txn.id, &held_locks);
    txn.commit();
}
Statement::Rollback => {
    // ROLLBACK時：Undo実行後に全ロックを解放
    let undo_log = txn.take_undo_log();
    ExecutionEngine::perform_rollback(bpm, undo_log)?;
    let held_locks = txn.take_held_locks();
    lock_manager.unlock_all(txn.id, &held_locks);
}
```

トランザクションは保持しているロックを`held_locks: HashSet<Rid>`で追跡し、終了時に`take_held_locks()`で取り出して`unlock_all()`に渡します。

## 動作確認

S2PLがどのような異常を防げるかを実際に確認します。ANSI SQL-92ではDirty Read、Non-Repeatable Read、Phantom Readという3つの異常が定義されています。

まずセットアップとして初期データを投入します。

```sql
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

### ROLLBACKの動作確認

まずトランザクションの基本動作を確認します。

```sql
BEGIN;
UPDATE users SET name = 'Alice Modified' WHERE id = 1;
SELECT * FROM users;
```

```
 id |      name
----+----------------
  2 | Bob
  1 | Alice Modified
(2 rows)
```

```sql
ROLLBACK;
SELECT * FROM users;
```

```
 id | name
----+-------
  1 | Alice
  2 | Bob
(2 rows)
```

ROLLBACKにより、更新が取り消されて元の状態に戻ります。これはday09で実装したUndo Logが正しく機能していることを示しています。

### Dirty Read（防止される）

```
T1: BEGIN;
T1: UPDATE users SET name = 'Modified' WHERE id = 1;

T2: BEGIN;
T2: SELECT * FROM users WHERE id = 1;  -- T1のExclusiveロック待ちでブロック

T1: ROLLBACK;

T2: -- ロック解放後に実行、元の'Alice'を読む
```

T1がExclusiveロックを保持しているため、T2はT1がコミットまたはロールバックするまで待機します。これにより、T2は未コミットの変更を読むことができません。

### Non-Repeatable Read（防止される）

```
T1: BEGIN;
T1: SELECT * FROM users WHERE id = 1;  -- 'Alice'（Sharedロック取得）

T2: BEGIN;
T2: UPDATE users SET name = 'Bob' WHERE id = 1;  -- T1のSharedロック待ちでブロック

T1: SELECT * FROM users WHERE id = 1;  -- まだ'Alice'
T1: COMMIT;  -- ロック解放

T2: -- UPDATE実行可能に
```

T1がSharedロックを保持し続けるため、T2のUPDATEはT1のコミットまで待機します。これにより、T1のトランザクション中は同じ行を読み直しても同じ値が返ります。

### Phantom Read（防止されない - 実際の実行結果）

行単位のS2PLではPhantom Readは防げません。以下は実際の実行結果です。

```sql
-- 初期状態
SELECT * FROM users WHERE id > 0;
```

```
 id | name
----+-------
  1 | Alice
  2 | Bob
(2 rows)
```

```sql
-- 別トランザクションで新しい行を挿入
INSERT INTO users VALUES (3, 'Charlie');

-- 再度同じ範囲クエリを実行
SELECT * FROM users WHERE id > 0;
```

```
 id |  name
----+---------
  1 | Alice
  2 | Bob
  3 | Charlie
(3 rows)
```

同じ範囲条件のクエリなのに、Charlieという新しい行（Phantom）が出現しています。これは、既存の行にはSharedロックを取得できますが、**まだ存在しない行にはロックを取得できない**ためです。Phantom Readを防ぐにはPredicate LockやGap Lock（MySQLのInnoDB等で実装）が必要ですが、今回は未実装です。

つまり、行単位のS2PLはANSI定義上は**Repeatable Read相当**となります。Isolation Levelの理論的背景については[私の過去の記事](https://zenn.dev/primenumber/articles/dbms-isolation)で体系的に解説しています。

### デッドロック

```
T1: BEGIN;
T1: UPDATE users SET name = 'T1' WHERE id = 1;  -- row 1にExclusiveロック

T2: BEGIN;
T2: UPDATE users SET name = 'T2' WHERE id = 2;  -- row 2にExclusiveロック

T1: UPDATE users SET name = 'T1' WHERE id = 2;  -- T2のロック待ち
T2: UPDATE users SET name = 'T2' WHERE id = 1;  -- T1のロック待ち → デッドロック

-- 30秒後にタイムアウト
ERROR: lock acquisition timeout (possible deadlock)
```

2PLはConflict Serializabilityを保証しますが、**デッドロックが発生しやすい**というトレードオフがあります。今回はタイムアウトで検知していますが、より洗練された方法としてWait-for Graph（待機グラフ）によるサイクル検出があります。

## 次回予告

今日はS2PLによるIsolationを実装しました。これでACIDのうちAtomicityとIsolationが揃いました。

明日は**Write-Ahead Logging (WAL)** を実装し、Durability（クラッシュリカバリ）に対応します。
