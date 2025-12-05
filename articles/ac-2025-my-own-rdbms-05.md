---
title: "Analyzer （一人自作RDBMS Advent Calendar 2025 5日目）"
emoji: "🐘"
type: "tech"
topics: ["database", "rust", "db", "rdbms", "transaction"]
published: true
publication_name: "primenumber"
---

この記事は「[一人自作RDBMS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/my-own-rdbms)」5日目の記事です。

本日の実装は[GitHub](https://github.com/gtnao/advent-calendar-2025-my-own-rdbms/tree/main/day05)にあります。昨日からの差分は以下のコマンドで確認できます。

```bash
git diff --no-index day04 day05
```

## 今日のゴール

**Analyzer**を実装し、Parserが生成したASTに対してテーブル名・カラム名の解決と型情報の付与を行います。

## Analyzerとは

Analyzerは、ASTに含まれる識別子（テーブル名、カラム名、関数名など）をカタログと照合して解決し、型情報を付与するコンポーネントです。

データベースによって呼び方は異なります。PostgreSQLでは「Analyzer」（[analyze.c](https://github.com/postgres/postgres/blob/master/src/backend/parser/analyze.c)）、[DuckDB](https://duckdb.org/docs/stable/internals/overview)では「Binder」と呼ばれています。本記事ではPostgreSQLに倣ってAnalyzerと呼びます。

## なぜAnalyzerが必要か

昨日実装したParserは、SQL文字列をASTに変換します。しかし、このASTには以下の情報が欠けています:

- テーブル`users`は実際に存在するのか？
- カラム`id`はどのテーブルのどのカラムなのか？
- `id > 10`の比較で、`id`はINTか？`10`と比較可能か？

これらを検証・解決するのがAnalyzerの役割です。

```
Parser出力: AST（生の構文木）
   ↓ Analyzer
Analyzer出力: Analyzed AST（識別子が解決され、型情報が付与された構文木）
```

## Catalog（システムカタログ）

Analyzerが識別子を解決するには、テーブルとカラムの情報を持つ**Catalog**が必要です。

PostgreSQLでは、カタログ情報自体もテーブルとして格納されています。`pg_class`にテーブル一覧、`pg_attribute`にカラム情報が入っており、SQLで直接参照できます。

```sql
-- PostgreSQLでテーブル一覧を見る
SELECT relname FROM pg_class WHERE relkind = 'r';
```

しかし、私たちの実装ではまだテーブルをまともに読み書きできる状態にありません。そこで今回は`users(id INT NOT NULL, name VARCHAR)`という固定スキーマをハードコードしています。

## Range Table Entry（RTE）

Analyzerはカラム参照を解決する際、「今どのテーブルがスコープにあるか」を追跡する必要があります。単一テーブルのクエリなら単純ですが、JOINやサブクエリが絡むと複雑になります。

```sql
SELECT u.id, o.order_date FROM users u JOIN orders o ON u.id = o.user_id
```

ここでは`u`と`o`という2つの「テーブルのようなもの」がスコープに入ります。`u.id`を見たとき、`u`がどのテーブルを指すのかを解決する仕組みが必要です。

さらにサブクエリを考えると、状況はより複雑になります。

```sql
SELECT * FROM (SELECT id, name FROM users) AS subquery
```

この`subquery`はカタログに存在しません。クエリ実行時に動的に生成される「仮想的なテーブル」です。しかし、外側のSELECT文から見れば、`subquery`も実テーブル`users`も同じように「カラムを持つもの」として扱いたいはずです。

**Range Table Entry**（RTE）は、これらを統一的に扱うための抽象化です。RTEは「カラムを出力するもの」を表し、その出典（実テーブル、サブクエリ、JOIN結果など）に関わらず、同じインターフェースでカラム解決ができます。

RTEは[PostgreSQLの内部実装](https://www.postgresql.org/docs/current/querytree.html)に由来する概念です。

```rust
#[derive(Debug, Clone)]
pub enum TableSource {
    BaseTable {
        table_id: usize,
        table_name: String,
    },
    // 将来: Subquery { ... }, Join { ... }
}

#[derive(Debug, Clone)]
pub struct OutputColumn {
    pub name: String,
    pub data_type: DataType,
    pub nullable: bool,
}

#[derive(Debug, Clone)]
pub struct RangeTableEntry {
    pub rte_index: usize,
    pub source: TableSource,
    pub output_columns: Vec<OutputColumn>,
}
```

重要なのは`output_columns`です。実テーブルならカタログから取得したカラム定義、サブクエリならそのSELECT結果のスキーマが入ります。カラム参照の解決は常にこの`output_columns`に対して行われるため、RTEの出典が何であるかを意識する必要がありません。

今回は単一の実テーブルのみ対応なので`TableSource::BaseTable`しかありませんが、この設計により将来`Subquery`や`Join`を追加しても、カラム解決のロジックは変更不要です。

## Analyzed AST

Analyzerの結果として生成される型付きASTです。カラム参照は`rte_index`（どのRTEか）と`column_index`（そのRTE内の何番目のカラムか）で表現されます。

```rust
#[derive(Debug, Clone)]
pub struct AnalyzedSelectStatement {
    pub range_table: Vec<RangeTableEntry>,
    pub select_items: Vec<AnalyzedSelectItem>,
    pub from_rte_index: usize,
    pub where_clause: Option<AnalyzedExpr>,
}

#[derive(Debug, Clone)]
pub struct AnalyzedSelectItem {
    pub expr: AnalyzedExpr,
    pub alias: Option<String>,
}

#[derive(Debug, Clone)]
pub struct AnalyzedColumnRef {
    pub rte_index: usize,
    pub column_index: usize,
    pub column_name: String,
    pub data_type: DataType,
}
```

生のASTとの違いは、`Column("id")`が`ColumnRef { rte_index: 0, column_index: 0, data_type: Int, ... }`のように具体的な参照に解決されている点です。

## Analyzer実装

Analyzerは解析中に2つの状態を持ちます。

```rust
pub struct Analyzer<'a> {
    catalog: &'a Catalog,
    range_table: Vec<RangeTableEntry>,  // 出力に含まれる
    scopes: Vec<Scope>,
}
```

`range_table`は解析結果（`AnalyzedSelectStatement`）に含まれる出力です。一方、`scopes`は解析中に「現在どのRTEがカラム解決の対象か」を追跡するための内部状態で、出力には含まれません。

### Scope（名前解決のコンテキスト）

SQLではテーブル名やエイリアスが有効な範囲が構文的に決まっています。Scopeはこの有効範囲を管理します。

```rust
struct ScopeEntry {
    name: String,      // テーブル名またはエイリアス
    rte_index: usize,  // range_table内のインデックス
}

struct Scope {
    entries: Vec<ScopeEntry>,
}
```

さらに、Scopeはスタックとして管理されます。これは相関サブクエリで、内側のクエリから外側のカラムを参照できるようにするためです。

```sql
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)
                                           ^^^^^^^^^^^
                                           外側のスコープを参照
```

内側のサブクエリで`u.id`を解決するには、外側のスコープも見る必要があります。スコープをスタックで管理し、内側から外側に向かって探索することで実現します。

### SELECT文の解析

FROM句のテーブルからRTEを作成し、スコープに登録します。

```rust
let table_ref = &stmt.from;  // TableRef { name, alias }

// RTEを作成してrange_tableに追加
let rte_index = self.add_rte(
    TableSource::BaseTable { table_id, table_name: table_ref.name.clone() },
    output_columns,
);

// スコープにRTEを登録（解析中のみ使用）
self.push_scope();
let scope_name = table_ref.alias.clone().unwrap_or(table_ref.name.clone());
self.current_scope().add_rte(scope_name, rte_index);
```

### カラム参照の解決

カラム名が与えられると、スコープを内側から外側へ探索してRTEを見つけます。

```rust
fn analyze_column(&self, name: &str) -> Result<AnalyzedExpr> {
    for scope in self.scopes.iter().rev() {
        for entry in &scope.entries {
            let rte = &self.range_table[entry.rte_index];
            if let Some(col_idx) = rte.get_column_index(name) {
                let col = &rte.output_columns[col_idx];
                return Ok(AnalyzedExpr::ColumnRef(AnalyzedColumnRef {
                    rte_index: entry.rte_index,
                    column_index: col_idx,
                    column_name: name.to_string(),
                    data_type: col.data_type.clone(),
                }));
            }
        }
    }
    bail!("column '{name}' not found")
}
```

スコープを内側から外側に向かって探索し、各RTEの`output_columns`からカラムを検索します。

### INSERT文の型チェック

INSERT文では、値の数がカラム数と一致するか、各値の型が期待される型と一致するかを検証します。NULL値については、対象カラムがNULL許容かどうかもチェックします。

## 動作確認

正常なクエリとエラーケースをテストします。

```rust
let valid_sqls = vec![
    "SELECT * FROM users",
    "SELECT id, name FROM users",
    "SELECT id FROM users WHERE id > 10",
    "INSERT INTO users VALUES (1, 'Alice')",
    "INSERT INTO users VALUES (2, NULL)",  // name is nullable
];

let error_sqls = vec![
    ("SELECT * FROM unknown_table", "table not found"),
    ("SELECT unknown_col FROM users", "column not found"),
    ("INSERT INTO users VALUES (1)", "wrong number of values"),
    ("INSERT INTO users VALUES ('Alice', 1)", "type mismatch"),
    ("INSERT INTO users VALUES (NULL, 'Alice')", "not nullable"),  // id is NOT NULL
];
```

実行結果（一部抜粋）:

```
SQL: SELECT * FROM users
Analyzed: Select(
    AnalyzedSelectStatement {
        range_table: [
            RangeTableEntry {
                rte_index: 0,
                source: BaseTable { table_id: 0, table_name: "users" },
                output_columns: [
                    OutputColumn { name: "id", data_type: Int, nullable: false },
                    OutputColumn { name: "name", data_type: Varchar, nullable: true },
                ],
            },
        ],
        select_items: [
            AnalyzedSelectItem { expr: ColumnRef(...), alias: None },
            AnalyzedSelectItem { expr: ColumnRef(...), alias: None },
        ],
        from_rte_index: 0,
        where_clause: None,
    },
)

SQL: SELECT id + 1 FROM users
Analyzed: Select(
    AnalyzedSelectStatement {
        ...
        select_items: [
            AnalyzedSelectItem {
                expr: BinaryOp { left: ColumnRef(...), op: Add, right: Literal(1), result_type: Int },
                alias: None,
            },
        ],
        ...
    },
)

=== Error Cases ===

SQL: SELECT * FROM unknown_table
Got: table 'unknown_table' not found

SQL: INSERT INTO users VALUES ('Alice', 1)
Got: type mismatch for column 'id': expected Int, got Varchar
```

`SELECT *`が2つの`AnalyzedSelectItem`に展開され、`SELECT id + 1`のような式も正しく解析されています。

## 現時点の課題

Analyzed ASTは「何をすべきか」を表現していますが、「どう実行するか」はまだ決まっていません。実際にディスクからデータを読み取り、結果を返すにはExecutorが必要です。

## 次回予告

明日は**Executor**を実装します。

Analyzed ASTを受け取り、実際にデータを読み取って結果を返すエンジンを作ります。
