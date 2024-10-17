---
title: "Goでサーバーを立ち上げてテストを実行するときはbufconnを使おう"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "test", "http"]
published: true
---

Go で HTTP サーバーを立ててインテグレーションテストを行いたい場合がありますよね。
そのまま実装すると実際に OS にポートを確保してもらい、そのポート経由で通信する経路のテストになります。

しかしこの方法にはいくつかの問題が発生する可能性があります。

- OS への呼び出しが増えるため、並列でテストを実行すればするほどテストが遅くなる。
- 実際にポートを確保して通信するため、問題発生時にネットワークレイヤーの考慮が必要になる可能性がある。
- ポートを固定で指定していた場合、並列でテストを回すのが難しい。

アプリケーションからポートの確保を行う経路のテストを確実に行いたいケースは別として、多くの場合は実際のアプリケーションレイヤーで正常に動作するかを確認したいケースが多いと思います。

そこで今回は、そのようなユースケースにおいて `bufconn` を使用して HTTP の通信を仮想化し、テストを高速化する方法を見ていきます。

# bufconn とは

> Package bufconn provides a net.Conn implemented by a buffer and related dialing and listening functionality.

https://pkg.go.dev/google.golang.org/grpc/test/bufconn

元は gRPC 用として設計されていますが、やっていることはバッファを利用して `net.Listener` の実装をしているだけなので、これを HTTP サーバーで使用する形のアプローチを取ります。

# bufconn を使用して仮想的な通信をテストする

実装した例はこちらにあります。

https://github.com/tatsuyayamauchi/go-samples/blob/ef0ef45c6a593458299bf8c889d6e25b99d3d93c/virtual-conn-http-server/server_test.go#L16-L57

やっていることとしてはシンプルに 2 つです。

- `http.Server` `http.Client` のリスナーを bufconn のものに差し替えて初期化する
- リクエストする際にリスナーのポート番号に対してリクエストするようにする

なお今回は `net/http` をベースに作成しましたが、OpenAPI の定義を元に https://github.com/oapi-codegen/oapi-codegen 等でコードを生成している場合も少し工夫は必要ですが指定可能でした。

# (おまけ) ポートをランダムで割り当てたいケースの対処方法

bufconn を使用するとポートのことを考えなくてよくなりますが、実際にポートの呼び出しをして通信したいケースもあると思います。
その際に並列でテストを回す際に、ポートをどう割り当てるかが問題になってきます。

実は空いてるポートを自動で割り当てる指定の仕方が存在します。
`net.Listen` のドキュメントに下記のような記載があります。

> If the port in the address parameter is empty or "0", as in "127.0.0.1:" or "[::1]:0", a port number is automatically chosen. The [Addr] method of [Listener] can be used to discover the chosen port.

https://pkg.go.dev/net#Listen

つまり空白か 0 を指定すると自動的にポートが割り当てられるため、下記のようにすることでポートのバッティングは防ぐことができます。

```go
svr := &http.Server{
    Addr: ":0",
}
go func() {
    if err := svr.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        return err
    }
}()
```

# おわりに

bufconn は gRPC の記事では出てくるのですが、なぜか HTTP サーバーの例だとなかったので書きました。
記事を書くにあたって久しぶりに `net/http` の実装を読む機会になったので、勉強になりました。
調べがら読んでいると https://zenn.dev/hsaki/books/golang-httpserver-internal/viewer/intro が出てきたりしたので近々熟読させていただきたいと思いました。
サーバーの実装はプロジェクトの立ち上げ時期しか書くことが少なく忘れがちなので、この記事がどこかで役に立てばいいなと思います。
