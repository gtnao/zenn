---
title: "Write-Ahead Logging (WAL) （一人自作RDBMS Advent Calendar 2025 11日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」11日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day11)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day10 day11
```

## 今日のゴール

これまでの実装では、トランザクションのAtomicityとIsolationを実現してきました。しかし、**Durability（永続性）** が欠けています。今日は**Write-Ahead Logging (WAL)** を実装し、クラッシュ時にもデータを復元できる基盤を整えます。

## 現状の問題

### BufferPoolの書き戻しタイミング

現在のBufferPoolManagerは、バッファプール（メモリ）がいっぱいになったときに、LRUポリシーでページを選んでディスクに書き戻します。つまり、**いつディスクに書き込まれるかはメモリの都合で決まり、トランザクションの状態とは無関係**です。

これにより、以下の問題が発生します：

1. **COMMITしたのに消える**: トランザクションがCOMMITしたが、変更がまだメモリ上にしかない状態でクラッシュすると、その変更は失われる
2. **ROLLBACKしたのに残る**: トランザクションがROLLBACKしたが、変更が既にディスクに書き込まれていた場合、不正なデータが残る

### COMMIT時にflushすれば解決？

「COMMITのタイミングで変更ページを全てflushすれば良いのでは？」と思うかもしれません。しかし、これでも問題は解決しません。

あるページには**複数のトランザクションによる変更**が混在している可能性があります。例えば：

- T1がページPの行Aを更新
- T2がページPの行Bを更新（まだCOMMITしていない）
- T1がCOMMIT

このとき、T1のためにページPをflushすると、T2のまだコミットしていない変更も一緒にディスクに書き込まれてしまいます。T2がROLLBACKしても、ディスク上には既にT2の変更が残っているのです。

ページ単位での書き込みでは、トランザクションの境界を正しく扱えません。

![COMMIT時にflushしても解決しない理由](https://storage.googleapis.com/zenn-user-upload/ec3566bd276d-20251213.png)

## Write-Ahead Logging (WAL)

この問題を解決するのが**Write-Ahead Logging (WAL)** です。

### WALの基本アイデア

WALの核心的なアイデアは非常にシンプルです：

**「データの変更内容を、ページへの書き込みより先にログファイルに書く」**

ログには「どのトランザクションが」「どのデータを」「どう変更したか」が記録されます。究極的に言えば、**WALさえあれば、データベースの全ての変更履歴がそこにある**のです。

### なぜページが必要なのか

「全ての情報がWALにあるなら、ページは不要では？」と思うかもしれません。理論的にはその通りですが、現実的には無理があります。

WALは**追記専用の時系列ログ**です。あるデータの最新値を知りたければ、ログを最初から全て読んで、そのデータへの変更を全て適用しなければなりません。これには2つの問題があります：

- **速度**: 10億件のログがあれば、1件の検索に10億件分の処理が必要
- **メモリ**: 全データの最新状態を得るには、全ログを処理してメモリ上に展開する必要がある

そこで、これまで扱ってきた**ページ**が重要になります。ページはデータの「現在の状態」を保持するスナップショットです。通常のクエリ実行時はページを参照し、クラッシュ時の復元にはWALを使います。

### STEAL/NO-FORCEとWrite-Aheadの意義

BufferPoolがページをディスクに書き出すタイミングには、2つのポリシーがあります。

**COMMIT時のポリシー（FORCE vs NO-FORCE）**
- **FORCE**: COMMIT時に変更ページを全てディスクに書き出す
- **NO-FORCE**: COMMIT時にページを書き出さなくてよい（後で書く）

**未コミットページの扱い（STEAL vs NO-STEAL）**
- **STEAL**: 未コミットのトランザクションが変更したページでも、バッファが足りなければディスクに書き出せる
- **NO-STEAL**: 未コミットのトランザクションが変更したページは書き出せない

パフォーマンスの観点からは**STEAL + NO-FORCE**が望ましいです：
- STEAL → バッファプールが足りなくなったら未コミットでも追い出せる（メモリ効率）
- NO-FORCE → COMMIT時に全ページflushしなくてよい（COMMITが速い）

しかし、一見するとこれは不整合を起こしそうです：

**STEALの問題**: 未コミットのデータがディスクに書かれた状態でクラッシュしたら？

```
T1: BEGIN
T1: UPDATE X = 100  (メモリ上のページを更新)
-- バッファプールが足りなくなり、ページをディスクに書き出し（STEAL）--
-- クラッシュ！ --
```

ディスク上にはT1の未コミットの変更が残っています。このままでは不正なデータです。

**NO-FORCEの問題**: コミット済みのデータがディスクにない状態でクラッシュしたら？

```
T2: BEGIN
T2: UPDATE Y = 200  (メモリ上のページを更新)
T2: COMMIT
-- ページはまだメモリ上にしかない（NO-FORCE）--
-- クラッシュ！ --
```

T2はコミット済みなのに、ディスク上には変更が反映されていません。

**WALが解決する**

WALが**ページより先に**ディスクに書かれていれば、両方の問題を解決できます：

- **STEALの問題** → WALを見れば「T1は未コミット」と分かるので、Undoで取り消せる
- **NO-FORCEの問題** → WALを見れば「T2はコミット済み」と分かるので、Redoで再適用できる

これが「Write-Ahead」の本質です。**ページをディスクに書く前に、必ずWALをディスクに書く**ことで、クラッシュ後に「何が起きたか」を判断でき、正しい状態に復元できます。

### Log Sequence Number (LSN)

各WALレコードには**LSN (Log Sequence Number)** という連番が付与されます。LSNは以下の役割を果たします：

1. **順序の保証**: WALレコードの時系列順序を保証
2. **Page LSN**: 各ページは「このページに最後に適用されたWALのLSN」を記録

Page LSNは、リカバリ時の**冪等性**を保証するために重要です。リカバリ時、Page LSNを見て「このページにはLSN=100まで適用済み」と分かれば、LSN=100以下のWALレコードはスキップできます。

明日実装するRecovery（ARIES）で、Page LSNを使った効率的なリカバリを行います。

## WalManagerの実装

では実装に入ります。WALレコードの種類とWalManagerを実装します。

### WALレコードの種類

```rust
pub type Lsn = u64;

