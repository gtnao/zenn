---
title: "Embulkのcoreのソースコードから紐解くデータ転送のしくみ"
emoji: "🐋"
type: "tech"
topics: ["embulk", "java"]
published: false
---

この記事は [trocco Advent Calendar 2023](https://qiita.com/advent-calendar/2023/trocco) の6日目の記事となります。

https://qiita.com/advent-calendar/2023/trocco

# はじめに

今回はtroccoの内部でも利用されているETLのためのOSSであるEmbulkについて、core部分のソースコードリーディングを通して、そのしくみを紐解いていきたいと思います。

## おことわり

- Embulkの基本的な使い方などについては解説しません。
- 筆者はembulk-coreにコントリビュートしているわけではないので、間違いなどがあればお気軽にご指摘ください。
- 今回見ていくcoreの実装自体は、比較的変更が少ないとされる各種プラグインが従うべきインターフェース部分(embulk-spi)から隠蔽されているため、今後この記事の内容が正しくなくなる可能性は容易にあります。
- Embulkにはguessやpreviewやresumeといった機能も含まれていますが、今回は単純な`embulk run`で実行される処理に絞ってお話しします。

# 前提

## 参照バージョン

embulk-coreの参照するバージョンは、2023/12/06の時点での最新であるv0.11.2とします。

https://github.com/embulk/embulk/tree/v0.11.2

後述するembulk-spiは以下のv0.11を参照します。

https://github.com/embulk/embulk-spi/tree/v0.11

## embulk-spi

コードを見ていく前に、いくつかの前提にも触れておきます。

各プラグインは、embulk-spiで定められたインターフェースに従ってembulk-coreと連携しています。
v0.9以前ではembulk-coreの`org/embulk/spi`パッケージ以下に配置されていましたが、インターフェース以外の各種ユーティリティクラスもこの中には存在しています。
v0.11以降ではそういった依存性を減らすために、インターフェース定義を別リポジトリに切り出しています。
コードを追っていく中で、embulk-coreの`org/embulk/spi`の中には見つからないインターフェースがあったらこちらのリポジトリを参照するとよいです。

https://github.com/embulk/embulk-spi

また、この辺りの詳しい話はcoreのメンテナーをしているdmikurubeさんの記事に詳しくまとまっています。

https://zenn.dev/dmikurube/articles/get-ready-for-embulk-v0-11-and-v1-0#%E4%BE%9D%E5%AD%98%E9%96%A2%E4%BF%82%E3%81%AE%E6%95%B4%E7%90%86

## プラグイン種別

こちらは、Embulkを利用していれば一般的な内容かもしれませんが、プラグインには以下のいくつかの種類が存在します。

- InputPlugin: 非ファイル系のデータ元(MySQLなど)
- FileInputPlugin: ファイル系のデータ元(S3など)
- OutputPlugin: 非ファイル系のデータ先(BigQueryなど)
- FileOutputPlugin: ファイル系のデータ先(S3など)
- FilterPlugin: データ元からデータ先への過程での変換処理。ETLのTの部分
- ParserPlugin: データ元ファイルのパース(csvなど)
- DecoderPlugin: データ元ファイルの解凍(gzipなど)
- FormatterPlugin: データ先ファイルのフォーマット(csvなど)
- EncoderPlugin: データ先ファイルの圧縮(gzipなど)
- (ExecutorPlugin): 今回は1つのインスタンス上での実行を想定したLocalExecutorを前提とする
- (GuessPlugin): guess機能で利用。今回は割愛

同名のインターフェースもembulk-spiに定義されており、プラグイン側はそれを満たすように実装する必要があります。

各プラグインの対応関係は、大まかに以下となります。
※ 厳密な実行時の流れというよりはイメージ図です。

![](https://storage.googleapis.com/zenn-user-upload/f89bf8204833-20231206.png)

FileInputPluginはデータ元の括りですが、embulk-core側の`FileInputRunner`という`InputPlugin`を実装したクラスから利用されます。
FileInputPluginが実装を要求するメソッドはInputPluginとは異なる点も注目です。
(FileOutputPluginも同様)

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/spi/FileInputRunner.java#L21-L25

また、FilterPlugin ,DecoderPlugin ,EncoderPluginは複数指定することが可能です。

# コードリーディング

前置きが長くなりましたが、それではembulk-coreのソースコードを読んでいきます。

## エントリーポイント

embulkコマンドを実行した際のエントリーポイントは以下のクラスのmainメソッドです。

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/cli/Main.java#L16

さまざまな初期化処理が行われますが、長いのでだいぶ飛ばします。
`embulk run`が実行されると、最終的に以下の`BulkLoader`クラスの`doRun` メソッドに到達します。
ここから読み進めていきます。

https://github.com/embulk/embulk/blob/6280200ffc502ebad440b0748a947937664d85d4/embulk-core/src/main/java/org/embulk/exec/BulkLoader.java#L512-L582

## transaction

まずは前述の`doRun`ですが、ここでは各プラグイン側で実装された`transaction`メソッドが順番に呼ばれていきます。
かなりネストしていて、core側とプラグイン側の処理がいり混じっている部分なので初見だと流れをイメージしづらいかもしれません。
整理すると、以下のシーケンス図のようになっています。細かくなってしまったので拡大して見てみてください。

![](https://storage.googleapis.com/zenn-user-upload/687c176a0a53-20231205.png)

core側に実装された`~.Control`が各プラグインの橋渡しをしているイメージです。
一番右の`ExecutorPlugin.Executor.execute`からは実際の転送処理の実行に移ります。
そのため、処理がreturnしてくるのは転送が終了したタイミングになります。

ファイル系のプラグインを利用しているともう少し複雑です。
例えばinput部分であれば、前述の`FileInputRunner`の`transaction`が呼ばれ、その後その中で定義されている`RunnerControl.run`がDecoderPlugin,ParserPluginの`transaction`が呼びだしていく構造になっていたりします。

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/spi/FileInputRunner.java#L102-L118

また、FilterPluginの様に複数プラグインを設定できる場合は、core側の`FilterInternal.transaction`が呼ばれ、各FilterPluginの`transaction`を順に呼び出す構造になっています。(DecoderPluginやEncoderPluginも同様)

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/spi/util/FiltersInternal.java#L57-L89

では、そもそも`transaction`では何が行われているのでしょうか。
プラグイン側の実装次第ではありますが、一言で言うとメインの転送処理に対する前処理や後処理を行なっています。
特に前述のシーケンス図の右矢印の過程(前処理)においては、各プラグインで事前に算出しておくべき値などが`~Control`のインターフェース定義を見ると分かります。
`transaction`の中では前処理を終えると`~Control`のメソッドを呼び出すからです。

https://github.com/embulk/embulk-spi/blob/576e98033a14ba8ac994ed581d3c9d8fcdda2749/src/main/java/org/embulk/spi/InputPlugin.java#L52

InputPluginでは、`Schema`と`int taskCount`を渡して`~Control.run`を呼び出しています。
データ元の設定によって、最初に入ってくるデータのスキーマが決定されるので、これを事前に算出して後続へ渡しています。
また、ファイル系のプラグインでは、ファイル数に応じた並列処理がこの後に行われていくため、事前にファイル数(`int taskCount`)を求めておきます。

https://github.com/embulk/embulk-spi/blob/576e98033a14ba8ac994ed581d3c9d8fcdda2749/src/main/java/org/embulk/spi/FilterPlugin.java#L49

FilterPluginでも`Schema`を算出しています。
これはInputPluginから渡されたスキーマがETLのTであるFilterPluginを通すことで変化していく可能性があるからです。

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/spi/ExecutorPlugin.java#L11

ExecutorPluginでは新たに`int outputTaskCount`という概念が出てきました。
詳しくは後述しますが、後続の並列処理ではInputPluginで求めた`int taskCount`に加えて、ホストマシンのCPUコア数や明示的な設定値に応じてその並列処理方法が変化します。
そのために必要な`int outputTaskCount`という値をExecutorPluginでは事前に算出しています。

また、`transaction`では`~Control.run`の呼び出し後に処理を書くと全ての転送処理が終了した後の後処理を書くことができます。
例えば、`embulk-output-jdbc`においては、tempテーブルに転送したデータを最後にまとめて対象のテーブルにInsertするような処理をここで行なっています。

https://github.com/embulk/embulk-output-jdbc/blob/d2145e3ac87d839c382ab25a540d450257808c59/embulk-output-jdbc/src/main/java/org/embulk/output/jdbc/AbstractJdbcOutputPlugin.java#L461-L463
https://github.com/embulk/embulk-output-jdbc/blob/d2145e3ac87d839c382ab25a540d450257808c59/embulk-output-jdbc/src/main/java/org/embulk/output/jdbc/AbstractJdbcOutputPlugin.java#L867-L914

`transaction`メソッドが呼び出されていく段階では、処理はまだ直列です。
この後は、シーケンス図の一番右の`ExecutorPlugin.Executor.execute`の処理内容を追っていきます。

## 並列転送処理

前述の`ExecutorPlugin.Executor.execute`以降の処理ですが、
`Executor`は、`DirectExecutor`,`ScatterExecutor`のどちらかということが`transaction`の段階で決定されています。

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/exec/LocalExecutorPlugin.java#L58-L70

`DirectExecutor`,`ScatterExecutor`に関しては、以下のsonotsさんの記事で詳しく解説されています。
対象バージョンはv0.8.6ですが、大まかには変わっていないと思いますので、詳細はこちらの記事を参照してください。

https://qiita.com/sonots/items/4445467ddb5cea0622a5

まずは、シンプルな方の`DirectExecutor`で読み進めると分かりやすいと思います。
以降は、JavaのFutureによってスレッドごとの処理に分岐してきますが、それぞれのThreadは以下の処理に到達します。

https://github.com/embulk/embulk/blob/6280200ffc502ebad440b0748a947937664d85d4/embulk-core/src/main/java/org/embulk/spi/util/ExecutorsInternal.java#L44-L75

ここが転送処理の始まりの部分です。

### PageOutput

最初にOutputPluginに実装された`open`というメソッドが呼ばれ、その返り値である`PageOutput`というインターフェースを実装したインスタンスを取得します。
次にFilterPluginの`open`が前段の`PageOutput`を受け取りまた新たな`PageOutput`を返します。
面白いことに、ここはデータの処理と流れが逆な点に注目です。
`PageOutput`を実装したクラスはそれぞれ`add`というメソッドがあり、この処理はそれぞれで異なります。
データの処理と逆の流れでリレーしていったことで、最終的にはデータの処理の順に`PageOutput`がネストされたオブジェクトが得られます。

![](https://storage.googleapis.com/zenn-user-upload/0f56a45fc06d-20231206.png)

そして、最終的な`PageOutput`は、InputPluginの`run`メソッドに渡されます。

https://github.com/embulk/embulk/blob/6280200ffc502ebad440b0748a947937664d85d4/embulk-core/src/main/java/org/embulk/spi/util/ExecutorsInternal.java#L60

`run`メソッドでは各InputPluginが思いのままにデータ元からデータを取得します。
その取得したデータは`Page`というクラスのインスタンスに格納されます。
生成した`Page`は後続のプラグインへ`PageOutput`の`add`メソッドを介して受け渡されていきます。
`PageOutput`を先に生成していたのはここに意味があるわけです。
各`PageOutput`は後続の`PageOutput`を内包しているため、各プラグインにおける処理を終えると、`Page`を生成して後続の`PageOutput`に`add`していきます。
最終的にはOutputPluginに到達して、その`add`メソッドはデータロードの処理を実行します。

流れは以下のシーケンス図のようになります。(FilterPluginが一つの場合)

![](https://storage.googleapis.com/zenn-user-upload/fba568a658ab-20231206.png)

つまり、まずデータ元のデータを全件取得して後続に渡して処理していくような逐次的処理ではなく、データの取得中(ETLのE)にも順次後続の処理(ETLのTL)を行っていくような準ストリーミングな方式の処理となっています。
そのため大規模なデータにおいてもメモリ消費を一定に抑えることができ、更には並列処理による速度向上も見込めるアーキテクチャとなっているわけです。

## Page

それでは`Page`とはなんでしょうか。
`Page`はそのフィールドとして、`buffer`と`stringReferences`, `jsonValueReferences`を持っています。

https://github.com/embulk/embulk/blob/749a0597571c26e71bf0cc1ec07578a3c421d406/embulk-core/src/main/java/org/embulk/spi/PageImpl.java#L29-L31

`buffer`は32KBのバイト列で以下のようにレイアウトとなっています。

![](https://storage.googleapis.com/zenn-user-upload/f38ed103c702-20231206.png)

`Page`という名称で行ごとのデータをシリアライズしたり、NULLのbitmapを行ヘッダーに持っているような構造は、どこかRDBMSにおけるページの概念の思い起こします。
とはいえ、string, json型のデータは可長幅のデータ型が故に、実データは`stringReferences`, `jsonValueReferences`に保持した上でその配列のindex値が`buffer`に格納されているようでした。
この辺のWhy?は歴史的経緯も絡んでくると思いますが深くは調べきれていません。

そして、上記のような`Page`の内部構造はプラグイン側に対しては隠蔽されているため、`PageBuilder`, `PageReader`というクラスを利用して読み書きをしていることがほとんどです。

### PageBuilder

`PageBuilder`を利用した`Page`の書き込みは、プラグイン側では例として以下のように実装できます。
特に厳密な制約はないですが、Visitorパターンを使ってEmbulkにおける各型(long, double, boolean, string, timestamp, json)に対する処理を書いているプラグインが多いと思います。

```java
while ( /* 全体ループ */ ) {
  while ( /* 行ループ */ ) {
    schema.visitColumns(new ColumnVisitor() {
      public void longColumn(Column column) {
        boolean v = /* 値を取得 */
        pageBuilder.setBoolean(column, v);
      }
      public void stringColumn(Column column) {
        String v = /* 値を取得 */
        pageBuilder.setString(column, v);
      }
      /* その他の型、double, boolean, timestamp, jsonについても実装 */
    })
  }
  pageBuilder.addRecord();
}
pageBuilder.finish();
```

カラムごとに値を`set~`していき、`addRecord`を呼び出すと1レコードとして記録されます。
bufferに収まらなくなったタイミングで、`PageBuilder.flush`が呼び出され後続の`PageOutput`へ`add`されます。

https://github.com/embulk/embulk/blob/v0.11.2/embulk-core/src/main/java/org/embulk/spi/PageBuilderImpl.java#L226-L246

### PageReader

一方で、読み取り側では`PageReader`を利用します。
こちらも概ね同じようにVisitorパターンで実装しているプラグインが多いと思います。

```java
while (pageReader.nextRecord()) {
  pageReader.getSchema().visitColumns(new ColumnVisitor() {
    public void booleanColumn(Column column) {
      /* FilterPluginであればPageBuilderのset~を呼び出す */
      /* OutputPluginであればLoadのための各種処理 */
    }
    /* その他の型についても実装 */
  })
}
```

ここで一点着目したいのは、FilterPluginにおいては`PageReader`によって`Page`が読み出されると然るべく処理を行った後に`PageBuilder`を利用して`Page`を生成して後続に渡している点です。
InputPlugin -> OutputPluginの流れの中で同一の一行であれば同一の`Page`と言うように一対一に紐づいているわけではなく、
以下のように`Page`は各プラグイン間においてのデータ受け渡しのたびに生成されます。

![](https://storage.googleapis.com/zenn-user-upload/5e437693bbe0-20231206.png)

# おわりに
