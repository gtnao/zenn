---
title: "原始的な実装"
---

まずは、非常に原始的な実装から始めます。
RDBMSと呼ぶにはあまりに簡素ですが、順を追って実装していくことで、各概念、コンポーネント、アルゴリズムがどういった課題から生まれたものかを理解できます。

# 仕様

本質的部分に焦点を当てるため、仕様を非常にシンプルなものに固定します。

- テーブルは1つのみ
- カラムも1つのみ
- カラムの値は符号無し1バイトの整数（0～255）
- データは物理ディスクに書き込まれた時点で永続化されたと仮定する

# 実装

:::message
問答無用で`unwrap()`していきます。
`use`は省略するため、GitHubを参照してください。
:::

`Database`という構造体を用意し、データを読み書きするためのファイルを管理します。
`init`メソッドでは、引数のファイル名が存在しない場合は作成し、存在した場合はtruncateします。

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

次に、ファイルを読み書きするメソッドを用意します。
末尾に1バイトの値を追記していくだけのシンプルな実装になっています。

```rust
    fn write(&mut self, value: u8) {
        self.file.seek(SeekFrom::End(0)).unwrap();
        self.file.write_all(&[value]).unwrap();
        self.file.sync_all().unwrap();
    }
    fn read_all(&mut self) -> Vec<u8> {
        self.file.seek(SeekFrom::Start(0)).unwrap();
        let mut values = Vec::new();
        self.file.read_to_end(&mut values).unwrap();
        values
    }
    fn read(&mut self, index: usize) -> u8 {
        self.file.seek(SeekFrom::Start(index as u64)).unwrap();
        let mut values = [0];
        self.file.read_exact(&mut values).unwrap();
        values[0]
    }
```

それぞれ、読み書き前に適切な位置へとオフセットを移動しています。
Linuxの場合、`lseek(2)`システムコールが呼び出されています。

また、書き込み時には最後に`sync_all`を呼び出しています。
Linuxの場合、`write(2)`システムコールではカーネルのキャッシュにまず格納され、ディスクへの書き込みは保証されていません。
そのため、`fsync(2)`システムコールに相当する`sync_all`を呼び出しています。

---

検証のため、既にファイルが存在した場合に、truncateしない`load`メソッドも用意しておきます。

```rust
    fn load(file_name: &str) -> Self {
        Self {
            file: OpenOptions::new()
                .read(true)
                .write(true)
                .open(file_name)
                .unwrap(),
        }
    }
```

それでは、`main`メソッドから検証してみます。

```rust
fn main() {
    let mut database = Database::init("db");
    database.write(10);
    println!("Write 10");
    database.write(20);
    println!("Write 20");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
    database.write(30);
    println!("Write 30");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
    println!("Read index 1(0-based)");
    let value = database.read(1);
    println!("  value: {:?}", value);

    println!("__________________________");
    println!("Load existing database");
    let mut database = Database::load("db");
    let values = database.read_all();
    println!("Read all");
    println!("  values: {:?}", values);
}
```

以下の結果になりました。
データが永続化されていることも確認できました。

```text
Write 10
Write 20
Read all
  values: [10, 20]
Write 30
Read all
  values: [10, 20, 30]
Read index 1(0-based)
  value: 20
__________________________
Load existing database
Read all
  values: [10, 20, 30]
```