#[derive(Debug, Clone)]
pub enum WalRecordType {
    Begin,
    Commit,
    Abort,
    Insert { rid: Rid, data: Vec<u8> },
    Delete { rid: Rid, data: Vec<u8> },
}

#[derive(Debug, Clone)]
pub struct WalRecord {
    pub lsn: Lsn,
    pub txn_id: u64,
    pub record_type: WalRecordType,
}
```

`Insert`と`Delete`には実際のタプルデータが含まれており、これを使ってRedo/Undoが可能です。

```
WALファイル構造:
┌─────────────────────────────────────────────────────────────────┐
│                        WAL File (append-only)                    │
├─────────────────────────────────────────────────────────────────┤
│  Record 1    │  Record 2    │  Record 3    │  Record 4    │ ... │
│  (LSN=1)     │  (LSN=2)     │  (LSN=3)     │  (LSN=4)     │     │
└─────────────────────────────────────────────────────────────────┘

各レコードの構造:
┌────────┬────────┬──────────┬──────┬──────────┬─────────────────┐
│  len   │  lsn   │  txn_id  │ type │ data_len │      data       │
│ 4bytes │ 8bytes │  8bytes  │ 1byte│  4bytes  │    variable     │
└────────┴────────┴──────────┴──────┴──────────┴─────────────────┘
```

### WALの書き込み

```rust
pub struct WalManager {
    writer: Mutex<BufWriter<File>>,
    current_lsn: AtomicU64,
    flushed_lsn: AtomicU64,
}

impl WalManager {
    // WALレコードを追加してLSNを返す
    pub fn append(&self, txn_id: u64, record_type: WalRecordType) -> Lsn {
        let lsn = self.current_lsn.fetch_add(1, Ordering::SeqCst);

        let record = WalRecord { lsn, txn_id, record_type };
        let data = record.serialize();

        let mut writer = self.writer.lock().unwrap();
        let len = data.len() as u32;
        writer.write_all(&len.to_le_bytes()).unwrap();
        writer.write_all(&data).unwrap();

        lsn
    }

    // WALをディスクにflush
    pub fn flush(&self) {
        let mut writer = self.writer.lock().unwrap();
        writer.flush().unwrap();
        writer.get_ref().sync_all().unwrap();  // fsyncで永続化を保証

        let current = self.current_lsn.load(Ordering::SeqCst);
        self.flushed_lsn.store(current - 1, Ordering::SeqCst);
    }

