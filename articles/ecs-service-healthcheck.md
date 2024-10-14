---
title: "ECS on Fargate でGoのアプリを起動させた時にコンテナのヘルスチェックで苦しんだ話"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ecs", "aws", "go", "docker"]
published: true
---

Go のアプリを ECS on Fargate 上にデプロイした時に、コンテナのヘルスチェックが通らない事象にあったのでその時の調査ログです。

# 結論

- ECS のコンテナのヘルスチェックは Docker のコンテナのヘルスチェック機能を利用している。
  - 指定がある場合は上書きしてモニタリングをする。
  - 指定がない場合はモニタリング自体をしない。
- Docker のヘルスチェックはコンテナのイメージに **ヘルスチェックで使うコマンドが含まれていること** を確認する

# ヘルスチェックについて理解する

シンプルにしていますが、以下の構成で話を進めます。

![](/images/ecs-healthcheck-architecture.drawio.png)

## ヘルスチェックの種類

上記の構成で ECS 上でアプリをルーティングする際に、ヘルスチェックと呼ばれるものは２つあります。

1. ALB のヘルスチェック
2. ECS コンテナのヘルスチェック

### ALB のヘルスチェック

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/alb.html

ECS のサービスに ALB のターゲットグループを紐づけている場合、ヘルスチェックが走ります。
この記事ではここはメインではないため割愛しますが、ここは今回すでにパスしていました。

余談ですが、いつの間にかサービスへのルーティング、ALB 以外に NLB や GLB も対応していたんですね。
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/service-load-balancing.html

### ECS コンテナのヘルスチェック

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/healthcheck.html

タスク定義でコンテナのヘルスチェックを行うことができます。
ドキュメントをよく読むと書いてあるのですが、指定した場合のみコンテナのヘルスチェックを走らせることができ、指定しない場合はヘルスチェック自体を行わない挙動をします。

ここに書いてある Docker のヘルスチェック機能については Docker の公式ドキュメントに詳細な説明があるので、こちらを参考にしてください。
https://docs.docker.com/reference/dockerfile/#healthcheck

# Docker のヘルスチェックで行なっていること

ECS のタスク定義に設定したコマンドを Docker コンテナ上で実行して、その結果に応じてコンテナの起動状態を監視するという理解をしています。
ここで重要なのが Docker コンテナ上でコマンドを実行して確認するという点です。

設定当時の ECS のタスク定義はこのような定義をしており、Go のアプリのビルドは [ko](https://ko.build/) を使用して、ECR にプッシュしていました。

```json
{
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:xxxx",
      "cpu": 256,
      "memory": 128,
      "essential": true,
      "portMappings": [
        {
          "hostPort": 8080,
          "protocol": "tcp",
          "containerPort": 8080
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/ || exit 1"]
      }
    }
  ],
  "family": "myapp",
  "requiresCompatibilities": ["FARGATE"]
}
```

## ko を使うと curl 等のコマンドが使えない問題

https://github.com/ko-build/ko/blob/main/docs/configuration.md#overriding-base-images

記事執筆時点では ko のデフォルトのベースイメージは chainguard 社が出している `cgr.dev/chainguard/static` を使用していました。
https://edu.chainguard.dev/chainguard/chainguard-images/overview/

なので実行に必要な curl はもちろんのこと、shell すらも入っていないのでヘルスチェックコマンドが成功せずタスクの起動に失敗し続けるというのが原因でした。
また今回調査していて難しいなと思ったのがコンテナのヘルスチェックを実行したログがどこにも出ていない点でした。
(実行できていないのでログが出ないのは当たり前ですが...)

### ko のベースイメージの話

`cgr.dev/chainguard/static` を pull してサイズを見てみたらとても小さいことがわかりますね

```bash
% docker images cgr.dev/chainguard/static
REPOSITORY                  TAG       IMAGE ID       CREATED      SIZE
cgr.dev/chainguard/static   latest    232134293814   3 days ago   1.33MB
```

Go 以外にも Node や Python など他のイメージもあったので、セキュアなコンテナを作りたい時にはこれを参考にしてみるのもいいかもしれません。
https://edu.chainguard.dev/chainguard/chainguard-images/getting-started/

## ko を使ったビルドでヘルスチェックをするにはどうすべきなのか

私個人としての考えは、LB からのヘルスチェックの実装にロジックを追加してあげる方向に倒すのがいいと思っています。例えば LB への疎通や、特定のファイルが存在しているか、外部サービスの起動が完了しているか、などです。
(ただしそれで負荷が急増するのであれば別のアプローチが必要)

これ以外にも ko のベースイメージを alpine 系に変更するというやり方もあると思います。
その際はベースのイメージサイズが増えるのと、アプリ実行に不要なライブラリ等が含まれるのでセキュリティのリスクが多少上がるため、関わっているメンバーと相談して決めていければいいのかなと考えています。

# 学び

- Docker のヘルスチェックの存在を知らなかった。docker compose でも簡単に書けるらしい。
- いつの間にか ko のイメージが Google のものから変わっていた。
- Google が管理している distroless 以外知らなかったので、 今回 chainguard の存在を知ることができてよかった。セキュアイメージを作る時に参考にしたい。
