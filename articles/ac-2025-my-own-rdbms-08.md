---
title: "UPDATE/DELETEと並行実行 （一人自作RDBMS Advent Calendar 2025 8日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」8日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day08)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day07 day08
```

## 今日のゴール

**UPDATE/DELETE文**の実装と、**並行実行**への対応を行います。今週からトランザクションの実装に入りますが、その準備としてこれらを実装しておきます。今日は新しい概念というよりは、これまでの実装の延長と下準備的な内容です。

## UPDATE/DELETEの実装方針

UPDATE/DELETEの実装方法にはいくつかのアプローチがあります。今回は、来週あたりにMVCC（Multi-Version Concurrency Control）を実装する予定があるため、MVCCと親和性の高いシンプルな方式を採用しました。

- **DELETE**: スロットの`length`を0にマークする（ソフトデリート）
- **UPDATE**: DELETE + INSERTで実現

この方式は物理削除と比べてシンプルで、MVCCでは「古いバージョンを残して新しいバージョンを追加する」という考え方になるため、今回の実装もその発想に近いものになっています。

### DELETE: ソフトデリート

```rust
pub fn delete(&mut self, slot_id: u16) -> Result<()> {
    if slot_id >= self.tuple_count() {
        bail!("slot {} does not exist", slot_id);
    }
    let (offset, length) = self.get_slot(slot_id);
    if length == 0 {
        bail!("slot {} is already deleted", slot_id);
    }
    // Mark as deleted by setting length to 0
    self.set_slot(slot_id, offset, 0);
    Ok(())
}
```

`get_tuple()`は`length == 0`のスロットを`None`として返すため、SeqScanは削除済みタプルを自動的にスキップします。

### UPDATE: DELETE + INSERT

```rust
// Delete old tuple
let page_arc = self.bpm.lock().unwrap().fetch_page_mut(rid.page_id)?;
let mut page_guard = page_arc.write().unwrap();
page_guard.delete(rid.slot_id)?;
drop(page_guard);
self.bpm.lock().unwrap().unpin_page(rid.page_id, true)?;

// Insert new tuple
let tuple_data = serialize_tuple(&new_values);
Self::insert_tuple(&self.bpm, &tuple_data)?;
```

### Row ID (RID)

DELETE/UPDATEでは対象タプルの物理位置を知る必要があるため、`Rid`を導入しました。

```rust
pub struct Rid {
    pub page_id: u32,
    pub slot_id: u16,
}

pub struct Tuple {
    pub values: Vec<Value>,
    pub rid: Option<Rid>,
}
```

SeqScanがタプルを返す際にRIDを付与し、DeleteExecutor/UpdateExecutorがそれを使って対象を特定します。

## 並行実行への対応

### Rustの所有権と並行処理

並行処理を導入すると、複数スレッドからBufferPoolManagerにアクセスする必要が出てきます。しかしRustの所有権システムでは、可変参照（`&mut`）は同時に一つしか存在できません。

day07までの実装:
```rust
pub struct SeqScanExecutor<'a> {
    bpm: &'a mut BufferPoolManager,  // 可変参照
    // ...
}
```

この形式では、複数のExecutorやスレッドでBufferPoolManagerを共有できません。

### Arc: 参照カウント方式のスマートポインタ

`Arc`（Atomic Reference Counted）は、複数の所有者間でデータを共有するためのスマートポインタです。

```rust
let bpm = Arc::new(BufferPoolManager::new(disk_manager));

// Arc::clone()で参照カウントを増やす（データはコピーされない）
let bpm_clone = Arc::clone(&bpm);
```

`Arc`は参照カウントをスレッドセーフに管理し、最後の参照がなくなった時点でデータを解放します。

### Mutex: 排他制御

`Arc`だけではデータを共有できますが、複数スレッドから同時に書き込むとデータ競合が起きます。実際、`Arc<BufferPoolManager>`を複数スレッドで使おうとすると、Rustコンパイラが「`BufferPoolManager`は`Sync`トレイトを実装していないのでスレッド間で安全に共有できない」とエラーを出します[^send-sync]。

`Mutex`は排他制御を提供し、同時に一つのスレッドだけがデータにアクセスできることを保証します。`Mutex<T>`は`T`が`Send`であれば`Sync`を実装するため、`Mutex`でラップすることでコンパイラに「このデータは排他制御されているから安全に共有できる」と伝えられます。

[^send-sync]: Rustではスレッド安全性を`Send`と`Sync`という2つのトレイトで表現します。`Send`は所有権を別スレッドに移動できること、`Sync`は参照を複数スレッドで共有できることを意味します。詳しくは[The Rust Programming Language - Extensible Concurrency with the Sync and Send Traits](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html)を参照してください。

```rust
let bpm = Arc::new(Mutex::new(BufferPoolManager::new(disk_manager)));

