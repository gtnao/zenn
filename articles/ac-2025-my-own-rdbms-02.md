---
title: "Page （一人自作RDBMS Advent Calendar 2025 2日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」2日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day02)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day01 day02
```

## 今日のゴール

データを固定サイズのブロック（**Page**）に整理し、Tupleの管理が効率的に行える**Slotted Page**構造を実装します。

## なぜPageが必要か

昨日の実装では、Tupleを単にファイルに追記していくだけでした。この方式には問題があります。

1. **ランダムアクセス不可**: 可変長データがあるため、N番目のTupleに直接ジャンプできない
2. **削除や更新ができない**: 削除後の隙間を埋められない、更新でサイズが変わると元の場所に収まらない可能性がある
3. **メモリ管理が困難**: ファイル全体をメモリに読み込むのは非効率

RDBMSは通常、データを固定サイズのPageに分割して管理します。PostgreSQLのデフォルトは8KB、MySQLは16KBです。
固定サイズにすることで、Page N番は`N * PAGE_SIZE`バイト目から始まることが保証され、ランダムアクセス可能になります。

今回は動作確認をしやすくするため64バイトという小さなサイズにしています。

## Slotted Pageとは

Slotted Pageは、可変長のTupleを効率的に格納するためのPage構造です。PostgreSQLで採用されています。

```
+------------------+
| Header           |  page_id, tuple_count, free_space_offset
+------------------+
| Slot Array       |  [offset, length] × tuple_count
|   →→→            |  （前方に成長）
+------------------+
| Free Space       |
|                  |
+------------------+
| Tuple Data       |  （後方に成長）
|   ←←←            |
+------------------+
```

なぜSlot Arrayが必要かというと、Tupleは可変長なのでPage内のどこに配置されるかが不定です。Slotは固定サイズなので、N番目のSlotの位置は計算で求められ、O(1)でTupleにアクセスできます。

Slot配列は前から後ろへ、Tupleデータは後ろから前へ成長していき、両者がぶつかったらPageがいっぱいになったということです。

また、なぜ逆方向に成長させるかというと、もし両方を前から配置しようとすると：

- Tupleのサイズがバラバラなので、Slot Arrayのためにどれだけの領域を確保すべきか事前にわからない
- Slot Arrayが伸びるたびに全Tupleを後ろにずらす必要がある

逆方向から成長させることで、両者が動的に領域を分け合えます。

## バイナリレイアウト

今回実装するPageのレイアウトは以下の通りです。

```
Page (64 bytes for testing)
+--------+--------+--------+--------+--------+--------+--------+--------+
|          page_id (4B)             | tuple_cnt (2B)  | free_offset (2B)|
+--------+--------+--------+--------+--------+--------+--------+--------+
|  Slot 0 offset  |  Slot 0 length  |  Slot 1 offset  |  Slot 1 length  |
+--------+--------+--------+--------+--------+--------+--------+--------+
|                             ...                                       |
+--------+--------+--------+--------+--------+--------+--------+--------+
|                             Free Space                                |
+--------+--------+--------+--------+--------+--------+--------+--------+
|                             Tuple Data                                |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

- **Header (8 bytes)**
  - `page_id`: 4 bytes - Pageの識別子
  - `tuple_count`: 2 bytes - 格納されているTupleの数
  - `free_space_offset`: 2 bytes - 空き領域の終端（＝Tuple領域の開始位置）。初期値はPAGE_SIZE
- **Slot Array (4 bytes per slot)**
  - 各Slotは`offset`（2 bytes）と`length`（2 bytes）で構成
- **Tuple Data**
  - 後ろから前に向かって成長

今回は最小限のHeaderだけを入れていますが、PostgreSQLも本質的には同じような構造になっています。

## 実装

今回からコードをファイルに分割しています（`page.rs`、`disk.rs`、`table.rs`、`tuple.rs`）。

### Page構造体

```rust
pub const PAGE_SIZE: usize = 64; // small for testing

const HEADER_SIZE: usize = 8;
const SLOT_SIZE: usize = 4;

pub struct Page {
    pub data: [u8; PAGE_SIZE],
}
```

Page全体を固定サイズのバイト配列として保持します。

### Insert

Tupleの挿入は以下の手順で行います:

1. 空き領域が十分かチェック（Tupleサイズ + Slotサイズ）
2. `free_space_offset`を更新してTupleデータを書き込み
3. 新しいSlotを追加

