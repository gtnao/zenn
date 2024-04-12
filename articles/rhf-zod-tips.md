---
title: "実録 React Hook Form x Zodによるフォームリプレイス"
emoji: "⚛️"
type: "tech"
topics: ["react", "typescript", "reacthookform", "zod"]
published: false
---

# はじめに

直近半年くらい、React Hook FormとZodの組み合わせで既存のフォームをリプレイスする作業に取り組んでいました。
今回は、その過程で溜まってきた個人的なTipsをざっくばらんに共有できればと思います。
同じような記事は巷に沢山ありますが、実プロダクトをデグレードなく移行するには詰まりどころも多く、適当な記事がヒットしないことも多かったため、同じような境遇の方のヒントとなれば幸いです。

## useFormのラップ

まず、以下の記事を大変参考させていただきました。

https://zenn.dev/yuitosato/articles/292f13816993ef

こちらはZodではなくyupなのですが、React Hook Formに関する部分で以下を取り入れています。

- `useForm`をラップして、`defaultValues`をタイプセーフにする
- 既存のコンポーネントをラップした`~Control`というReact Hook Form依存のコンポーネントを用意する

詳しくはリンク先の記事をご覧いただければと思いますが、特に`~Control`のprops定義にジェネリクスを使っていることにより、呼び出し元でname属性をタイプセーフにできる点が大きいです。

```tsx
// 定義元
type Props<T> = {
  name: FieldPath<T>;
  control: Control<T>;
};

// 呼び出し元
// controlにZodのschema情報が入っているので、nameの文字列のパスが一致していないと型エラーとなる
<InputControl name="hoge.fuga" control={control} />;
```

以下ではできる限りこの恩恵に預かるための工夫が出てきます。

## Zodのアトミックなスキーマ定義を1箇所で管理

Zodのインターフェースは、メソッドチェーンでスキーマを定義していくため初心者にも分かりやすい反面で、特にフロントエンド専任エンジニアがいない環境だったため、論理的に同じ意味合いのフォームでも定義が揺れそうという懸念がありました。
例えば、バリデーションが微妙に異なるなどです。

そこで、それ以上分解できない汎用的な文字列や数値のスキーマを変数で定義しておき、各画面側ではなるだけこちらを利用するようにしました。

```tsx
// 定義元
export const requiredString = z.string().min(1);
export const emptyPermitString = z.string();
export const requiredInt = z.number().int();

// 利用側
const someSchema = z.object({
  name: requiredString,
  description: emptyPermitString,
  age: requiredInt,
});
```

### not nullだが初期値はnull

こちらの実用的な例の一つとしては、初期値としてはnullにしておきたいけど、最終的にはnot nullのバリデーションを行いたいというケースです。
具体的には、`number`型として取り回したい(type='number'のinputや、idのselectなど)けど、初期値は空なのでnullとしたい場合です。
`string`型であれば、空文字を初期値としておいて`min(1)`のバリデーションを入れれば済みますが、`number`だと暗黙的な空を意味する値を定義するのもよくないので、nullを初期値としたいケースがあると思います。
しかしながら、`z.int().nullable()`としてしまうと、バリデーションは効きませんし、`z.int()`だけだと初期値を定義する部分で型エラーになってしまいます。

そこで以下のような書き方をしました。
これも先ほどの型と同様に1箇所に定義をしておきます。

```tsx
export const requiredIntDefaultNull = z
  .int()
  .nullable()
  .transform((value, ctx): number => {
    if (value == null) {
      ctx.addIssue({
        code: "invalid_type",
        expected: "number",
        received: "null",
      });
      return z.NEVER;
    }
    return value;
  });
```

## 条件によって分岐するフォームの書き方

ここからは特に難しかったところです。

あるフィールドの値によって、フォームの構成要素が動的に変化するようなものは現実によくあると思います。
そのようなフォームの実装方法もいくつか参考になるものはあったのですが、上記の`~Control`によるタイプセーフな実装と両立していくには難しい点がいくつかありました。