// lock()でロックを取得し、MutexGuardを得る
let mut guard = bpm.lock().unwrap();
guard.fetch_page(page_id)?;
// guardがスコープを抜けると自動的にロック解放
```

`lock()`は`MutexGuard`を返します。このGuardはスコープを抜けると自動的にロックを解放するため、ロックの解放忘れを防げます。これはRAII（Resource Acquisition Is Initialization）と呼ばれるパターンで、リソースの取得をオブジェクトの初期化時に、解放を破棄時に行うことで、リソース管理を安全に行う手法です[^raii]。

[^raii]: [RAII - Rust By Example](https://doc.rust-lang.org/rust-by-example/scope/raii.html)

### RwLock: 読み書きを区別するロック

BufferPoolManagerが返すPageも共有する必要があります。day07では`&mut Page`を返していましたが、day08では`Arc<RwLock<Page>>`を返すように変更しました。

```rust
// day07
pub fn fetch_page(&mut self, page_id: u32) -> Result<&mut Page>

// day08
pub fn fetch_page(&mut self, page_id: u32) -> Result<Arc<RwLock<Page>>>
```

`RwLock`は読み取りと書き込みを区別します:
- 読み取りロック（`read()`）: 複数のスレッドが同時に取得可能
- 書き込みロック（`write()`）: 排他的に一つのスレッドのみ取得可能

SELECTは読み取りのみなので複数同時実行可能、INSERT/UPDATE/DELETEは書き込みなので排他的に実行、という使い分けができます。

```rust
// 読み取り
let page_guard = page_arc.read().unwrap();
let tuple_data = page_guard.get_tuple(slot_id);
drop(page_guard);  // 明示的に解放（スコープを抜けても自動解放されるが、早めに解放したい場合）

// 書き込み
let mut page_guard = page_arc.write().unwrap();
page_guard.delete(slot_id)?;
drop(page_guard);
```

### SeqScanExecutorでの適用

これらを組み合わせたSeqScanExecutorの実装:

```rust
pub struct SeqScanExecutor<'a> {
    bpm: Arc<Mutex<BufferPoolManager>>,
    // ...
}

fn next(&mut self) -> Result<Option<Tuple>> {
    // 1. BPMのロックを取得してページを取得、すぐにロック解放
    let page_arc = self.bpm.lock().unwrap().fetch_page(self.current_page_id)?;

    // 2. ページの読み取りロックを取得（BPMはロックされていない）
    let page_guard = page_arc.read().unwrap();
    let tuple_data = page_guard.get_tuple(self.current_slot_id);
    // ...
    drop(page_guard);

    // 3. 再度BPMをロックしてunpin
    self.bpm.lock().unwrap().unpin_page(self.current_page_id, false)?;
    // ...
}
```

ポイントは、BPMのロックを長時間保持しないことです。`fetch_page`と`unpin_page`の呼び出し時のみBPMをロックし、ページの読み取り中は解放しておくことで、他のスレッドもBPMにアクセスできます。

## マルチスレッド接続

### thread::spawnによる並行処理

コネクションごとにスレッドを生成します。

```rust
for stream in listener.incoming() {
    match stream {
        Ok(stream) => {
            let conn = Connection::new(stream);
            let catalog = Arc::clone(&self.catalog);
            let bpm = Arc::clone(&self.bpm);

            thread::spawn(move || {
                Self::handle_client(conn, catalog, bpm)
            });
        }
        // ...
    }
}
```

`Arc::clone()`で参照を複製し、`move`クロージャで新しいスレッドに所有権を移動します。

### Instanceの変更

```rust
pub struct Instance {
    catalog: Arc<Catalog>,
    bpm: Arc<Mutex<BufferPoolManager>>,
}
```

CatalogとBufferPoolManagerをArcでラップし、複数スレッドで共有できるようにしました。

## 動作確認

```
INSERT INTO users VALUES (1, 'Alice');
INSERT INTO users VALUES (2, 'Bob');
INSERT INTO users VALUES (3, 'Charlie');

SELECT * FROM users;
 id |  name
----+---------
  1 | Alice
  2 | Bob
  3 | Charlie
(3 rows)

DELETE FROM users WHERE id = 2;
DELETE 1

SELECT * FROM users;
 id |  name
----+---------
  1 | Alice
  3 | Charlie
(2 rows)

UPDATE users SET name = 'Alice Updated' WHERE id = 1;
UPDATE 1

SELECT * FROM users;
 id |     name
----+---------------
  3 | Charlie
  1 | Alice Updated
(2 rows)
```

UPDATEの結果でid=1の行が末尾に移動しているのは、DELETE + INSERTで実装しているためです。リレーショナルモデルでは行に順序の概念がないため、これは正常な動作です。

## まとめと次回予告

今日はトランザクション実装への準備として、UPDATE/DELETEの実装と、Rustの並行処理プリミティブ（Arc、Mutex、RwLock）を使った並行実行への対応を行いました。

明日からは**トランザクション**の実装に入ります。まずはCOMMIT/ROLLBACKの簡易実装から始めます。
