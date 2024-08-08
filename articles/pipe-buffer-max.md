---
title: "パイプのバッファーサイズの罠"
emoji: "🐃"
type: "tech"
topics: ["unix", "linux", "pipe", "ruby"]
published: true
publication_name: "primenumber"
---

## はじめに

あるプログラムから別のプログラムを実行し、結果を受け取りながら処理を進めたい場面が時折生じます。
しかし、パイプの挙動への注意を怠ると思わぬ問題を引き起こす可能性があります。

## 実行環境

本記事は、以下の環境にて実行しています。

- Ruby: 3.0.6
- Ubuntu: 20.04 LTS
- Linux kernel: 5.15.0-71-generic

以降のプログラムは筆者の普段使いの Ruby で記述していますが、本質的には言語によらない問題です。
また、本記事で取り上げているパイプの概念は Unix 系 OS に広く共通するものですが、具体的な説明としては Linux を中心に話を進めます。

## サンプルプログラム

### 子プログラム

実行に一定時間を要し、その途中で標準出力に結果を書き込むプログラムです。
単純化のため、1 秒の固定頻度で標準出力に書き込むだけですが、現実世界では外部への副作用を伴いながら都度ログを出力しているようなプログラムをイメージしていただければと思います。

```ruby:child.rb
60.times do |i|
  puts "[child: #{Time.now}] #{'a' * 1000}"
  sleep 1
end
```

### 親プログラム

子プログラムを実行し、数秒おきに子の標準出力を読み取って利用するプログラムです。
単純化のため、子の標準出力+α を出力しているだけですが、現実世界では DB へのアクセスなどが生じると仮定して、負荷を考慮して数秒おきのループとしています。

```ruby:parent.rb
require 'open3'

Open3.popen3('ruby child.rb') do |stdin, stdout, stderr, wait_thr|
  stdin.close

  loop do
    line = stdout.gets
    break if line.nil?

    puts "[parent: #{Time.now}] #{line.slice(0, 100)}..."
    sleep 5
  end
end
```

### 実行結果

親プログラムを実行すると、5 秒ごとに 1 行ずつ出力されます。
行の先頭には親の出力時刻と、子が自身の標準出力に書き込みをした時刻が表示されます。
親は 5 秒ごと、子は 1 秒ごとにループ処理を行っているため、結果はおおむね予想通りと思います。

```log
$ ruby parent.rb
[parent: 2023-07-31 22:10:02 +0900] [child: 2023-07-31 22:09:55 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:10:07 +0900] [child: 2023-07-31 22:09:56 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:10:12 +0900] [child: 2023-07-31 22:09:57 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:10:17 +0900] [child: 2023-07-31 22:09:58 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:10:22 +0900] [child: 2023-07-31 22:09:59 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
```

### 子のプログラムの変更

次に、子が標準出力に書き込む文字数を増やしてみます。
先ほどは 1 行が約 1,000 Byte でしたが、約 40,000 Byte に増やしました。

```ruby:child.rb
60.times do |_i|
  puts "[child: #{Time.now}] #{'a' * 40_000}"
  sleep 1
end
```

これを実行すると、途中から子の時刻が 5 秒ほど開くようになりました。
本来ならば子の 1 ループ処理は 1 秒ごとに行われるはずです。
書き込み量を増やしただけですが、どうやら親のループ処理時間に影響を受けているように見えます。

```log
[parent: 2023-07-31 22:12:49 +0900] [child: 2023-07-31 22:12:48 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:12:54 +0900] [child: 2023-07-31 22:12:49 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:12:59 +0900] [child: 2023-07-31 22:12:50 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:13:04 +0900] [child: 2023-07-31 22:12:55 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:13:09 +0900] [child: 2023-07-31 22:13:00 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:13:14 +0900] [child: 2023-07-31 22:13:05 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
[parent: 2023-07-31 22:13:19 +0900] [child: 2023-07-31 22:13:10 +0900] aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...
```

## 原因

結論から言うと、表題の通りパイプのバッファーサイズが影響しています。

### パイプ について

パイプとは、プロセス間のデータのやりとりを行うための Unix 系 OS の機能の一つです。
プロセス間通信（IPC）と呼ばれ、パイプ以外にもいくつか形態が存在しますが[^1]、最も一般的に利用されると思われるものがパイプになります。
以下のように、シェル上で`|`を使ってコマンドを連結することは日常的でも行う操作の一つかと思います。

```shell
$ ps aux | grep <process_name>
```

シェルはこのような入力を受け取ると、`pipe(2)`システムコール[^2]を呼び出し、ファイルディスクリプタ[^3]（以下 fd）を 2 つ得ます。

> #include <unistd.h>
>
> int pipe(int pipefd[2]);

> The array pipefd is used to return two file descriptors referring to the ends of the pipe.
> pipefd[0] refers to the read end of the pipe. pipefd[1] refers to the write end of the pipe.

