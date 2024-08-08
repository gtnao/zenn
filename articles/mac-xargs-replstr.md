---
title: "Macのxargsはバイト数制限に注意が必要"
emoji: "🍏"
type: "tech"
topics: ["mac", "unix", "xargs", "command"]
published: true
publication_name: "primenumber"
---

## TL;DR

- `xargs`はGNU版(Linuxなど)とBSD版(Macなど)が存在し挙動に違いがある。
- BSD版の`xargs`では`-I`オプション(文字列置換)を利用する場合にバイト数の制限がある。
- `-I`オプションは置換後の文字列が255バイト以上になった場合に置換されない。
- `-J`オプションを代わりに利用することでバイト数制限の問題は解決するが、`-I`オプションとは異なる挙動もあるので注意が必要。
- MacにGNU版の`xargs`を入れてしまうのも手。

## 実行環境

本記事は、以下の環境にて実行しています。

- macOS Catalina 10.15.7 (Intel)
- GNU bash, version 3.2.57

## ハマった点

スクリプトを書くまでもないけど自動化したい処理を`sed/awk/xargs`などのコマンドを駆使したワンライナーで済ますことは多いと思います。
また、アドホックな処理では各人のローカルマシンで叩くことも多いと思います。
私のローカルマシンはMacなのですが、それが影響して、`xargs`の`-I`オプション(文字列置換)を利用した際に、構文上は同じでも正常に置換がされるケースとされないケースがありました。

```shell
printf x"%.s" {1..254} | xargs -I {} echo {}
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

printf x"%.s" {1..255} | xargs -I {} echo {}
# {}
```

以上の結果だけを見れば文字数に原因があることがお分かり頂けると思います。
しかし、エラーが発生するわけではなく置換がされないだけということや、実際にはもう少し複雑な処理を書いていたということもあり、当初は私の`xargs`の使い方が間違っているのではないかと思い原因の特定まで時間がかかってしまいました。

## 原因

Mac上で`xargs`の`man`を見てみると以下の記述がありました。

> -I replstr
> ...
> The resulting arguments, after replacement is done, will not be allowed to grow beyond 255 bytes;
> ...

置換後の文字列に対して255バイトの制約があるようです。
これは標準入力から渡ってきた文字列ではなく、置換が発生した文字列一つ一つに関する制約のようです。

```shell
printf x"%.s" {1..253} | xargs -I {} echo a{}b a{}
# a{}b axxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

以上のコマンドを実行してみると、
`echo`の第一引数は、前後の文字を含めると合計で255文字(255バイト)に達するため置換が行われません。
一方で第二引数は、合計で254文字(254バイト)のため正常に置換が行われます。

## 解決方法

BSD版では文字列置換の`-J`オプションというものもあるようです。

> -J replstr
> If this option is specified, xargs will use the data read from standard input to replace the first occurrence of replstr instead of appending that data after all other arguments. This option will not affect how many arguments will be read from input (-n), or the size of the command(s) xargs will generate (-s).

こちらのオプションは255バイトの制限がないようで、以下の結果となります。

```shell
printf x"%.s" {1..255} | xargs -J {} echo {}
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### -Jオプションの挙動

ただし、`-J`オプションには`-I`オプションとの間に挙動の違いがありました。

以下のように`a{}b`と置換文字列の前後に空白がない場合は認識されないようです。
`a{}b`がそのまま出力された上で、標準入力の文字列は末尾に展開されて出力されました。
一方で、`-I`オプションでは同様の書き方でも置換文字列が認識され想定通りに置換されました。

```shell
echo foo | xargs -J {} echo a{}b
# a{}b foo

echo foo | xargs -I {} echo a{}b
# afoob
```

また、以下の通り、クォート内の置換文字列は`-J`オプションでは認識されないようです。

> It will not be recognized if, for instance, it is in the middle of a quoted string.

ユースケースとして、標準入力の値を使ってコマンドを構築して実行する場合、以下のような書き方をすることがあると思います。
`-I`オプションでは想定通りの挙動を示すものの、`-J`オプションでは上手くいきませんでした。

```shell
echo foo | xargs -J {} bash -c 'echo {}'
# {}

echo foo | xargs -I {} bash -c 'echo {}'
# foo
```

また、以下の通り、`-J`オプションでは2箇所以上に展開することもできないようです。

> only the first occurrence of the replstr will be replaced.

```shell
echo foo | xargs -J {} echo {} {}
# foo {}

echo foo | xargs -I {} echo {} {}
# foo foo
```

### GNU版xargsをインストールする

以上のように、`-J`オプションを用いても`-I`オプションとの挙動の違いにより解決できないケースがありました。
`homebrew`経由で`findutils`というパッケージをインストールすると、GNU版の`xargs`をMacでも使えるようです。

```shell
brew install findutils
```

`gxargs`というコマンドとしてインストールされるので、`xargs`でaliasを貼っておいてもいいかもしれません。

以下、`-J`オプションではできなかったことを実行してみました。

```shell
# -Iオプションで255バイト以上を置換してみる
printf x"%.s" {1..255} | gxargs -I {} echo {}
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 前後に空白がなくても-Iオプションなので置換される
printf x"%.s" {1..255} | gxargs -I {} echo a{}b
# axxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxb

# クォート内でも-Iオプションなので置換される
printf x"%.s" {1..255} | gxargs -I {} bash -c 'echo {}'
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 2箇所以上の置換もIオプションなので全て置換される
printf x"%.s" {1..255} | gxargs -I {} echo {} {}
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## まとめ

調べてみると他にもGNU版とBSD版との挙動の違いはあるようなので、GNU版を入れてしまうのが手っ取り早い気がしました。
しかし、実際に問題にぶつかってみないことには、デフォルトのコマンドをそのまま使っている方も多いと思います。
当該記事が同じような境遇の方のヒントになれば幸いです。
