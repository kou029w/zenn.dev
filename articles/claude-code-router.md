---
title: Claude APIを使わず無料でClaude Codeを使う
emoji: 🤖
topics:
  - claude
  - claudecode
type: tech
published: true
---

## はじめに

[Claude Code](https://www.anthropic.com/claude-code) 使っとりますか。
最近ワタシも使い始めたとこです。

Claude Code の利用には $20/月〜 Pro プラン (年払いなら $200/年) に加入するか、Claude API キーを課金する必要あります。
ワタシは小心者なので正直この請求金額にビビっていました。定額ならまだしも従量課金でついうっかり制限忘れて破産するんじゃないかという漠然とした不安感みたいなものがあります。
同じように躊躇してしまった方もいるのではないでしょうか。

そんなわけで試しに使ってみたのが [Claude Code Router](https://github.com/musistudio/claude-code-router)。Claude Code Router は Claude Code からあらゆる OpenAI 互換 API を利用できるようにする、そういうオープンソースプロジェクトです。

https://github.com/musistudio/claude-code-router

詳しくはソースコードを読んでもらうとして、つまり Claude Code が使用する Anthropic 独自フォーマットの API を中継してあらゆるモデルを自由に使うことができる、そういう手段が Claude Code にあるということです。

## 使い方

Claude Code のインストール:

```sh
npm install -g @anthropic-ai/claude-code
```

Claude Code Router をインストール:

```sh
npm install -g @musistudio/claude-code-router
```

設定ファイル `~/.claude-code-router/config.json` を配置:

```json
{
  "Providers": [
    {
      "name": "gemini",
      "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
      "api_key": "ここにAPIキー",
      "models": ["gemini-2.5-flash", "gemini-2.5-pro"],
      "transformer": {
        "use": ["gemini"]
      }
    }
  ],
  "Router": {
    "default": "gemini,gemini-2.5-flash",
    "background": "gemini,gemini-2.5-flash",
    "think": "gemini,gemini-2.5-pro",
    "longContext": "gemini,gemini-2.5-flash"
  }
}
```

Gemini の API キー:

1. https://aistudio.google.com/apikey にアクセス (Google アカウントでのログイン)
2. 「Get API key」「API キーを作成」をクリック
3. `ここにAPIキー` の部分に貼り付け

これで無料の Gemini API を利用できる。

Claude Code Router を介して Claude Code を実行:

```
$ ccr code
> /model gemini,gemini-2.5-flash
```

内部的には:

1. ローカルで API サーバー (`http://127.0.0.1:3456`) を起動
2. 環境変数 `ANTHROPIC_BASE_URL=http://127.0.0.1:3456` に設定
3. `claude` コマンドを実行

…という動作をしているようです。

## おわりに

Claude Code Router、Gemini API の他にも Ollama など OpenAI 互換 API であれば自由に使えるようです。
実用的には Claude API の品質にはまだ及ばないものの Claude Code をカスタマイズするアイデアの一つとして面白いなーと思いました。

確認したバージョン:

```
$ claude --version
1.0.48 (Claude Code)
$ ccr -v
claude-code-router version: 1.0.15
```

参考: https://egoist.dev/claude-code-free