    // 指定LSNまでflush（WALプロトコル用）
    pub fn flush_to(&self, lsn: Lsn) {
        if self.flushed_lsn.load(Ordering::SeqCst) >= lsn {
            return;  // 既にflush済み
        }
        self.flush();
    }
}
```

`append`はWALレコードをバッファに追加し、`flush`で実際にディスクに書き込みます。`sync_all()`（fsync）を呼ぶことで、OSのバッファではなく物理ディスクまで書き込まれることを保証します。

### Page LSN

ページには最後に適用されたWALのLSNを記録します。

```rust
pub struct Page {
    pub data: [u8; PAGE_SIZE],
    pub page_lsn: Lsn,  // このページに最後に適用されたWALのLSN
}
```

## WAL書き込みのタイミング

### トランザクション制御

```rust
Statement::Begin => {
    txn.begin();
    wal_manager.append(txn.id, WalRecordType::Begin);
}
Statement::Commit => {
    // Commitレコードを書いてflush（Durabilityの保証）
    wal_manager.append(txn.id, WalRecordType::Commit);
    wal_manager.flush();
    // ロック解放
    lock_manager.unlock_all(txn.id, &txn.take_held_locks());
    txn.commit();
}
Statement::Rollback => {
    wal_manager.append(txn.id, WalRecordType::Abort);
    // Undo処理...
}
```

**COMMIT時にWALをflushする**のが重要です。これにより、COMMITが成功した時点で、その変更は確実にディスク上のWALに記録されています。

### データ操作

INSERTの例です。

```rust
// ページに挿入
let slot_id = page_guard.insert(&tuple_data)?;
let rid = Rid { page_id, slot_id };

// WALに記録し、Page LSNを更新
let lsn = wal_manager.append(
    txn.id,
    WalRecordType::Insert { rid, data: tuple_data.clone() },
);
page_guard.page_lsn = lsn;
```

操作のたびにWALに記録し、ページのLSNを更新します。

### BufferPoolでのWALプロトコル

ページをディスクに書き戻す前に、WALを先にflushします。

```rust
fn evict(&mut self, frame_id: usize) -> Result<()> {
    let frame = &mut self.frames[frame_id];
    if let Some(old_page_id) = frame.page_id {
        if frame.is_dirty {
            let page_guard = frame.page.read().unwrap();
            // WALプロトコル: ページを書く前にWALをflush
            self.wal_manager.flush_to(page_guard.page_lsn);
            self.disk_manager.write_page(old_page_id, &page_guard.data)?;
        }
        // ...
    }
    Ok(())
}
```

`flush_to(page_lsn)`により、このページに関連する全てのWALレコードがディスクに永続化されてから、ページが書き込まれます。

## 動作確認

WALが正しく記録されることを確認します。

### サーバー起動とデータ操作

```bash
# 初期化して起動
cargo run -- --init
```

別ターミナルで操作します。

```sql
BEGIN;
INSERT INTO users VALUES (1, 'Alice');
INSERT INTO users VALUES (2, 'Bob');
COMMIT;

BEGIN;
DELETE FROM users WHERE id = 1;
COMMIT;

BEGIN;
UPDATE users SET name = 'Bobby' WHERE id = 2;
COMMIT;
```

### サーバー再起動でWALを確認

Ctrl+Cでサーバーを停止し、再起動します。

```bash
cargo run
```

起動時にWALの内容が表示されます（今回は確認用にWALを読み取って表示するコードを入れています）。

```
[Instance] Existing WAL records:
  LSN=1 txn=1 Begin
  LSN=2 txn=1 Insert { rid: Rid { page_id: 0, slot_id: 0 }, data: [...] }
  LSN=3 txn=1 Insert { rid: Rid { page_id: 0, slot_id: 1 }, data: [...] }
  LSN=4 txn=1 Commit
  LSN=5 txn=2 Begin
  LSN=6 txn=2 Delete { rid: Rid { page_id: 0, slot_id: 0 }, data: [...] }
  LSN=7 txn=2 Commit
  LSN=8 txn=3 Begin
  LSN=9 txn=3 Delete { rid: Rid { page_id: 0, slot_id: 1 }, data: [...] }
  LSN=10 txn=3 Insert { rid: Rid { page_id: 0, slot_id: 2 }, data: [...] }
  LSN=11 txn=3 Commit
```

全てのトランザクションの操作がWALに記録されていることがわかります。UPDATEは内部的にDELETE + INSERTとして実装されているため、LSN=9とLSN=10のように分かれています。

## 次回予告

今日はWALを実装しました。しかし、現時点ではWALを「書いているだけ」で、クラッシュ時に実際にデータを復元する機能はまだありません。

明日は**ARIES (Algorithm for Recovery and Isolation Exploiting Semantics)** を実装し、WALを使ったクラッシュリカバリを実現します。これでACIDのDurabilityが完成します。
