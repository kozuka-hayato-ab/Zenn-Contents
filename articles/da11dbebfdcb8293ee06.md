---
title: "Unity PackageManagerでインストール可能なtarball(tgz)ファイルを作成する"
emoji: "♨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Unity, PackageManager, tgz, tarball]
published: true
---

## はじめに
今回の記事は備忘録です。
PackageManagerにて、自作のtgzファイルが読み取れず、苦労したので記録しておきます。

## 概要
今回の目標としてはpackage.jsonを含むディレクトリをtgzに圧縮し、PackageManagerから読み取ることです。packageの作り方に関しては今回やりたいことではないのでお手数ですが、別記事を参照願います。

## 手順
次のようなディレクトリを圧縮することを想定します
```圧縮対象のディレクトリ
Target
├── Editor
│   ├── Hoge.cs
│   └── Hoge.cs.meta
├── Editor.meta
├── README.md
├── README.md.meta
├── package.json
└── package.json.meta
```

Targetディレクトリ内で`$ npm pack`を実行します。
package.json内に定義された`名前`-`バージョン`でtgzファイルが生成されます。
Unity PackageManagerから生成されたtgzファイルを読み込むことで使用できるようになりました。

## あとがき
自分は
**[Targetディレクト内]**: package.jsonの置いてある箇所
**[npm pack]**: tgzの圧縮コマンド
の2点で詰まってました。忘れた時、検索でひっかかることを願っています。
