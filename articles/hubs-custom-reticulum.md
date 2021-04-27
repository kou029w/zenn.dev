---
title: Mozilla HubsのバックエンドサーバーReticulumを改造する方法
emoji: 🐤
topics: [hubs, habitat, reticulum]
type: tech
published: true
---

![Mozilla Hubs の /duck コマンドのアヒル](https://i.gyazo.com/6805c9439d511c02d8cdf3b9655d65c7.png)

:::message
この記事は [Hubs Advent Calendar 2020](https://qiita.com/advent-calendar/2020/hubs) - 8 日目の記事です。
:::

## はじめに

[Mozilla Hubs](https://hubs.mozilla.com/) 使っていますか。

最近、諸事情で Mozilla Hubs のバックエンドサーバー [Reticulum](https://github.com/mozilla/reticulum) を改造したり、[Hubs Cloud](https://hubs.mozilla.com/cloud) を構築してデプロイするなどしていました。

[Hubs Cloud そのものを構築する方法は公式ドキュメントにあり](https://hubs.mozilla.com/docs/hubs-cloud-aws-quick-start.html)、またクライアント側に関しても[公式ドキュメントにある](https://hubs.mozilla.com/docs/hubs-cloud-custom-clients.html)ので、それに沿ってやればそれほど困らないのですが、バックエンドサーバー Reticulum についてはソースコードは公開されているものの、カスタマイズする方法に関してドキュメントは存在せずほとんど情報がありませんでした。

そのため、それぞれ [Mozilla の公開しているリポジトリのソースコード](https://hubs.mozilla.com/docs/contributing.html)を読みつつ、[AWS Marketplace にある Hubs Cloud Personal](https://aws.amazon.com/marketplace/pp/B084RZH56R) を実際に構築して EC2 インスタンスに SSH でアクセスして、手探りでガチャガチャといろいろ試していくうちに、なんとなく仕組みが分かり改造できるようになったのでその方法を紹介します。

そもそも、どのくらいの人が Reticulum を改造したいのかよく分からないですが、そういう人にとってはこの記事は参考になるかもしれません。

## Reticulum とは

[Elixir](https://elixir-lang.org/)/[Phoenix](https://www.phoenixframework.org/) で作られた Web アプリです。基本的には Mozilla Hubs のユーザーのビデオ・音声のストリーミング以外の部分を担っています。
Reticulum をカスタマイズすることによって、独自のメッセージハンドラーを追加したり、既存の認証・認可機構に介入したりできます。

## Reticulum のセットアップ

まず [Reticulum の README](https://github.com/mozilla/reticulum#readme) を参考にして開発環境を構築します。

### PostgreSQL

PostgreSQL の構築などは Docker Compose を使うと便利なのですが、[リポジトリに含まれる `docker-compose.yml` は `postgres:10` イメージでバージョンが古く](https://github.com/mozilla/reticulum/blob/0f0af37e3d0c914dbdd2636eb885d1c028e13110/docker-compose.yml#L4)、あまりメンテナンスされていない様子だったため役に立ちませんでした。
そのため、私は次のような `docker-compose.yml` ファイルを作成し、`docker-compose up` して使いました。

```yml:docker-compose.yml
version: "3"
services:
  db:
    image: postgres:11
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
```

### Elixir

Ubuntu 20.04 LTS であれば、おそらく次のコマンドでセットアップできます。

```sh
sudo apt-get install -y erlang erlang-dev elixir inotify-tools
mix deps.get
mix ecto.create
yarn --cwd assets
```

ここでは [Yarn](https://yarnpkg.com/) を使っていますが [npm](https://www.npmjs.com/) を使っても問題ないと思います。

以上でセットアップ完了です。

### 開発用サーバーの起動

[README によれば](https://github.com/mozilla/reticulum#3-start-reticulum:~:text=Run%20scripts%2Frun.sh,iex%20%2DS%20mix%20phx.server)、

```sh
iex -S mix phx.server
```

とすれば開発用サーバーを起動できるはずなのですが、少なくとも私の環境では環境変数 `PERMS_KEY` を与えなければ機能しませんでした。
[`config/test.exs` の中にある `perms_key`](https://github.com/mozilla/reticulum/blob/0f0af37e3d0c914dbdd2636eb885d1c028e13110/config/test.exs#L67-L68) をそのまま使うことによって対処しました。

```sh
export PERMS_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIEpgIBAAKCAQEA3RY0qLmdthY6Q0RZ4oyNQSL035BmYLNdleX1qVpG1zfQeLWf\n/otgc8Ho2w8y5wW2W5vpI4a0aexNV2evgfsZKtx0q5WWwjsr2xy0Ak1zhWTgZD+F\noHVGJ0xeFse2PnEhrtWalLacTza5RKEJskbNiTTu4fD+UfOCMctlwudNSs+AkmiP\nSxc8nWrZ5BuvdnEXcJOuw0h4oyyUlkmj+Oa/ZQVH44lmPI9Ih0OakXWpIfOob3X0\nXqcdywlMVI2hzBR3JNodRjyEz33p6E//lY4Iodw9NdcRpohGcxcgQ5vf4r4epLIa\ncr0y5w1ZiRyf6BwyqJ6IBpA7yYpws3r9qxmAqwIDAQABAoIBAQCgwy/hbK9wo3MU\nTNRrdzaTob6b/l1jfanUgRYEYl/WyYAu9ir0JhcptVwERmYGNVIoBRQfQClaSHjo\n0L1/b74aO5oe1rR8Yhh+yL1gWz9gRT0hyEr7paswkkhsmiY7+3m5rxsrfinlM+6+\nJ7dsSi3U0ofOBbZ4kvAeEz/Y3OaIOUbQraP312hQnTVQ3kp7HNi9GcLK9rq2mASu\nO0DxDHXdZMsRN1K4tOKRZDsKGAEfL2jKN7+ndvsDhb4mAQaVKM8iw+g5O4HDA8uB\nmwycaWhjilZWEyUyqvXE8tOMLS59sq6i1qrf8zIMWDOizebF/wnrQ42kzt5kQ0ZJ\nwCPOC3sxAoGBAO6KfWr6WsXD6phnjVXXi+1j3azRKJGQorwQ6K3bXmISdlahngas\nmBGBmI7jYTrPPeXAHUbARo/zLcbuGCf1sPipkAHYVC8f9aUbA205BREB15jNyXr3\nXzhR/ronbn0VeR9iRua2FZjVChz22fdz9MvRJiinP8agYIQ4LovDk3lzAoGBAO1E\nrZpOuv3TMQffPaPemWuvMYfZLgx2/AklgYqSoi683vid9HEEAdVzNWMRrOg0w5EH\nWMEMPwJTYvy3xIgcFmezk5RMHTX2J32JzDJ8Y/uGf1wMrdkt3LkPRfuGepEDDtBa\nrUSO/MeGXLu5p8QByUZkvTLJ4rJwF2HZBUehrm3pAoGBANg1+tveNCyRGbAuG/M0\nvgXbwO+FXWojWP1xrhT3gyMNbOm079FI20Ty3F6XRmfRtF7stRyN5udPGaz33jlJ\n/rBEsNybQiK8qyCNzZtQVYFG1C4SSI8GbO5Vk7cTSphhwDlsEKvJWuX+I36BWKts\nFPQwjI/ImIvmjdUKP1Y7XQ51AoGBALWa5Y3ASRvStCqkUlfFH4TuuWiTcM2VnN+b\nV4WrKnu/kKKWs+x09rpbzjcf5kptaGrvRp2sM+Yh0RhByCmt5fBF4OWXRJxy5lMO\nT78supJgpcbc5YvfsJvs9tHIYrPvtT0AyrI5B33od74wIhrCiz5YCQCAygVuCleY\ndpQXSp1RAoGBAKjasot7y/ErVxq7LIpGgoH+XTxjvMsj1JwlMeK0g3sjnun4g4oI\nPBtpER9QaSFi2OeYPklJ2g2yvFcVzj/pFk/n1Zd9pWnbU+JIXBYaHTjmktLeZHsb\nrTEKATo+Y1Alrhpr/z7gXXDfuKKXHkVRiper1YRAxELoLJB8r7LWeuIb\n-----END RSA PRIVATE KEY-----"
```

あとは [Client 側](https://github.com/mozilla/hubs)を構築するなどして、Hubs Cloud の開発を行うことができるので、自由にカスタマイズしましょう。

## Chef Habitat パッケージの作成

Reticulum のカスタマイズを終えたら、Hubs Cloud にデプロイするための Chef Habitat パッケージを作成します。
[Chef Habitat CLI](https://www.habitat.sh/docs/habitat-cli/) を利用してパッケージを作成し、[Chef Habitat Builder](https://bldr.habitat.sh/) にデプロイします。
Chef Habitat CLI をインストールするには環境に合わせた [Chef Habitat CLI のインストール方法](https://www.habitat.sh/docs/install-habitat/)を参照してください。

まず、Chef Habitat Builder にサインインします。
アカウントの作成は、[GitHub アカウントを使ったサインインページ](https://bldr.habitat.sh/#/sign-in)にアクセスして行います。

次に、`hab setup` コマンドで Builder の認証トークンを指定したり、デフォルトの Origin を指定したりします。

```sh
hab setup
```

以降の説明では例として、 Chef Habitat Builder Origin に `kou029w`、パッケージ名に `kou029w/reticulum` として説明します。

次のコマンドを実行し、Chef Habitat CLI を利用して Origin を作成します。

```sh
hab origin create kou029w
```

Reticulum リポジトリの `habitat/plan.sh` を書き換え、Chef Habitat Builder の Origin を指定します。

```diff:habitat/plan.sh
 pkg_name=reticulum
-pkg_origin=mozillareality
+pkg_origin=kou029w
 pkg_version="1.0.1"
 pkg_maintainer="Mozilla Mixed Reality <mixreality@mozilla.com>"
 pkg_upstream_url="http://github.com/mozilla/reticulum"
```

Origin を書き換えたらビルドします。

```shell
hab pkg build .
```

ビルドに成功すれば `results` ディレクトリ以下に `.hart` アーティファクトが生成されます。

### Chef Habitat Builder へのアップロード

生成された `.hart` アーティファクトを指定して `hab pkg upload` コマンドを実行します。

```shell
hab pkg upload --channel stable results/kou029w-reticulum-1.0.1-20201029030558-x86_64-linux.hart
```

以上で、Chef Habitat Builder にビルドしたパッケージをアップロードすることができました。

## Hubs Cloud の構築

[公式ドキュメント](https://hubs.mozilla.com/docs/hubs-cloud-aws-quick-start.html)に沿って Hubs Cloud AWS を構築します。

## EC2 インスタンスで実行されている Reticulum サービスの置換

Hubs Cloud の構築完了後、それぞれの EC2 インスタンスに対して繰り返し下記の作業を実施します。

### SSH でアクセス

[管理コンソールから [Server Access] を選択すると、アクセスするための 2 要素認証のための情報が得られます。](https://hubs.mozilla.com/docs/hubs-cloud-accessing-servers.html)

```shell
ssh -i <key file> ubuntu@<host name>
```

インスタンス名は、AWS マネジメントコンソールから確認できます。

### Chef Habitat パッケージのインストールと起動スクリプトの修正

以降の作業は SSH で EC2 インスタンスにアクセスした後の作業の実際のコマンドとその説明です。

```shell
# 稼働中のサービスを停止
sudo bio sup status | tail +2 | awk '$0=$1' | sudo xargs -n1 bio svc stop

# 作成した Chef Habitat パッケージのインストール
sudo bio pkg install kou029w/reticulum

# サービス mozillareality/reticulum を kou029w/reticulum に置換
sudo bio svc unload mozillareality/reticulum
sleep 10 # …アンロードするまで数秒待機
sudo bio svc load kou029w/reticulum

# 起動スクリプトの改変: kou029w/reticulum を読み込んで実行されるように変更
sudo perl -i -pe 's(\b(mozillareality/reticulum)\b)(kou029w/reticulum)g' /opt/polycosm/polycosm_boot.sh

# 起動スクリプトの改変: 起動スクリプト実行時のインスタンス名の変更に伴う証明書の更新対応
sudo perl -i -pe 's/^(ln -s)\b/ln -sf/' /opt/polycosm/polycosm_boot.sh

# 起動
sudo /opt/polycosm/polycosm_boot.sh aws aws
```

## いかがでしたか

以上です。
この記事では Mozilla Hubs のバックエンドサーバー Reticulum のカスタマイズ手順を紹介しました。Hubs Cloud の全体構成や Reticulum の構成など触れなかった部分についてはまた別の機会にでも書きたいと思います。
