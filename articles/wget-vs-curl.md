---
title: "wget と curl は何が違うのか - 数十億台に載るライブラリと、左手で打てるダウンローダ"
emoji: "🌀"
type: "tech"
topics: ["curl", "wget", "linux", "embedded", "docker"]
published: false
---

## はじめに

最近 Dockerfile を生成 AI に書いてもらっていると、wget と curl が毎回のように出てきます。両方入れておけば困らないことは分かっているのですが、改めて何が違うのか答えられないので、Claude に手伝ってもらいながら調べました。

```Dockerfile
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

## 検証環境

本記事のコマンド例は以下の Docker 環境で動かしています。誰でも同じ結果を再現できるはずです。

```Dockerfile
# Dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    ca-certificates \
    jq \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /work
```

```bash
docker build -t wget-vs-curl .
docker run --rm -it wget-vs-curl bash
```

`debian:bookworm-slim` を選んだのは、Ubuntu より小さく（圧縮後 約 30MB に対し Ubuntu は 約 80MB）、curl / wget の挙動を見るだけならこれで十分だからです。`alpine` (約 7MB) ならさらに軽量ですが、apt 系の操作に慣れている方が分かりやすいので Debian 系にしました。

検証時のバージョンは以下の通り：

```bash
$ curl --version | head -1
curl 7.88.1 (x86_64-pc-linux-gnu) libcurl/7.88.1 ...

$ wget --version | head -1
GNU Wget 1.21.3 built on linux-gnu.
```

## ざっくり比較

| 項目 | wget | curl |
|-----|------|------|
| ライセンス | GPL v3 | MIT (改変版) |
| 開発元 | GNU プロジェクト | 独立プロジェクト |
| 形態 | コマンドのみ | コマンド + ライブラリ (libcurl) |
| 対応プロトコル | HTTP(S), FTP, FTPS の3つ | 20以上 |
| 再帰ダウンロード | ◯ (看板機能) | ✗ |
| 双方向通信 | HTTP POST のみ | アップロード全般 |
| プロジェクト開始 | 1996年 (前身の Geturl 時代) | 1996年 (前身の httpget 時代、curl と命名されたのは1998年) |

出典: curl 作者 Daniel Stenberg の[公式ページ "curl vs Wget"](https://daniel.haxx.se/docs/curl-vs-wget.html)、および [GNU Wget 公式](https://www.gnu.org/software/wget/)。

## 設計思想の違い

### wget — Unix の `cp` のように

wget は **ファイルを取りに行く専用ツール** です。コマンド名は **"World Wide Web" + "get" (HTTP の GET メソッド)** に由来します [^wget-name]。設計思想を Daniel Stenberg は「Unix の `cp` コマンドに近い」と表現しています。`cp` がファイルをコピーする道具であるように、wget は URL からローカルへのコピーを担う道具です。

[^wget-name]: GNU Wget の Wikipedia 記事に "Its name derives from 'World Wide Web' and 'get', a HTTP request method." と明記されている。https://en.wikipedia.org/wiki/Wget

最大の特徴は **再帰的ダウンロード**。`-r` オプションで指定 URL からリンクを辿り、サイト全体をミラーリングできます。

```bash
# シンプルな再帰ダウンロード: 深さ1までリンクを辿る
$ wget -r -l 1 https://example.com/
--2026-05-04 12:34:56--  https://example.com/
Resolving example.com (example.com)... 93.184.216.34
Connecting to example.com (example.com)|93.184.216.34|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1256 (1.2K) [text/html]
Saving to: 'example.com/index.html'

example.com/index.html        100%[========>]   1.23K  --.-KB/s    in 0s

FINISHED --2026-05-04 12:34:57--
```

実用的には以下のようなオプションを組み合わせて使います：

```bash
$ wget --mirror \              # -N -r -l inf --no-remove-listing と同等（ミラーリング用）
       --convert-links \        # ローカル間のリンクに変換（オフライン閲覧用）
       --adjust-extension \     # .html / .css などの拡張子を補完
       --page-requisites \      # 画像・CSS など表示に必要なものも取得
       --no-parent \            # 親ディレクトリを辿らない
       https://example.com/