```rust
pub fn insert(&mut self, tuple_data: &[u8]) -> Result<u16> {
    let tuple_len = tuple_data.len();
    let required_space = tuple_len + SLOT_SIZE;

    // 空き領域が足りるかチェック
    if self.free_space() < required_space {
        bail!("not enough space in page");
    }

    // Tupleデータを後ろから書き込む
    let new_offset = self.free_space_offset() - tuple_len as u16;
    self.data[new_offset as usize..new_offset as usize + tuple_len]
        .copy_from_slice(tuple_data);

    // Slotを追加
    let slot_id = self.tuple_count();
    self.set_slot(slot_id, new_offset, tuple_len as u16);
    self.set_tuple_count(slot_id + 1);
    self.set_free_space_offset(new_offset);

    Ok(slot_id)
}
```

### Get

Tupleの取得はSlotを参照するだけです。

```rust
pub fn get_tuple(&self, slot_id: u16) -> Option<&[u8]> {
    if slot_id >= self.tuple_count() {
        return None;
    }
    // Slotからoffsetとlengthを取得してTupleを返す
    let (offset, length) = self.get_slot(slot_id);
    Some(&self.data[offset as usize..(offset + length) as usize])
}
```

Slot IDさえわかれば、O(1)でTupleにアクセスできます。

### DiskManager

Pageの読み書きを担当します。`page_id * PAGE_SIZE`の位置にシークしてPageを読み書きします。

```rust
pub fn read_page(&mut self, page_id: u32, buf: &mut [u8; PAGE_SIZE]) -> Result<()> {
    let offset = (page_id as u64) * (PAGE_SIZE as u64);
    self.file.seek(SeekFrom::Start(offset))?;
    self.file.read_exact(buf)?;
    Ok(())
}
```

### Table

Tableは複数のPageを管理し、挿入時に空きがなければ新しいPageを割り当てます。

```rust
pub fn insert(&mut self, values: &[Value]) -> Result<(u32, u16)> {
    let tuple_data = serialize_tuple(values);

    // 最後のPageに空きがあれば挿入
    let page_count = self.disk_manager.page_count();
    if page_count > 0 {
        let last_page_id = page_count - 1;
        let mut buf = [0u8; PAGE_SIZE];
        self.disk_manager.read_page(last_page_id, &mut buf)?;
        let mut page = Page::from_bytes(&buf);
        if let Ok(slot_id) = page.insert(&tuple_data) {
            self.disk_manager.write_page(last_page_id, &page.data)?;
            return Ok((last_page_id, slot_id));
        }
    }

    // 空きがなければ新しいPageを割り当て
    let page_id = self.disk_manager.allocate_page()?;
    let mut page = Page::new(page_id);
    let slot_id = page.insert(&tuple_data)?;
    self.disk_manager.write_page(page_id, &page.data)?;
    Ok((page_id, slot_id))
}
```

## 動作確認

Pageが複数作られるように複数のTupleを挿入してみます。

```rust
let loc1 = table.insert(&[Value::Int(1), Value::Varchar("Alice".to_string())])?;
let loc2 = table.insert(&[Value::Int(2), Value::Varchar("Bob".to_string())])?;
let loc3 = table.insert(&[Value::Int(3), Value::Varchar("Charlie".to_string())])?;
let loc4 = table.insert(&[Value::Int(4), Value::Varchar("Dave".to_string())])?;
let loc5 = table.insert(&[Value::Int(5), Value::Varchar("Eve".to_string())])?;

println!("Inserted at: {loc1:?}, {loc2:?}, {loc3:?}, {loc4:?}, {loc5:?}");
println!("Total pages: {}", table.page_count());
```

実行結果（stdout）:

```
Inserted at: (0, 0), (0, 1), (0, 2), (1, 0), (1, 1)
Total pages: 2
Scanned 5 tuples:
  [Int(1), Varchar("Alice")]
  [Int(2), Varchar("Bob")]
  [Int(3), Varchar("Charlie")]
  [Int(4), Varchar("Dave")]
  [Int(5), Varchar("Eve")]
Page 0: tuple_count=3, free_space_offset=22, free_space=2
Page 1: tuple_count=2, free_space_offset=39, free_space=23
```

64バイトのPageに3つのTupleが入り、4つ目以降は新しいPageが作成されています。

## 今日の実装の限界

現在の実装ではPageを操作するたびに毎回ディスクから読み込んでいます。同じPageに複数回アクセスする場合でも、その都度ディスクI/Oが発生してしまいます。ディスクI/Oはメモリアクセスに比べて非常に遅いため、頻繁にアクセスするPageをメモリにキャッシュしたいところです。一方で、全Pageをメモリに載せるとデータ量が増えたときにメモリが溢れてしまいます。限られたメモリ内でどのPageをキャッシュするか管理する仕組みが必要になります。

ちなみに、削除や更新も実装していませんが、Slotのlengthを0にすることで削除、削除＋挿入で更新、読み取り時に長さ0のSlotを読み飛ばすようにすれば実装できるので、興味があれば試してみてください。

## 次回予告

明日は**Buffer Pool Manager**を実装し、上記のメモリ管理の問題を解決します。
