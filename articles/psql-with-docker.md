---
title: "PostgreSQLのDockerを使う開発環境でよくやるTips"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postgresql", "docker", "dbmate"]
published: true
---

最近はデータベースをローカルにインストールしないで Docker を使用するケースが増えてきたなと思います。
DB をコンテナにすることでローカルや CI 上で簡単に環境を構築できるメリットがある一方、少し工夫をしないといけない点もあるのでそれを記事にまとめました。
なおタイトルに PostgreSQL と書いてありますが、他の DB でも同じようなケースには対応できるはずです。

# 前提環境

これ以降はこのような `docker-compose.yml` を使用している前提で書いています。

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17
    container_name: postgresql-17-zenn
    environment:
      - POSTGRES_USER=zenn
      - POSTGRES_PASSWORD=zenn-pw
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - zenn-network
    ports:
      - 5432:5432
    platform: linux/amd64

volumes:
  db_data:
    driver: local

networks:
  zenn-network:
```

# DB 付随のソフトウェアについて考える

**DB 付随のソフトウェア** とは `psql` や `pg_dump` などです。これらは PostgreSQL をローカルにインストールすると一緒にインストールされるのですが、Docker で立ち上げると存在しません。

:::message
バージョンによるかもしれないですが PRM をみた感じ `postgresql-server` を指定してインストールした場合はローカルに存在しないようです。
大体の場合は開発環境だとクライアントとセットで入れるので両方入っているケースが多い気がします。

RPM
https://rpmfind.net/linux/RPM/openmandriva/cooker/x86_64/main/release/postgresql-server-17.0-1.x86_64.html

Brew
https://formulae.brew.sh/formula/postgresql@17
:::

そのためこれらを行いたい場合、以下どちらかのアプローチを取ることになります。

1. 単体でインストールする
2. 付随のソフトウェアも Docker 経由で使う

今回は 2 を採用して以降をまとめます。

## シンプルな DB の操作をしたいケース

具体的には SQL を投げたり、データをダンプしたりしたいケースです。
これらを実行したい場合は解決方法もとてもシンプルで、コンテナに直接入り CLI でコマンドを投げましょう。

```bash
docker compose up -d
docker compose exec db bash -c "PGPASSWORD=zenn-pw psql -h localhost -U zenn"
```

ただし実行している db のコンテナに bash などがインストールされていないケースや、そもそも付随のソフトウェアのコマンドが入っていないケースはこの方法が使えませんので、次のケースを参考にしてください。

## スクリプトや他のソフトウェアを実行して DB の操作をしたいケース

今度は少し複雑になります。スクリプトや他のソフトウェアを実行したいケースを考えます。

スクリプトがとても簡素な場合且つ bash などが事前にインストールされている場合はこのようなものを作っておき、少し上に記載したコンテナに直接入ってコマンドを実行するのが良いでしょう。

```dockerfile
# Dockerfile
FROM postgres:17

COPY /path/to/script.sh /usr/local/bin/script.sh
RUN chmod +x /usr/local/bin/script.sh
CMD [ "/bin/bash" ]
```

```yaml
# docker-compose.yml
services:
  db:
    build:
      context: .
      dockerfile: ./Dockerfile
    tty: true
    container_name: postgresql-17-zenn
    # 以下は同じ
```

それ以外は別途 Dockerfile を作成し、コンテナを立てて実行する形にするのがいいでしょう。
具体的なコードのサンプルを書いてみました。

```dockerfile
# Dockerfile
FROM postgres:17

# 必要なソフトをインストールする処理を書く
RUN apt update && \
    apt upgrade -y && \
    DEBIAN_FRONTEND=noninteractive apt install -y \
    ca-certificates \
    curl \
    vim \
    wget \
    unzip

CMD [ "/bin/bash" ]
```

```yaml
# docker-compose.yml
services:
  db:
    # 以下略

  bastion:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: postgresql-17-bastion
    tty: true
    networks:
      - zenn-network
    command: tail -F /dev/null
    platform: linux/amd64