まず、単純にrootに並べて書くと失敗します。
値によってはrequiredでも、それ以外の場合はフィールドが存在しないため意図せずバリデーションエラーとなってしまうためです。
この場合の解決方法としては、`discriminatedUnion`を使う方法があります。
以下のように列挙的定義することができ、`byType.type="timestamp"`の時だけ`byType.format`がrequiredなことを保証できます。

```tsx
// これは途中の実装です（後述の問題あり）
const someSchema = z.object({
  byType: z.discriminatedUnion('type', [
    z.object({
      type: z.literal('string'),
      value: requiredString,
    }),
    z.object({
      type: z.enum(['timestamp']),
      value: requiredString
      format: requiredString,
    }),
  ]),
})
```

このようなフォームが沢山あったため、命名でいちいち悩まないように、一律で`by<切り替えに関連するフィールド名>`という規則を採用しました。

これで解決したように思えますが、まだいくつか問題が発生します。

- 非直感的な初期化処理が必要になる
- errorsを取り出す際に型エラーになる

例えば上記例で初期値が`byType.type=string`の場合、`defaultValues`を指定する際に以下のようにします。

```tsx
{
  byType: {
    type: "string",
    value: ""
  }
}
```

ですが、実行時に`byType.type`の変更が走ると、`byType.format`が`undefined`になってしまいます。
利用コンポーネント側で、`uswWatch`,`useEffect`を組み合わせて、値の変更に伴い初期化するコードを書いてみましたが、初回変更時のみの処理という点でも複雑ですし、スキーマに関するコードが散らばってしまい非直感的です。

また、validationの結果は`formState.errors`で取れるのですが、上記の`byType.format`が必ず存在するフィールドではないため、`errors.byType.format`が型エラーになってしまいます。

### unknown

上記の問題を解決するために、`discriminatedUnion`の中では含まれるフィールドが共通になるようにします。
一方で片方で不要な値は`unknown()`を使って不定としておきます。

```tsx
const someSchema = z.object({
  byType: z.discriminatedUnion('type', [
    z.object({
      type: z.literal('string'),
      value: requiredString,
      format: z.unknown()
    }),
    z.object({
      type: z.enum(['timestamp']),
      value: requiredString
      format: requiredString,
    }),
  ]),
})
```

初期化部分では以下のように`byType.format`も空文字の初期値を入れておきます。

```tsx
{
  byType: {
    type: "string",
    value: ""
    format: ""
  }
}
```

フィールドは`unknown`ですが存在はするので型エラーは発生しません。
（一方で、`format: 1`のようになんでも入れられちゃうので、この辺は限界な部分な気がします)
同様にerrorsでもうまく行きます。
stringの場合はformatは不定なんだなということが初見でもコードから分かりやすいです。

## prefixが汎用的になる場合

既存のフォームをリプレイスする作業をしていたため色々なフォームがあります。
例えば、複数の異なる画面から使われる共通のフォームを切り出したい場合です。
複数の異なる画面なのでZodのスキーマのパスが異なります。

```tsx
const commonSchema = z.object({
  fuga: requiredString
})

const page1Schema = z.object({
  hoge: {
    common: commonSchema;
  }
})

const page2Schema = z.object({
  common: commonSchema;
})
```

`namePrefix`を渡せるコンポーネントを作りたかったのですが、これは困難でした。
無理矢理`useFormContext`のジェネリクスに`[key: string]`を渡すと一応`InputControl`の型チェックは通りましたが、errors部分は無理でした。

```tsx
type DummyFormSchema = {
  [key: string]: { fuga: string }
}

type Props = {
  namePrefix: string
}

export const CommonForm: React.FC<Props> = ({
  namePrefix,
}) => {
  const {
    control,
    formState: { errors },
  } = useFormContext<DummyFormSchema>()
  return (
    <InputControl
      // ここは型エラーにならない
      name={`${namePrefix}.foo`}
      control={control}
      // ここが型エラーになる
      error={errors[namePrefix].foo}
    />
  )
```

この辺は解決できず、何かいい案があれば聞きたいところです。