[pipe(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pipe.2.html)

fd は 1 つのパイプにつき 2 つ作成されて、それぞれ読み/書き専用という意味では少し特殊ですが、通常ファイルやソケットと同様に`read(2)`や`write(2)`システムコールを通して読み書きを行うことができます。
単一のプロセス内では使い道はないですが、通常は`pipe(2)`を実行後に`fork(2)`を実行して子プロセスとパイプを共有します。

### 容量

Linux では、パイプ のバッファーサイズはデフォルトで65,536 Byte です。
バッファーにデータが溜まったまま、容量を超えて書き込みを行おうとすると、処理がブロッキングします。
もちろん、読み取ることでデータは消費されていくため、適切に読み取り処理が行われていれば問題になりません。

> A pipe has a limited capacity. If the pipe is full, then a write(2) will block or fail, depending on whether the O_NONBLOCK flag is set (see below).

> Since Linux 2.6.11, the pipe capacity is 16 pages (i.e., 65,536 bytes in a system with a page size of 4096 bytes).

[pipe(7) — Linux manual page](https://man7.org/linux/man-pages/man7/pipe.7.html)

### 根本原因

Ruby の`Open3.popen3`を実行すると、パイプが 3 つ作成されます[^4]。
その後、引数で渡したコマンドが実行され、親子プロセスの双方で 3 つのパイプの fd に対して必要な処理を行います。

|                        | 親・読み取り fd            | 親・書き込み fd            | 子・読み取り fd        | 子・書き込み fd              |
| ---------------------- | -------------------------- | -------------------------- | ---------------------- | ---------------------------- |
| 標準入力用パイプ       | クローズ                   | `popen3`ブロックの第一引数 | 子の標準入力に紐づける | クローズ                     |
| 標準出力用パイプ       | `popen3`ブロックの第二引数 | クローズ                   | クローズ               | 子の標準出力に紐づける       |
| 標準エラー出力用パイプ | `popen3`ブロックの第三引数 | クローズ                   | クローズ               | 子の標準エラー出力に紐づける |

不要な fd はクローズします。例えば、子への入力のために作成したパイプの読み取り fd は親としては不要です。
子プロセス側では、`dup2(2)`システムコール[^5]で、3 つのパイプの fd を標準入力、標準出力、標準エラー出力の fd と紐づけます。
親プロセス側ではそれぞれの fd を読み書きできる IO クラスを`Open3.popen3`のブロック内から利用できるように返します。

子では、1 秒ごとにおおよそ 40,000 Byte の書き込みをしているため、読み取りが行われないまま 3 回以上書き込みを行おうとしたタイミングで、上限の 65,536 Byte を超えてしまいます。
親は 5 秒経過しないと読み取りをしないため、それまでは子の処理もブロッキングしてしまうことになります。
結果として、単体で子のプログラムだけを実行すれば約 60 秒で終わるはずですが、一見すると読み取りだけをしていて副作用がなさそうに見える親のロジックが影響し、子のプログラムは約 5 倍の実行時間がかかってしまっていました。

## 最悪のケース

前述の例では、親子のプロセスはいずれも時間が経てば終了はします。
しかし、親プログラムを次のように変更すると状況が変わります。
以下は子の標準出力からは一切読み取らずに、単に終了ステータスのみを確認するだけのプログラムです。

```ruby:child.rb
require 'open3'

Open3.popen3('ruby child.rb') do |stdin, stdout, stderr, wait_thr|
  stdin.close

  status = wait_thr.value.exitstatus
  puts status
end
```

このプログラムを実行すると、次の理由で親子のプロセスがデッドロック状態になり、永遠に終了しなくなってしまいます。

- 親プログラムでは、`wait_thr.value.exitstatus`にて、子プロセスが終了するまでブロックされます。
- 子プログラムでは、前述の理由によりパイプからの読み取りが行われない限り、標準出力への書き込み部分にてブロックされます。

上記の処理内容では、わざわざ`Open3.popen3`を使って パイプ を生成する必要はないのですが、
既存箇所からコピペしてきた上で必要のない読み取り処理の行を削った、といったようなケースではこのようなコードが生まれてしまうかもしれません。

## まとめ

単発で実行する場合に比べ、別のプログラムから実行することでスループットが悪化してしまったり、最悪処理が進まなくなってしまうケースがありました。
C 言語などのシステムコールを直接意識する言語でない場合、パイプのバッファーサイズという原因に辿り着くのに時間を要してしまうこともあるかもしれません。
同様の事象に遭遇した方の何かしらのヒントになれば幸いです。

## 参考文献

- [詳解 UNIX プログラミング 第 3 版](https://www.shoeisha.co.jp/book/detail/9784798135021)

[^1]: 名前付きパイプや Unix ドメインソケット

[^2]: プログラムがカーネルの機能を呼び出すための API

[^3]: オープンしているファイルを参照する非負の整数。ネットワーク間通信に用いられるソケットなど、通常ファイル以外でも抽象化され用いられる。

[^4]: https://github.com/ruby/open3/blob/c5a7dde80608e724b17647f7f61abec5d2dff50f/lib/open3.rb#L93-L101

[^5]: fd を複製するシステムコール
