---
title: Web GPIO API/Web I2C APIをRaspberry PiのNode.jsから使う
emoji: 🤖
topics: [nodejs, raspberrypi, chirimen]
type: tech
published: true
---

## 概要

Raspberry Pi の Node.js から Web GPIO API と Web I2C API を扱う方法を説明します。

:::message
[Web GPIO API](http://browserobo.github.io/WebGPIO/)/[Web I2C API](http://browserobo.github.io/WebI2C/) とは [W3C の Browsers and Robotics コミュニティグループ](https://www.w3.org/community/browserobo/)で標準化に向けての検討されている、JavaScript で Web アプリから電子パーツを直接制御できる低レベルハードウェア制御 API です。
:::

:::message
[Node.js](https://nodejs.org/) は、オープンソース・クロスプラットフォームな JavaScript 実行環境です。
[パッケージマネージャー npm](https://www.npmjs.com/) を利用して膨大なパッケージにアクセスでき、IoT プロトタイピングだけでなく幅広いアプリケーションを作るために使われています。
:::

## 準備

プログラムを実行するには Raspberry Pi に Node.js をインストールします。CHIRIMEN を利用する場合はあらかじめ Node.js がインストールされているので不要です。
もし CHIRIMEN の microSD カードを作成する方法を知りたい場合は [CHIRIMEN の SD カードイメージの作成方法](https://tutorial.chirimen.org/raspi/sdcard)を参照してください。

:::message
[CHIRIMEN](https://chirimen.org/) は、Web ブラウザからハードウェアを制御するプロトタイピング環境です。
:::

## Raspberry Pi に Node.js をインストールする方法

いくつか方法はありますが、ここでは [NodeSource を使ったインストール方法](https://github.com/nodesource/distributions#installation-instructions) を紹介します。

ターミナルを起動して以下のコマンドを実行します。

```sh
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
```

```sh
sudo apt-get install -y nodejs
```

これで Node.js のインストールは完了です。

## 新しいディレクトリの作成

続いて、実際にアプリケーションを作る前にプログラムを実行するための環境を整えます。作業用のディレクトリを作り、そのディレクトリの中でプログラムを実行します。

ターミナルを起動して以下のコマンドを実行します。

```shell
mkdir hello-real-world
```

```shell
cd hello-real-world
```

npm のためのファイル package.json を作成します。

```shell
npm init -y
```

作業用のディレクトリの中に npm パッケージ [node-web-gpio](https://www.npmjs.com/package/node-web-gpio) と [node-web-i2c](https://www.npmjs.com/package/node-web-i2c) をインストールします。

```sh
npm install node-web-gpio node-web-i2c
```

これで Node.js から Web GPIO API と Web I2C API を使う準備は完了です。

## Hello Real World

Raspberry Pi に接続した LED を点滅させるプログラムを書きます。

次の図のとおりに配線します。

![LEDの点滅回路の配線図](https://i.imgur.com/419yMxN.jpg)

空のテキストファイル main.js を作成し、Node.js のための JavaScript のプログラムを書きます。

```sh
editor main.js
```

テキストエディターで main.js を次のように書きます。

```js
const { requestGPIOAccess } = require("node-web-gpio");
const sleep = require("util").promisify(setTimeout);

async function blink() {
  const gpioAccess = await requestGPIOAccess();
  const port = gpioAccess.ports.get(26);

  await port.export("out");

  for (;;) {
    await port.write(1);
    await sleep(1000);
    await port.write(0);
    await sleep(1000);
  }
}

blink();
```

書き終えたら保存します。

Node.js で main.js を実行するには、次のコマンドを実行します。

```sh
node main.js
```

LED が点滅すれば完成です 🎉

## いろいろなデバイスを試す

CHIRIMEN ブラウザーから利用できるいろいろなデバイスはすべて同じように Node.js から扱うことができます。

たとえば、次のコードは[温度センサー ADT7410](http://akizukidenshi.com/catalog/g/gM-06675/)を利用して温度を表示するプログラムです。

```js
const { requestI2CAccess } = require("node-web-i2c");
const ADT7410 = require("@chirimen/adt7410");

async function measure() {
  const i2cAccess = await requestI2CAccess();
  const i2c1 = i2cAccess.ports.get(1);
  const adt7410 = new ADT7410(i2c1, 0x48);
  await adt7410.init();
  const temperature = await adt7410.read();
  console.log(`Temperature: ${temperature} ℃`);
}

measure();
```

コマンド `npm i @chirimen/adt7410` を実行すると、温度センサー ADT7410 を利用するための `@chirimen/adt7410` パッケージをインストールできます。

デバイスを扱うためのドライバーのパッケージの一覧は [CHIRIMEN Drivers](https://github.com/chirimen-oh/chirimen-drivers) にありますのでご覧ください。

また、CHIRIMEN チュートリアルのなかには、Web GPIO や Web I2C によって扱うことのできる[外部デバイスとサンプルコードの一覧があります](https://tutorial.chirimen.org/raspi/partslist)。こちらも参考になるかもしれません。

## まとめ

CHIRIMEN ブラウザー版での API の差異をまとめると下記とのおりです。

| CHIRIMEN ブラウザー版         | Node.js                                                                       |
| ----------------------------- | ----------------------------------------------------------------------------- |
| `navigator.requestGPIOAccess` | `const { requestGPIOAccess } = require("node-web-gpio");` `requestGPIOAccess` |
| `navigator.requestI2CAccess`  | `const { requestI2CAccess } = require("node-web-i2c");` `requestI2CAccess`    |
| `sleep`                       | `const sleep = require("util").promisify(setTimeout);` `sleep`                |

以上です。
この記事では、Raspberry Pi の Node.js から Web GPIO API と Web I2C API を扱う方法を説明しました。
