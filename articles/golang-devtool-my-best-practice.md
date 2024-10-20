---
title: "Goで開発者用コマンドを作る時はcobraでまとめておくとよかった話"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "refactoring", "cobra", "makefile"]
published: true
---

Go のプロジェクトに関わっていると、 **何かのデータを元にファイルを自動生成するツールを Go で開発すること** があるのではないでしょうか？
例えば API の定義ファイルから自動的にハンドラーを生成したり、Go に書いた設定ファイルを json や yaml に書き出したりなどです。
Go では書くコード量が多くなりがちなので、なるべくコピペで書くような作業を自動生成ツールに任せ、ビジネスにコミットするような重要なコードを書くことに集中したいですよね。

しかしプロジェクトが成長するにあたり、それらの実行速度がボトルネックになってくる時があると思います。今回はこれを CLI のライブラリで有名な [cobra](https://github.com/spf13/cobra) を使って解消したという話をしたいと思います。

# 先に結論

- **アプリを実現するコード**と**開発に使用する自動生成ツール** の依存関係を整理しておく
- 後者を cobra を使ってまとめておく
- 自動生成ツールの実行前に後者をビルドしておき、自動生成ではビルドしたバイナリを使用する

# 改善前の状況

以下のようなディレクトリ構成になっているとしましょう。

- **main.go** では cmd/root.go の呼び出しだけをしています。
  - cmd/root.go が cobra の実際の設定になります。
- **cmd 配下** ではアプリケーションを単一バイナリにまとめるためのコマンドが入っています。
  - サブコマンドごとにディレクトリを分けており、この例だと `app` と `data user add`コマンドが登録されています。
- **tools 配下** では自動生成コマンドごとにディレクトリを切り、ディレクトリ別で main パッケージから呼び出す形で保存しています。
- **internal 配下** では実際のアプリケーションのコードが入っています。

```
├── Makefile
├── cmd
│   ├── app
│   │   └── cmd.go
│   ├── data
│   │   ├── cmd.go
│   │   └── user
│   │       ├── add
│   │       │   └── cmd.go
│   │       └── cmd.go
│   └── root.go
├── go.mod
├── internal
├── main.go
└── tools
    └── foo
        └── main.go
```

---

この状態で Makefile には以下のようなコードが書かれていたとしましょう。
コマンドは適当ですが、環境が増えるごとに `go run tools/foo/main.go` の数が増えるため、遅くなりそうです。
このようなコマンドが 10 個あるとその分遅くなることは容易に想像できると思います。

```makefile
codegen: ## Code generate
    go run tools/foo/main.go --env qa --type model --output /path/to/qa
    go run tools/foo/main.go --env qa --type repository --output /path/to/qa
    go run tools/foo/main.go --env qa --type handler --output /path/to/qa
```

# [改善その１] 事前に開発者コマンドをビルドする

`codegen` のコマンドが遅いのであれば、事前にビルドすることでこの問題は解消します。

```makefile
build-tools: ## Build tools command
    go build -o bin/foo tools/foo/main.go

codegen: ## Code generate
    make build-tools
    bin/foo --env qa --type model --output /path/to/qa
    bin/foo --env qa --type repository --output /path/to/qa
    bin/foo --env qa --type handler --output /path/to/qa
```

しかしビルドするコマンドが増えると、 `build-tools` が肥大化しボトルネックが移動することになります。
なのでこれらのファイルを単一バイナリとしてビルドできるようにします。

# [改善その２] cobra で開発者コマンドをまとめる

呼び出し先を一括で集約すればあとは cobra が自動で単一バイナリにしてくれるのでこうします。

- **cmddev/main.go** を開発者用コマンドの呼び出し元とする。
  - 正直この main.go の位置は微妙だと思っているので、もっといい場所があったら教えてください！
- **cmddev 配下** は自動生成を行うための開発者用コマンドです。
  - ファイルの具体的な実装は **tools 配下** にあります。

```go
// cmddev/main.go
package main

import (
	"log"

	"github.com/spf13/cobra"

	"github.com/tatsuyayamauchi/example/cmddev/foo"
)

var rootCmd = &cobra.Command{
	Use: "development command",
}

func init() {
	rootCmd.AddCommand(foo.Cmd())
}

func main() {
	err := rootCmd.Execute()
	if err != nil {
		log.Fatal(err)
	}
}
```

```
├── Makefile
├── cmd
│   ├── app
│   │   └── cmd.go
│   ├── data
│   │   ├── cmd.go
│   │   └── user
│   │       ├── add
│   │       │   └── cmd.go
│   │       └── cmd.go
│   └── root.go
├── cmddev
│   ├── foo
│   │   └── cmd.go
│   └── main.go
├── go.mod
├── internal
├── main.go
└── tools
    └── foo
        └── run.go
```

このようにすることで internal とも実装をディレクトリ単位で依存を分離しつつ、単一バイナリにすることができるようになります。

上記のディレクトリ構成にした場合、実際の Makefile はこのようになります。`foo` がサブコマンドとして登録されています。
ここに `bar` などの他の開発用のサブコマンドを追加してもビルドの時間はそこまで伸びないので、go build を複数回叩いていた時のボトルネックを潰すことができるようになりました。
またボトルネックを潰したので動作も高速化しました 🎉

```makefile
build-dev: ## Build cmddev command
    go build -o bin/dev cmddev/main.go

codegen: ## Code generate
    make build-dev
    bin/dev foo --env qa --type model --output /path/to/qa
    bin/dev foo --env qa --type repository --output /path/to/qa
    bin/dev foo --env qa --type handler --output /path/to/qa
```

# 依存関係について

余談なのですが、改善前も改善後もパッケージの依存関係を整理してある状態で記載しています。
ここでいう依存関係は Go のファイルにおいてどのパッケージを import しているか？を指します。
Go では依存関係が循環していると循環参照状態なりビルド(go run による実行も含む)ができなくなるので意識することが大切です。

依存関係を簡潔に整理するという意味で、以下のような形に分けてみました。なお tools 配下で internal 配下のファイルを参照してはいけない等の決まりは特に作っていないです。

- cmd 配下のアプリケーションに関わるパッケージは、internal 配下のディレクトリで具体的な実装をする
- cmddev 配下の開発者ツールは、 tools 配下のディレクトリで実装する。

## 依存関係の可視化

もし依存関係が把握できていないのであれば、 https://github.com/kazdevl/prelviz 等のツールを使用して、今現在の依存関係を可視化してから進めることもできるのでおすすめです。
私はこのような Makefile を書いて依存関係の図をドキュメントに反映しています。（ただし毎回同じ画像を吐き出してくれるわけではないので CI で自動で更新する形にはしていません）

```makefile
update-dependency-image:
    go install github.com/kazdevl/prelviz/cmd/prelviz@latest
    prelviz -i . | docker run --rm --platform=linux/amd64 -i nshine/dot >> images/dependency_graph.gen.png
```

# cobra を使ってドキュメントを自動生成する

README 等には書いてある箇所を見つけられていないのですが、この関数等を使用して cobra を使用しているコードのドキュメントを自動で生成することができます。
https://github.com/spf13/cobra/blob/3a5efaede9d389703a792e2f7bfe3a64bc82ced9/doc/md_docs.go#L119-L129

ただし上記の cmddev/main.go の例だと main パッケージに `cobra.Command` のルートの情報を入れてしまっているので参照できません。
別の場所におけばいいのですが、どこにおくのがいいか迷っているのでいい方法があれば教えてください！
1 点注意としては、cobra のドキュメント自動生成コマンドは `cobra.Command` を参照してドキュメントを自動生成します。
そのため `cmd/root.go` にある `cobra.Command` を cmddev 配下のコマンドとして登録すると cmddev -> cmd の依存関係が発生して cmddev のビルドが通らないケースが発生します。
私もこの罠にハマったので、このコマンドだけ仕方がなく cmddev から除外して `tools/cobradocsgen/main.go` のような完全別ファイルとして追加しています。

# おわりに

Go が自動生成を推奨する文化なので、他の言語よりも独自でファイルを自動生成をするケースが多い気がしています。
自動生成が増えてくるとファイルの生成順に矛盾が生じてしまい、とある状態になると自動生成ができないケースも出てくるのでなるべくシンプルにしてあげる必要が出てきます。
(私は前に元ファイル A から自動生成ファイル A,B を作成した時に自動生成ファイル A が B に依存していたため、 A から生成すると自動生成がこけるケースに遭遇しました 😇)

今回の対応で私が関わっている Go のプロジェクトは自動生成周りがかなり整理できたので記事にしてみました。
皆さんそれぞれの Tips 等があると思いますが、 `go run xxx` を多様していて処理が遅くなっていて困っている人たちがいれば、この記事を参考していただければと思います。
