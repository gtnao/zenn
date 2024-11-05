---
title: "GolangでPATCH方式の更新APIのリクエストを作成する方法"
emoji: "🌐"
type: "tech"
topics: ["golang", "restapi", "json"]
published: true
publication_name: "primenumber"
---

# はじめに

今回は、更新系のREST APIをGolangで叩く際に、つまずくケースが何点かあったので共有します。

# PATCHメソッドとは

HTTPメソッドの一つであるPATCHは、リソースの一部を更新する用途を意図して定義されたものです。
例えば、リクエストボディのJSONに含まれていないプロパティがあれば、元の値のままといった挙動を想定しています。
更新用途のHTTPメソッドにはPUTも存在しますが、こちらはリクエストでリソースを完全に上書きすることを意図しています。
PATCHに関してはRFC 5789で定義されています。

> ... PATCH, which is used to apply partial modifications to a resource.

> The PUT method is already defined to overwrite a resource with a complete new body

[RFC 5789 - PATCH Method for HTTP](https://datatracker.ietf.org/doc/html/rfc5789)

もちろん、巷のREST APIが全てこの定義に従って実装されているとは限りません。

# Golangによる実装方法

GolangでPATCH方式のリクエストを作成する方法を見ていきます。
今回は、リクエストボディのJSON文字列を作成する部分のみに焦点を当てます。
単純に実装すると以下のようになると思います。

- リクエストパラメータのstructを定義
- jsonのstruct tagを設定し、`json.Marshal`関数でJSON文字列を得る
- struct tagに`omitempty`オプションを指定することで、ゼロ値のフィールドはJSONから除外される

```go
type PatchRequest struct {
	Email string `json:"email,omitempty"`
	Name  string `json:"name,omitempty"`
	Age   int    `json:"age,omitempty"`
}

func sendPatchRequest(req PatchRequest) error {
	jsonData, err := json.Marshal(req)
	if err != nil {
		return err
	}
	// 以下のように続く
	request, err := http.NewRequest(http.MethodPatch, "https://api.example.com", bytes.NewBuffer(jsonData))
```

structの初期化時に指定しないフィールドは自動でゼロ値となります。
例えば、以下のようにしてNameだけを更新するリクエストを送ることができます。

```go
req := PatchRequest{
    Name: "John Doe",
}
sendPatchRequest(req)
```

しかし、これではカバーできないケースが存在します。

# ゼロ値を渡したいケース

Golangのstringやintのゼロ値はそれぞれ、空文字、0です。
そのため、ゼロ値に明示的に更新したいケースは、前述の実装ではまかなえません。

例えば、先の例はユーザーリソースのようなAPIでしたが、以下のようなケースが現実的に考えられます。

- Age(年齢)を誤っていたので、1から0(歳)に更新したいケース
- Name(ハンドルネーム)は任意のため、指定状態から空に更新したいケース

## 解決方法

ポインタ型にすることで解決します。
Golangのポインタ型はゼロ値がnilです。
`json.Marshal`の仕様としても、nilの場合はJSONから省略され、それ以外ではdereferenceした元の値を入れてくれます。

```go
type PatchRequest struct {
	Email *string `json:"email,omitempty"`
	Name  *string `json:"name,omitempty"`
	Age   *int    `json:"age,omitempty"`
}
```

ただし、structの初期化に少し手間がかかるようになります。
Golangでは即値に対して直接ポインタを取ることはできません。
以下のような初期化方法はエラーになります。

```go
req := PatchRequest{
    Email: &"example@example.com",
    Name:  &"John Doe",
    // 関数の戻り値も即値
    Age: &getAge(),
}
```

そのため、一度変数に入れてからポインタを取る、あるいはsetter系の関数を定義しておく、などが必要になります。

```go
email := "example@example.com"
name := "John Doe"
age := getAge()

req := PatchRequest{
    Email: &email,
    Name:  &name,
    Age:   &age,
}

// あるいは
func (p *PatchRequest) SetEmail(email string) {
	p.Email = &email
}

func (p *PatchRequest) SetName(name string) {
	p.Name = &name
}

func (p *PatchRequest) SetAge(age int) {
	p.Age = &age
}

req := PatchRequest{}
req.SetEmail("example@example.com")
req.SetName("John Doe")
req.SetAge(getAge())
```

結構手間です。
しかし、手間の問題だけでなく、まだカバーできないケースが存在します。

# null値を渡したいケース

例えば、Ageが任意項目だった場合、PATCH APIとしては以下の3(4)つのユースケースが考えられます。

- 値を指定して更新する
  - ゼロ値(0)の場合
- 更新しない
- 設定しないことを意味する`null`値を指定して更新する(設定している状態から)

前述の実装では、ポインタフィールドに`nil`を設定するとJSONから省略されてしまうため、後者二つを使い分けることができません。

## 解決方法

カスタム型を定義し、`Marshaler`インターフェースを実装します。

[encoding/json.Marshaler](https://pkg.go.dev/encoding/json#Marshaler)

```go
type NullableInt struct {
	Value int
	Valid bool
}

func (n NullableInt) MarshalJSON() ([]byte, error) {
	if !n.Valid {
		return []byte("null"), nil
	}
	return json.Marshal(n.Value)
}

type PatchRequest struct {
	Email *string      `json:"email,omitempty"`
	Name  *string      `json:"name,omitempty"`
	Age   *NullableInt `json:"age,omitempty"`
}

// 値を指定して更新するケース
age := NullableInt{
    Value: 0,
    Valid: true,
}
req := PatchRequest{
    Age: &age,
}

// 更新しないケース
req := PatchRequest{
    // Ageは指定しないでゼロ値
}

// nullで更新するケース
age := NullableInt{
    Valid: false,
}
req := PatchRequest{
    Age: &age,
}
```

引き続きポインタ型とすることでJSONから省略できるようにしつつ、カスタム型のValidフィールドによって`null`を明示的に指定するかを左右します。

文字列型のフィールドでも`null`を扱いたい場合、同様に`NullableString`を実装する必要があります。
文字列型の場合は、現実的には「設定しないこと = 空文字」としている場合も多いかもしれないので、数値に比べると必要になることは少ないかもしれません。

# おわりに

細かく考えていくと意外に面倒で、GolangのstructとJSONのインピーダンスミスマッチからくるものなのかなと思います。
とはいえ、適当にやるとバグの温床になりやすい部分かなと思うので、異なる技術間での変換処理はぬかりなく考えたいものです。
なお、筆者は普段Golangを主戦場にしているわけではないので、もっとスマートな方法な誤りがあればご指摘ください。
