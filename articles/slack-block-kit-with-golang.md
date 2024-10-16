---
title: "Slack の Block Kit をGoで使ってみる"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["slack", "golang"]
published: true
---

Slack の API 通信を書くケースがありいつも Attachments で書いていたのですが、リファレンスを見たらどうやら Slack 的には推奨していないようでした。

> Secondary attachments are a legacy part of the messaging functionality. While we aren't deprecating them, you should understand that they might change in the future, in ways that reduce their visibility or utility.

https://api.slack.com/reference/surfaces/formatting#when-to-use-attachments

また Block Kit の方が機能が充実しているとのことなので、これを機に触ってみました。
なお書き方がだいぶ変わってたので https://github.com/slack-go/slack を使って Go で両者を比較した実装してみたいと思います。

# Block Kit について

> The Block Kit UI framework is built with blocks, block elements and composition objects.
> Blocks are visual components that can be arranged to create app layouts. Apps can add blocks to surfaces like the Home tab, messages and modals.

https://api.slack.com/block-kit

UI フレームワークと記載してある通り、構造化されたブロックの組み合わせでメッセージを表現するようになっています。
なお [Block Kit Builder](https://api.slack.com/block-kit-builder) というものがあり、これで GUI ベースで大枠のメッセージを組み立てることができます。

# 実際に Go で Block Kit を使った Slack のメッセージを組み立ててみる

サンプルで実装したものはこちらにあります。
https://github.com/tatsuyayamauchi/go-samples/blob/main/slack-client/main.go

実行結果
![](/images/slack-block-kit.png)

Block Kit を使った方が見やすいですね。
ただ実装量が Block Kit の方が多いので、どちらを使うかは結構迷うところ。
(Attachments も markdown 記法とか乱用してるのどうなんだ？という気持ちにはなる...)
実装例が豊富にあればそこまで苦労しないのかなと。ただ私が探したときはあんまり見つかりませんでした 😂

使ってみてわかったのですが、 _Block Kit Builder_ はこう使うと効率よく実装を作れそうな気がしました。

1. 大枠の Payload の形を整える
2. 細かいブロック要素についてリファレンスを読んで調べつつ Builder 上に追記し Payload の最終的な形を整える
3. 作成した Payload を元に slack-go 等のライブラリでコードを実装する（大体フィールド名が一致した関数があるはず）

多くのケースにおいて Slack への通知は RichText を使用すると完結するような気がするので、ここだけしっかり読み込んでおけば大体なんとかなるのでは？と思いました。
https://api.slack.com/reference/block-kit/blocks#rich_text

# 感想

Slack に通知するケースは CICD 含む業務でも個人アプリでも結構あるような気がしているので、今回時間をとって整理できてよかったです。
Attachments はチートシートがありましたが、BlockKit はチートシートがなかったので誰か作ってくれることを祈っています！
