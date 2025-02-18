---
title: "TROCCOのSelf-Hosted RunnerをAmazon ECS（Fargate）で動かしてプライベートネット間の転送をしてみた"
emoji: "🛤️"
type: "tech"
topics: ["trocco", "aws", "ecs", "fargate"]
published: false
publication_name: "primenumber"
---

# はじめに

2025/02現在、クラウドETL「TROCCO」では、任意の環境でデータ転送処理を完結できる、「Self-Hosted Runner」機能の開発が進められています。
この度、β版としてトライアルの受付が始まったので、実際に試してみました。

## 筆者について

筆者は、サービスの黎明期よりTROCCOの開発に従事、Self-Hosted Runnerの設計や実装の一部も行っている、いわゆる中の人です。
今回は、現時点での公開可能情報から、できる限り本機能のイメージを明確にしていただければと、執筆しています。

:::message
本記事でご紹介する内容は執筆時点のものであり、正式リリースに向けて仕様が変更される可能性があります。
:::

# 既存のTROCCOのアーキテクチャ

まず、既存のTROCCOのアーキテクチャについて軽く触れておきます。
TROCCOでは、データ転送ジョブが立ち上がると、弊社管理のAmazon EKS上でPodが立ち上がり、終了するとPodも破棄されます。
Pod内では、OSSのEmbulkで転送処理が行われつつ、その実行状況を管理するコードが動いています。
詳しくは「アーキテクチャConference2024」での拙スライドを参照ください。

https://speakerdeck.com/gtnao/paastosaasnojing-mu-dexin-lai-xing-tokai-fa-su-du-woliang-li-suru-trocco-r-nokoremadetokorekara?slide=42

## 課題

ここで問題となるのは、転送元/先のリソースがインターネットからアクセスできない場所にあるケースです。
SaaS系のコネクターは、トークンやID/Passwordによる認証があるとはいえ、インターネット上のどこからでもアクセス可能なことが多いです。
一方で、クラウドやオンプレミス上のデータベースやファイルストレージなどは、何かしらネットワーク面での制約があるはずです。
その場合、何らかの方法でTROCCOからアクセス可能となるように設定いただく必要があります。
例えば、踏み台サーバーを経由したり、AWSであればSSMやPrivateLinkを利用したり、といった具合です。
しかしながら、上記方法が使えない場合もありますし、踏み台経由にせよTROCCOのグローバルIPからのアクセスを明示的に許可いただく必要があります。

