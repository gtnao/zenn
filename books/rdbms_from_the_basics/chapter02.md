---
title: "ページの導入"
---

# 課題

前章では、読み書きするたびにオフセットを移動していました。
ただでさえ、ディスクアクセスはメモリアクセスに比べて遅いです。
更に、ディスクアクセスにおいて、ランダムアクセスはシーケンシャルアクセスに比べて遅いです。
では、メモリに読み書きすれば良いのかというと、メモリは揮発性があるので、書き込んだデータはそのままだと永続化できません。

# 方針

そこで、以下の3点を全て考慮した実装ができると良さそうです。

- できる限りメモリへ読み書きして、ディスクへのアクセスを減らす
- ディスクへのアクセスが必要な際にも、まとまった単位で行いシーケンシャルアクセスを増やす
- メモリへの書き込みの際は、データが消失しないようにする
  - ※ 前章で述べた通り、今回はディスク上のファイルに書き込みが成功した時点で永続化されたとみなします。

## ページ

本章ではまとまった単位でのアクセスを実装します。

一般的なRDBMSでは、固定長の論理的にまとまった「ページ」という概念を用います。
同一ファイルでも、ここからxxKBはページ1、ここからxxKBはページ2、と解釈するといった具合です。

# 実装

`Page`というstructを用意し、ひとまとまりのバイト列を管理します。
今回は検証のため、1ページ16バイトと極端に小さい値にしています。
各種RDBMSごとに異なりますが、実際には数キロバイトです。

```rust
const PAGE_SIZE: usize = 16;

struct Page {
    bytes: [u8; PAGE_SIZE],
}
```

ページのバイト列のレイアウトは以下のように定義します。

:::message
BigEndian/LittleEndianや2の補数を考えなくても済むように、今回は基本的に符号無し1Byteで統一します。
当然256ページ目を追加したらpanicになります。
:::

```text
header
page_id(1Byte), tuple_length(1Byte)

data
1Byte * 14
```

かなり単純化しました。
実際にはもう少し複雑な構造をしていますが、特に規格があるわけではないので、俺の考えた最強のページレイアウトでOKです。

※ tupleというのは、一般的にRDBMSの世界で行を指す概念です。

メソッド群も定義しておきます。

```rust
impl Page {
    const HEADER_SIZE: usize = 2;
    const TUPLE_MAX_LENGTH: u8 = PAGE_SIZE as u8 - Self::HEADER_SIZE as u8;
    fn init(page_index: u8) -> Self {
        let mut bytes = [0; PAGE_SIZE];
        bytes[0] = page_index;
        Self { bytes }
    }
    fn load(bytes: [u8; PAGE_SIZE]) -> Self {
        Self { bytes }
    }
    fn page_index(&self) -> u8 {
        self.bytes[0]
    }
    fn length(&self) -> u8 {
        self.bytes[1]
    }
    fn increment_length(&mut self) {
        self.bytes[1] += 1;
    }
    fn has_space(&self) -> bool {
        self.length() < Self::TUPLE_MAX_LENGTH
    }
    fn read_tuples(&self) -> &[u8] {
        &self.bytes[Self::HEADER_SIZE..(self.length() as usize + Self::HEADER_SIZE)]
    }
    fn insert_tuple(&mut self, tuple: u8) {
        let length = self.length();
        self.bytes[length as usize + Self::HEADER_SIZE] = tuple;
        self.increment_length()
    }
}
```

---

ページへの読み書きは、先ほどの`Database`の読み書きの実装をページ単位に変更し、新たに`PageManager`というstructを定義します。
`init`, `load`は同じなので省略します。

```rust
struct PageManager {
    file: File,
}

impl PageManager {
    fn write_page(&mut self, page: &Page) {
        let index = page.page_index() as u64 * PAGE_SIZE as u64;
        self.file.seek(SeekFrom::Start(index)).unwrap();
        self.file.write_all(&page.bytes).unwrap();
        self.file.flush().unwrap();
    }
    fn read_page(&mut self, page_index: u8) -> Page {
        let index = page_index as u64 * PAGE_SIZE as u64;
        self.file.seek(SeekFrom::Start(index)).unwrap();
        let mut bytes = [0; PAGE_SIZE];
        self.file.read_exact(&mut bytes).unwrap();
        Page::new_with_bytes(bytes)
    }
    fn allocate_page(&mut self) -> u8 {
        let page_index = self.next_page_index();
        let page = Page::new(page_index);
        self.write_page(&page);
        page_index
    }
    fn next_page_index(&self) -> u8 {
        let metadata = self.file.metadata().unwrap();
        (metadata.len() / PAGE_SIZE as u64) as u8
    }
}
```

先ほどは、末尾に値を追加していくだけでしたが、ページを同じ箇所にインプレイスに書き戻すようにします。
どの位置かを決定するために、`page_id`からオフセットを割り出せるように設計します。
`page_id`を0から順にインクリメントされるユニークな値とすることで、`page_id` \* `page_size`をファイルのオフセットと考えて実装できます。
そのため、`write_page`, `read_page`では、事前にその位置まで移動した上で読み書きをしています。
また、ページを追加するための`allocate_page`も定義しておきます。

---

`PageManager`では、ページ一単位での操作を行う責務しか持たせませんでした。
`Database`に、ユーザーから利用するのインターフェースを実装します。

```rust
struct Database {
    page_manager: PageManager,
    last_page_id: u8,
}

impl Database {
    fn init(file_name: &str) -> Self {
        let mut page_manager = PageManager::init(file_name);
        page_manager.allocate_page();
        Self {
            page_manager,
            last_page_id: 0,
        }
    }
    fn load(file_name: &str) -> Self {
        let page_manager = PageManager::load(file_name);
        let last_page_id = page_manager.next_page_index() - 1;
        Self {
            page_manager,
            last_page_id,
        }
    }
    pub fn insert(&mut self, tuple: u8) {
        let page_index = self.last_page_id;
        let mut page = self.page_manager.read_page(page_index);
        if !page.has_space() {
            let next_page_index = self.page_manager.allocate_page();
            self.last_page_id = next_page_index;
            page = self.page_manager.read_page(next_page_index);
        }
        page.insert_tuple(tuple);
        self.page_manager.write_page(&page);
    }
    pub fn read_all(&mut self) -> Vec<u8> {
        let mut values = Vec::new();
        for page_index in 0..=self.last_page_id {
            let page = self.page_manager.read_page(page_index);
            values.extend_from_slice(page.read_tuples());
        }
        values
    }
}
```

`insert`では、最後のページに空きがあるかを確認し、あれば追加、なければ新しいページを確保して追加します。
`read_all`では、最後のページまでループして読み込みます。

---

それでは、先ほどと同じ処理を実行してみます。

```rust
fn main() {
    let mut database = Database::init("db");
    database.insert(0);
    println!("Insert 0");
    database.insert(1);
    println!("Insert 1");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
    for i in 2..100 {
        database.insert(i);
    }
    println!("Insert 2..100");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
    println!("last_page_id: {}", database.last_page_id);

    println!("__________________________");
    println!("Open existing database.");
    let mut database = Database::load("db");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
}
```

出力結果は以下のようになり、ページ単位での読み書きに変えても先ほどと同じことが実現できました。
まだ、これだけでは実装が無駄に増えただけで恩恵がないため、次章にてメモリを使った効率化を実装していきます。

```text
Insert 0
Insert 1
Read all
  values: [0, 1]
Insert 2..100
Read all
  values: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99]
last_page_id: 7
__________________________
Open existing database.
Read all
  values: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99]
```
