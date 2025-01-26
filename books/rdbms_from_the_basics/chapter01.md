---
title: "原始的な実装"
---

# 方針

本書ではRDBMSを作ってきますが、「データベースとは」、「DBMSとは」、「RelationalModelとは」、という大きな話からスタートすると本題に入れないため割愛します。

まずは、「テーブルが一つ」、「カラムが一つ」、「カラムの値は1バイトで表現可能な符号なし整数（0 ~ 255）」、を管理できるシステムを作ります。
データは、不揮発性ストレージ(HDD/SSD)上の単一ファイルに書き込み、書き込みが済んだ時点で永続性が担保されていると仮定します。

# 実装

:::message
本書では問答無用で`unwrap()`していきます。
`use`は省略するため、GitHubの方を参照してください。
:::

`Database`というstructを用意し、ファイルを管理します。
`init`メソッドでは、引数で渡したファイル名が存在しない場合は作成し、存在した場合でもtruncateするようにしておきます。

```rust
struct Database {
    file: File,
}

impl Database {
    fn init(file_name: &str) -> Self {
        Self {
            file: OpenOptions::new()
                .read(true)
                .write(true)
                .create(true)
                .truncate(true)
                .open(file_name)
                .unwrap(),
        }
    }
}
```

---

次に、ファイルへの読み書きのメソッドを定義します。
今回はシンプルに、`append`では末尾への1バイトの書き込み、`read_all`では全ての値を読み込みます。

```rust
    fn append(&mut self, value: u8) {
        self.file.seek(SeekFrom::End(0)).unwrap();
        self.file.write_all(&[value]).unwrap();
        self.file.flush().unwrap();
    }
    fn read_all(&mut self) -> Vec<u8> {
        let mut values = Vec::new();
        self.file.seek(SeekFrom::Start(0)).unwrap();
        self.file.read_to_end(&mut values).unwrap();
        values
    }
```

`append`では末尾に、`read_all`では先頭に、毎回オフセットを移動してから処理を行なっています。
これがないと、書き込み直後には末尾にオフセットが移動しているため、読み込みで値を読めなかったりします。

また、書き込みでは最後に`sync_all`を呼び出しています。
Linuxの場合、`write(2)`システムコールではカーネルのキャッシュにまず格納され、ディスクへの書き込みは保証されてはいません。
そのため、`fsync(2)`に相当する`sync_all`を呼び出しています。

---

検証のため、既に渡したファイル名が存在した場合に、truncateせずに読み込む`load`メソッドも実装しておきます。

```rust
    pub fn load(file_name: &str) -> Self {
        Self {
            file: OpenOptions::new()
                .read(true)
                .write(true)
                .open(file_name)
                .unwrap(),
        }
    }
```

それでは、`main`メソッドから利用してみます。

```rust
fn main() {
    let mut database = Database::init("db");
    database.append(10);
    println!("Append 10");
    database.append(20);
    println!("Append 20");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
    database.append(30);
    println!("Append 30");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);

    println!("__________________________");
    println!("Load existing database.");
    let mut database = Database::load("db");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
}
```

結果は以下のようになりました。
`db`ファイルにデータが永続化されていることも検証できました。

```text
Append 10
Append 20
Read all
  values: [10, 20]
Append 30
Read all
  values: [10, 20, 30]
__________________________
Load existing database.
Read all
  values: [10, 20, 30]
```