![old_architecture](https://storage.googleapis.com/zenn-user-upload/cbd60cac4b20-20250217.webp)

# Self-Hosted Runnerのアーキテクチャ

前述の課題を解決するべく、Self-Hosted Runner機能ではコンテナイメージを配布して、データ転送処理部分をユーザーの任意の環境で行えるアーキテクチャを取っています。
一方で、設定の作成や管理は今まで通りTROCCOのサーバー上で行われる、ハイブリッドな形式です。
GitHub ActionsのSelf-Hosted Runnerを触ったことがある方は、それをイメージしていただけると分かりやすいはずです。

https://docs.github.com/ja/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners

## 今回の題材

今回は、Amazon ECS（Fargate）上でコンテナを動かしてみます。
より価値を実感してもらうため、以下のような環境下での転送を題材にしてみます。

- 2つの異なるVPCのプライベートサブネット上でAmazon RDS（MySQL）が動いている
- 両ネットワークはVPC Peeringで繋がっているが、RDSへは外部からのアクセスができない
- 片方のVPC上でECSを動かすことを想定し、そこからのRDSへのアクセスを許可しておく

![aws_architecture](https://storage.googleapis.com/zenn-user-upload/812226420eaf-20250217.webp)

赤色の矢印（リクエスト）が、全てSelf-Hosted Runnerを起点に走っています。
データ転送処理が実行されるのはSelf-Hosted Runner上なので、転送元/先のリソースへアクセス可能な環境下で起動することで、ネットワーク的制約を乗り越えることができます。

上記の構成は機能に直接関係はしない一例ですが、再現性のためTerraformのHCLファイルを置いておきます。
（ECSに関連する部分は、後述で画面から作成しているため含まれていません。）

https://gist.github.com/gtnao/73ae235b4f7f576327711f3543a7f0fb

## TROCCOとの通信

ハイブリッドな機能なので、Self-Hosted RunnerとTROCCOは随時を通信する必要があります。
例えば、以下が挙げられます。

- 死活監視のために、Runner側からTROCCO側へ、定期的にHeartbeatリクエストを行う
  - ジョブが発火した際には、Heartbeatの**レスポンス**としてその旨を受け取る
- ジョブ実行前に、Runnerg側からTROCCO側へ、必要な接続情報を取得しに行く
- ジョブ実行中に、Runnerg側からTROCCO側へ、実行ログを送信する
- ジョブ完了時に、Runner側からTROCCO側へ、ステータスを通知する

**簡略図**
![sequence](https://storage.googleapis.com/zenn-user-upload/db413d15b308-20250218.png)

その全てが、**Sels-Hosted Runner起点**となっていることが重要です。

例えば、AWSのSecurity Groupではステートフルな（返りのパケットは自動的に許可される）仕組みなので、アウトバウンドな設定が通れば通信できます。
また、一般的にアウトバウンドな設定は、厳しく制限されるインバウンドな設定に比べて、比較的緩めに設定されることが多いです。
このようなアーキテクチャを取ることで、既存のTROCCOでは難しかった要件を満たせるようになります。

# Self-Hosted Runnerを動かす

それでは、実例に入っていきます。

## クラスターの作成

Self-Hosted Runnerのオプションが有効な場合、TROCCOのサイドメニューに表示されます。
![sidemenu](https://storage.googleapis.com/zenn-user-upload/2816d19b9f21-20250217.png)

まずは、コンテナをまとめる論理的な概念である「クラスター」を作成します。
![create_cluster](https://storage.googleapis.com/zenn-user-upload/ba2a19b6824a-20250217.png)

作成時にはトークンが発行されます。
セキュリティの都合上、一度しか表示されないため控えておいてください。
トークンは2つまで作成できるため、安全にローテーションが可能です。
![cluster_detail](https://storage.googleapis.com/zenn-user-upload/044ed9ac27c5-20250217.png)

例として表示されているDockerコマンド（`docker pull`, `docker run`）を実行することで簡単に立ち上げることができます。
`docker run`では、環境変数として先ほどのトークンを指定してください。

Self-Hosted Runnerの1つの設計思想としてポータビリティがあります。
クラウドサービスや構成に寄らず、コンテナが実行できる環境であれば、1台から複数台まで互いに依存せずに立ち上げることができます。
極端な話、手元のPC上でも実行可能です。
とりあえず挙動を確認したい、といったアドホックな場合には有用でしょう。

## ECSの設定

全体の構成はTerraformで作成しておきましたが、こちらは分かりやすさのため、マネージドコンソールから手動で準備していきます。

### トークンの安全な管理

まず、トークンをセキュアにECSタスクへ設定すべく、System ManagerのParameter Storeに登録しておきます。
一手間かかりますが、こういうところはしっかりやっておきましょう。
![parameter_store](https://storage.googleapis.com/zenn-user-upload/a35e4bc98219-20250217.png)

### TaskExecutionRoleの用意

ECSのTaskExecutionRoleに指定するためのIAM Roleを作成しておきます。
今回は、CloudWatch Logsへのロギング、Parameter Storeからの取得権限が必要です。
前者はマネージドポリシー（`AmazonECSTaskExecutionRolePolicy`）に含まれていますが、後者は自前で作成する必要があります。

![policy](https://storage.googleapis.com/zenn-user-upload/a787d7ab751d-20250217.png)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMParameterAccess",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParameters"],
      "Resource": [
        "arn:aws:ssm:ap-northeast-1:<ACCOUNT_ID>:parameter/self_hosted_runner_token"
      ]
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:ap-northeast-1:<ACCOUNT_ID>:alias/aws/ssm"
    }
  ]
}
```

ECSタスクに`AmazonECSTaskExecutionRolePolicy`と作成したポリシーを許可するIAM Roleを作成します。
![role_1](https://storage.googleapis.com/zenn-user-upload/3cd63d8829a0-20250217.png)
![role_2](https://storage.googleapis.com/zenn-user-upload/e9788d3e5915-20250217.png)

### ロググループの作成

CloudWatch Logsのロググループも事前に作成しておきます。
`AmazonECSTaskExecutionRolePolicy`には、ロググループを作成する権限がないためです。
![log_group](https://storage.googleapis.com/zenn-user-upload/f23ead5ce498-20250217.png)

### セキュリティグループの作成

SecutiryGroupを作成しておきます。
アーキテクチャ上インバウンドなアクセスは発生しないので、アウトバウンドなルールだけ許可しておきます。
![sg](https://storage.googleapis.com/zenn-user-upload/d1362c64dfec-20250217.png)

### ECSクラスター

それでは、ECS自体の設定をしていきます。
今回は、Fargateを利用してECSクラスターを作成します。
![ecs_cluster](https://storage.googleapis.com/zenn-user-upload/e1531abd2a31-20250217.png)

### ECSタスク定義

次に、ECSタスク定義を作成します。

Self-Hosted Runnerの最小推奨メモリは2GBです。
とりあえず、その値に設定しておきます。
タスク実行ロールでは先ほど作成したIAM Roleを指定します。
![task_definition_1](https://storage.googleapis.com/zenn-user-upload/09a20b96aaf6-20250217.png)
イメージのURIにはTROCCOの画面上に表示されていたものを指定してください。
インバウンドなリクエストを受け付けることはないため、ポートマッピングは指定不要です。
環境変数には、`TROCCO_REGISTRATION_TOKEN`にValueFromでParametor StoreのARNを指定して埋め込みます。
![task_definitioin_2](https://storage.googleapis.com/zenn-user-upload/0e586c3c00b9-20250217.png)
ログ設定では、先ほど作成したCloudWatch Logsのロググループを指定します。
![task_definition_3](https://storage.googleapis.com/zenn-user-upload/44ccdfad5ba7-20250217.png)

### ECSサービス

最後に、ECSサービスを作成してデプロイします。
とりあえず、先ほど作成したタスク定義を1タスク起動する設定とします。
![service_1](https://storage.googleapis.com/zenn-user-upload/91bd8c7d4069-20250217.png)
ネットワークは転送元側のパブリックサブネットで起動するようにしておき、先ほど作成したSecurity Groupを指定します。
![service_2](https://storage.googleapis.com/zenn-user-upload/009e494dc2fb-20250217.png)

タスクが起動し始めると、CloudWatch Logsでログが確認できます。
`Joined cluster successfully`と出ていればひとまず成功です。

![join](https://storage.googleapis.com/zenn-user-upload/0d0fc39a98bd-20250218.png)

TROCCOの画面上からも確認できます。
まだジョブ待ち状態なので`IDLE`ステータスが表示されています。

![idle](https://storage.googleapis.com/zenn-user-upload/c59b787ff6b9-20250218.png)

## 転送設定の作成

TROCCO側の設定に戻ります。

:::message
Self-Hosted Runnerでは、全てのコネクターをサポートしているわけではありません。
DB系、DWH系、ファイル系のコネクターを中心にサポートしていく予定です。
:::

### 接続情報

まずは、それぞれのMySQLへの接続情報を作成します。
ホスト名が解決されるIPは、それぞれのプライベートサブネットのIPのため、従来のTROCCOであればもちろんアクセスできません。

![source](https://storage.googleapis.com/zenn-user-upload/98d26fd965b1-20250217.png)
![destination](https://storage.googleapis.com/zenn-user-upload/797190c4c11e-20250217.png)

### 転送設定

次に、転送設定を作成します。
Self-Hosted Runnerオプションが有効の場合、ジョブを動かすクラスターを選択可能です。
選択しないことも可能なので、ある設定は従来通りTROCCO上で動かし、ある設定は自前のコンテナ上で動かすこともできます。

![step1](https://storage.googleapis.com/zenn-user-upload/f574ad210bbb-20250217.png)

その後は、設定を通常通り作成していきます。
Self-Hosted Runner機能の特徴として、設定を作成するフローは既存のTROCCOとシームレスという点があります。

![step1_in](https://storage.googleapis.com/zenn-user-upload/40f3ea2b238e-20250217.png)
![step1_out](https://storage.googleapis.com/zenn-user-upload/4a2a8939bd3a-20250217.png)

### プレビュー

STEP2へ進むと、従来通りプレビューが走ります。
ただしこちらは、Self-Hosted Runner上でEmbulkのpreviewコマンドが実行されています。

![preview](https://storage.googleapis.com/zenn-user-upload/aa747c2a61c1-20250217.png)

TROCCOの仕様上、スキーマ情報を決定するために必ずこのフローが必要になります。
しかしながら、セキュリティ都合上、実プレビューデータをTROCCOへ送ることを許可できないケースもあります。
この場合は、コンテナ起動時に環境変数`TROCCO_NO_SEND_PREVIEW=true`を渡すことにより、必要不可欠なカラム情報のみが送られるようにできます。

![no_send_preview](https://storage.googleapis.com/zenn-user-upload/4c15dd919d34-20250218.png)

## ジョブ実行

それでは、データ転送ジョブを実行してみます。

CloudWatch Logsを見ると実行ログが流れ始めました。
![active_log](https://storage.googleapis.com/zenn-user-upload/4977da328f4f-20250217.png)
TROCCOの画面上でも、Runnerのステータスが`ACTIVE`になっています。
![active_status](https://storage.googleapis.com/zenn-user-upload/80eada9f1411-20250217.png)

無事成功しました。
![success_job](https://storage.googleapis.com/zenn-user-upload/f5d9b822fc0f-20250217.png)

転送先のMySQLを覗いてみます。
実際にデータが入っていることも確認できました。
![result](https://storage.googleapis.com/zenn-user-upload/27220d304d78-20250218.png)

### CPUを増やしてみる

ECSタスク定義でvCPU数を増やしてデプロイしてみます。

TROCCOの裏側で利用しているEmbulkは、CPU数に比例して並列度が上がります。
コネクタの組み合わせや諸条件次第なので、一概に処理時間が比例的に短くなるわけではないですが、試してみましょう。
今回は、1vCPUから8vCPUに上げてみます。
![task_definition_cpu](https://storage.googleapis.com/zenn-user-upload/3c8459d304ef-20250217.png)

新しいタスク定義でサービスを更新します。
![service_cpu](https://storage.googleapis.com/zenn-user-upload/618ba544e00c-20250217.png)

新しいタスクが立ち上がりました。
![new_task](https://storage.googleapis.com/zenn-user-upload/34271a64abba-20250217.png)

同一の転送設定を実行してみると以下になりました。
![8vCPU](https://storage.googleapis.com/zenn-user-upload/9b5df184c117-20250217.png)

以下の先ほどの結果と比較すると、20秒弱早くなったことが確認できました。
![1vCPU](https://storage.googleapis.com/zenn-user-upload/dfec0d5ad712-20250217.png)

このように、ユーザー側で柔軟にリソースを調整できることも、Self-Hosted Runnerの価値の1つです。

### 並列度を上げてみる

ECSサービスでタスク数を4に設定してデプロイしてみます。
![ecs_service_task_4](https://storage.googleapis.com/zenn-user-upload/1b8e3bd65ceb-20250217.png)

その後、転送ジョブを同時に4件起動してみます。
それぞれのRunnerがACTIVEステータスになっており、並列でジョブが実行されていることが確認できます。
![runner_4](https://storage.googleapis.com/zenn-user-upload/45b37de3aab8-20250217.png)

# おわりに

今回は、TROCCO Self-Hosted Runnerが解決する課題、技術仕様、実際の構築例まで、広くご紹介させていただきました。
トライアルも受け付けておりますので、ご関心をお持ちの方は![サービスページ](https://primenumber.com/trocco/features/self-hosted-runner)からお問合せください。