```

こうすることで docker compose 上で db, bastion というコンテナが立ち上がります。
db, bastion は同じ Docker の仮想ネットワーク上に構築されるため、bastion -> db という経路で `psql` を実行することができます。

```bash
% docker compose up -d
[+] Running 4/4
 ✔ Network psql-with-docker_zenn-network  Created                                                                              0.0s
 ✔ Volume "psql-with-docker_db_data"      Created                                                                              0.0s
 ✔ Container postgresql-17-zenn           Started                                                                              0.2s
 ✔ Container postgresql-17-bastion        Started                                                                              0.2s

% docker compose ps
NAME                    IMAGE         COMMAND                  SERVICE   CREATED         STATUS         PORTS
postgresql-17-bastion   postgres:17   "docker-entrypoint.s…"   bastion   9 seconds ago   Up 8 seconds   5432/tcp
postgresql-17-zenn      postgres:17   "docker-entrypoint.s…"   db        9 seconds ago   Up 8 seconds   0.0.0.0:5432->5432/tcp

% docker compose exec bastion bash -c "PGPASSWORD=zenn-pw psql -h db -U zenn"
psql (17.0 (Debian 17.0-1.pgdg120+1))
Type "help" for help.

zenn=# \l
                                                 List of databases
   Name    | Owner | Encoding | Locale Provider |  Collate   |   Ctype    | Locale | ICU Rules | Access privileges
-----------+-------+----------+-----------------+------------+------------+--------+-----------+-------------------
 postgres  | zenn  | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
 template0 | zenn  | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/zenn          +
           |       |          |                 |            |            |        |           | zenn=CTc/zenn
 template1 | zenn  | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/zenn          +
           |       |          |                 |            |            |        |           | zenn=CTc/zenn
 zenn      | zenn  | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
(4 rows)
```

# 結局どっちを使えばいいか迷う時

私個人の意見ですが、DB のコンテナをビルドする頻度に依存すると思っています。
例えば DB のコンテナを作り直すことがあったとします。（大体そういう時はコンテナが調子悪くなってきた時ですが）その時にコンテナの立ち上げに時間かかるのは嫌ですよね。CI 上でも高頻度で実行するコードは気軽に再現性のある環境を早く実行できるようにしておくほうが開発の体験としては良くなると思います。

なのでそのチームや環境でどういう操作をすることが多いかを考えて決めるのがいいと思います。

# (おまけ) 私が遭遇した pg_dump のバージョン違いで起きたトラブル

なんでこんな記事を書いたかというと、 [dbmate](https://github.com/amacneil/dbmate) を使用して PostgreSQL with Docker をマイグレーションする機会があった際に **`pg_dump` のバージョン違いによって特定の人の環境で `schema.sql` が作成されないという事象が発生した** からです。

dbmate のドキュメントにしっかり書いているのですが、マイグレーション後のスキーマを自動で吐き出してくれる機能が実行環境のローカルのソフトウェアに依存しているんですね。そしてそれが実行できない場合はスルーされるという 😇

> this command relies on the respective pg_dump, mysqldump, or sqlite3 commands being available in your PATH. If these tools are not available, dbmate will silently skip the schema dump step during up, migrate, or rollback actions.

https://github.com/amacneil/dbmate?tab=readme-ov-file#exporting-schema-file

CI 上では変更をチェックするワークフローで差分が出てエラーとして扱われる(内部で `git diff --cached --exit-code` でチェクしている)のでそのタイミングで気づくことができたのですが、原因を調べるのに少し苦労しました...

この挙動のためだけにローカルに psql などのソフトをビルドをするのも嫌ですし、Docker 上でなんとかしたいと思って今回こんな記事を書いてみました。

# おわりに

pgAdmin を使えばいいじゃないか！ という声もある気がするんですが、なるべく CLI 上で完結したい＆ローカルにインストールしたくない派だったのと、Docker 使ってるなら Docker 上で完結させたいという思いがあったのでこういう形にして私は使っています 😁

書きながら少し頭をよぎったのは、最近は DevContainer を使用して開発環境を全て Docker 上に持っていくケースが増えてきた点。DevContaner 上で扱う Docker は色々とつらいと聞いているので両者が混在している環境だと改善の方法をもう少し考える必要があるんだろうなと思いました。