```

オプション名がそのまま意味になっていて、慣れると読みやすい設計です。

### curl — Unix の `cat` のように

curl は **データ転送のための汎用ツール** です。"curl" は "Client URL" の略。Stenberg はこちらを「Unix の `cat` に近い」と説明しています [^curl-cat]。標準入出力を介してデータを流すパイプ指向の道具です。

[^curl-cat]: https://daniel.haxx.se/docs/curl-vs-wget.html の「pipes: curl works more like the traditional Unix cat command」より。

```bash
# 例: GitHub API を叩いて、curl ユーザーの公開リポジトリ数を jq で抜く
$ curl -s https://api.github.com/users/curl | jq '.public_repos'
38
```

期待される出力は `curl` プロジェクトのリポジトリ数（変動するので執筆時点の値）。`-s` は silent mode で進捗表示を抑制し、パイプで `jq` に渡しやすくしています。

curl のもう一つの大きな特徴は、コマンドラインツールとは別に **`libcurl`** というライブラリを提供している点です。これが後述する組み込み機器への普及を可能にしました。

## libcurl はあなたの身の回りで動いている

curl の作者 Daniel Stenberg 自身の見積もりによれば、libcurl は世界に **数十億台** の規模でインストールされています [^billions]。これは大袈裟な表現ではなく、以下のような場所で実際に動いています。

[^billions]: 公式ページ https://daniel.haxx.se/my-name-in-products.html に "Products I helped create have been installed in billions of installations" と記載。なお、別のインタビュー [^interview] では "tens of billions" という更に大きな数字も挙げられているが、本記事では公式ページの記述に統一する。

[^interview]: SE Radio Episode 505 (2022) — https://se-radio.net/2022/03/episode-505-daniel-stenberg-on-25-years-with-curl/

| 場所 | 出典 |
|------|------|
| Windows 10/11 と macOS にプリインストール | curl 公式ページ ["curl vs Wget"](https://daniel.haxx.se/docs/curl-vs-wget.html) |
| iOS のクレジット画面 | Stenberg 公式ページの[スクリーンショット](https://daniel.haxx.se/my-name-in-products.html) |
| Android、Chrome OS の標準コンポーネント | [Almsec インタビュー](https://www.almsec.se/inside-security-17-interview-with-curl-creator-daniel-stenberg/) |
| PlayStation、Xbox、Nintendo Switch | [The New Stack 記事](https://thenewstack.io/the-creator-of-curl-remembers-23-wildly-successful-years/) |
| 主要メーカーの車載インフォテインメント | 同上、および本人の各種講演 |
| NASA Ingenuity (Mars 2020 ヘリコプター) | [Stenberg 公式ブログ](https://daniel.haxx.se/blog/2021/04/19/mars-2020-helicopter-contributor/) — GitHub の "Mars 2020 Helicopter Contributor" バッジから確認 |

最後の Ingenuity の件は、Stenberg 自身も 2021 年まで未確認と書いていました。GitHub が Mars 2020 ヘリコプタープロジェクトの貢献者バッジを配布したことで、curl が同プロジェクトのコードベースに含まれていたことが間接的に確認されたという経緯です。彼は冗談混じりに **「curl は2つの惑星で動いている」** と語っています。

組み込みエンジニアにとって libcurl が魅力的なのは、**C89 コンパイラさえあればビルドできる移植性** と、**11種類の SSL/TLS バックエンドから選べる柔軟性** です。MCU レベルでも mbedTLS や WolfSSL と組み合わせて HTTPS 通信を実現できます。

> ⚠️ ただし、この MCU レベルでの利用については、本記事の筆者は実際に試したわけではなく、Claude の説明を確認した範囲での記述です。実機での検証はまだ行っていません。

## ライセンスの違いが生んだ採用範囲

wget は **GPL v3**、curl は **MIT (改変版)** です。GPL v3 はソースコード開示義務があるため、クローズドソースの製品にバンドルすることが難しく、組み込み機器の世界では curl の方が圧倒的に採用されやすい。これも libcurl がここまで普及した一因です。

なお curl のライセンスは、Stenberg 自身が「初期に少しだけ MIT を改変したので厳密には MIT ではない」と[インタビュー](https://www.almsec.se/inside-security-17-interview-with-curl-creator-daniel-stenberg/)で語っています。実用上は MIT 同等として扱われます。

## wget の意外な特徴

ここまで curl 寄りの話が続きましたが、wget にも明確な強みがあります。

- **デフォルトが親切**: `wget URL` だけでファイルがローカルに保存される（curl は `-O` または `-o` が必要）
- **不安定な回線での再開機能**: `-c` オプションで中断したダウンロードを再開できる。curl にも `-C -` があるが、wget の方が再開挙動が堅牢と評価されている
- **再帰ダウンロードは唯一無二**: curl にはこの機能がない
- **左手だけでタイプできる**: QWERTY キーボードで w-g-e-t は全て左手の範囲 [^left-hand]

[^left-hand]: Daniel Stenberg の[公式ページ](https://daniel.haxx.se/docs/curl-vs-wget.html) にも書かれている、Stenberg 公認のジョーク。

サイト全体をアーカイブしたい、不安定な接続で巨大ファイルを取りたい、というケースでは wget の方が向いています。

## 最近の動向 — wcurl と Wget2

### wcurl — curl プロジェクトが作った「wget 風 curl」

curl は便利な反面、`-o` や `-O` を付けないとファイルとして保存されないなど、初心者には不親切な面があります。これを解決するため curl プロジェクト自身が **wcurl** という薄いラッパーを 2024 年にリリースしました [^wcurl]。`wcurl URL` だけでファイルが保存できます。curl 8.14.0 以降は curl のリリース tarball にバンドルされています。

[^wcurl]: https://curl.se/wcurl

### Wget2 — wget の次世代版

GNU プロジェクトでは **Wget2** が wget の後継として開発されています [^wget2]。HTTP/2 対応、並列ダウンロード、libwget としてのライブラリ提供など、現代的な機能を備えています。ディストリビューションによっては既にパッケージ化されているので、`apt install wget2` などで試せます。

[^wget2]: https://gitlab.com/gnuwget/wget2

## まとめ

雑にまとめると以下のような使い分けになります。

- **API を叩く / 単発のリクエストを試す**: curl
- **サイト全体をオフライン保存**: wget
- **不安定な回線で巨大ファイルをダウンロード**: wget
- **スクリプトに組み込む / ライブラリとして使う**: curl (libcurl)
- **Windows でどっちかしか入っていない**: 多くの場合 curl（プリインストール）

両者は競合というより補完関係にあります。Stenberg 自身、curl 公式ページに「`curl` という名前を使った wget の互換クローンが BusyBox にある」と書いていますが、これは皮肉ではなく事実の記述です。組み込み Linux の世界では BusyBox の wget で十分なことが多く、libcurl をフルで載せるのは贅沢な選択になります。

## 参考文献

- Daniel Stenberg, "curl vs Wget" — https://daniel.haxx.se/docs/curl-vs-wget.html
- Daniel Stenberg, "My name in products" — https://daniel.haxx.se/my-name-in-products.html
- Daniel Stenberg, "Mars 2020 Helicopter Contributor" — https://daniel.haxx.se/blog/2021/04/19/mars-2020-helicopter-contributor/
- curl 公式 — https://curl.se/
- GNU Wget 公式 — https://www.gnu.org/software/wget/
- GNU Wget Wikipedia — https://en.wikipedia.org/wiki/Wget
- wcurl — https://curl.se/wcurl
- Wget2 — https://gitlab.com/gnuwget/wget2
- "10 Billion Devices Run His Code" — Medium / Can Artuc
- SE Radio Episode 505: Daniel Stenberg on 25 years with cURL — https://se-radio.net/2022/03/episode-505-daniel-stenberg-on-25-years-with-curl/