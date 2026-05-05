---
title: "なぜ `/bin` フォルダは `bin` という名前なのか — 1971年のディスク事故から始まった話"
emoji: "🗄️"
type: "tech"
topics: ["linux", "unix", "filesystem", "history"]
published: false
---

<!--
================================================================================
ドラフト全体方針
================================================================================
独自性の出しどころ:
1. Rob Landley の 2010年メーリス投稿を一次情報として正確に引用
2. usrmerge という現代の動きを盛り込む（既存日本語記事の弱点）
3. 組み込み Linux / BusyBox の事情にも触れる（晴貴さん視点）

既存記事との差別化:
- 「bin = binary」だけで終わらせない
- /bin と /usr/bin の分離理由を「PDP-11のディスク事故」として正確に語る
- usrmerge で歴史が終わりつつあることを示す

構成案:
A. 名前の由来 → 分離の歴史 → 現代の usrmerge → 組み込みの事情
B. 現代の usrmerge から入る → 歴史を遡る → 組み込み
→ 採用: A（時系列が読みやすい）

注意: タイトルが「なぜ bin という名前か」なので、binary の話で導入してから
分離の話に展開する流れが自然。
================================================================================
-->

## はじめに

<!-- 採否判断: 入り方
     [選択肢A] 軽い疑問から入る
     [選択肢B] いきなり結論
     現案: A -->

Linux や macOS のターミナルで `ls /` すると、`/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/usr/local/bin` のように似たようなフォルダが並んでいます。なぜこんなに「bin」があるのか、そもそも「bin」って何の略なのか。

短い答えは「**binary（バイナリ実行ファイル）の bin**」です。でもそれだけでは「なぜ4つも5つもあるのか」「なぜ最近の Linux では `/bin` が `/usr/bin` のシンボリックリンクになっているのか」が説明できません。

実はこの構造の起源は **1971年のディスク容量不足という事故** です。本記事はそこから現代までの50年分の経緯を辿ります。

## 「bin」は binary の略

<!-- 採否判断: 必須セクション。短くてOK。 -->

実行ファイル（バイナリ）を置く場所、なので "bin"。Unix の初期から使われている命名で、現在の Linux や macOS、BSD など Unix 系 OS のほぼ全てで `/bin` というディレクトリ名を採用しています。

語源は単純ですが、**配置のルール** がディストリビューションをまたいで複雑になっており、ここから先が本題です。

## 1971年、PDP-11 で何が起きたか

<!-- 採否判断: ここが本記事の独自性の柱。
     一次情報として Rob Landley の 2010年12月のメーリングリスト投稿を引用する。
     URL: https://lists.busybox.net/pipermail/busybox/2010-December/074114.html
     内容を独自の言葉でまとめる。引用は短く、正確に。 -->

Unix の歴史を遡ると、1969年に Ken Thompson と Dennis Ritchie が PDP-7 上で Unix を作ったところから始まります。1971年頃、彼らは PDP-11 にアップグレードし、ストレージとして **RK05 ディスクパックを2台** 使うようになりました。1台あたりの容量はわずか 1.5 MB です [^landley]。

[^landley]: Rob Landley, "Understanding the bin, sbin, usr/bin, usr/sbin split" (busybox メーリングリスト, 2010-12-09) — https://lists.busybox.net/pipermail/busybox/2010-December/074114.html

OS が肥大化して 1台目のディスク（ルートファイルシステム）に収まらなくなったとき、彼らは2台目のディスクを **`/usr`** にマウントしました。ここはユーザーのホームディレクトリが置かれていた場所です。そして、1台目の `/bin`, `/sbin`, `/lib`, `/tmp` と同じディレクトリ構造を `/usr` の下にも作り、入りきらないファイルをそちらに書き始めました。

これが `/usr/bin` の起源です。**「ユーザー領域に紛れ込んだ OS バイナリ置き場」** がそのまま定着したものでした。

その後、3台目のディスクが導入されたときに、ユーザーのホームディレクトリは新設の `/home` に移されましたが、`/usr` の名前と用途はそのまま引き継がれました。だから `/usr` の語源は **user** であって、後付けで「Unix System Resources」と呼ばれるようになるのは更に後の話です。

<!-- 採否判断: ここで「user → Unix System Resources」という後付け解釈に触れるか。
     現案: 触れる。雑学として面白く、誤解を解く効果もある。 -->

## 「ブートに必要なものは / に、それ以外は /usr に」というルール

<!-- 採否判断: 分離の理由を後付けで合理化したルールを説明。
     一次情報: Rob Landley の同じメーリス投稿。 -->

ディスクを分けたことで、運用上の制約が生じました。**システム起動時、`/usr` をマウントするまでの間は `/usr` の下のコマンドを使えない**。だから `mount` コマンドのようなブート初期に必要なものは `/bin` に置く必要がありました。

このルールが後に **FHS (Filesystem Hierarchy Standard)** で正式な仕様として残ります [^fhs]。

[^fhs]: Filesystem Hierarchy Standard 3.0 — https://refspecs.linuxfoundation.org/FHS_3.0/fhs/

- `/bin`: シングルユーザーモードで使う基本コマンド
- `/sbin`: システム管理用コマンド (`s` は system 由来)
- `/usr/bin`: 一般のコマンド（パッケージ管理されているもの）
- `/usr/sbin`: 一般のシステム管理コマンド
- `/usr/local/bin`: 管理者がローカルにインストールしたコマンド
- `/usr/local/sbin`: 管理者がローカルにインストールしたシステム管理コマンド

しかし Landley は、このルール自体が **「もう意味を失っている」** と痛烈に書いています。

## なぜルールはもう意味がないのか

<!-- 採否判断: Landley の議論を3点に整理して紹介。
     原文の論点を正確に踏まえつつ、自分の言葉で書き直す。 -->

Landley が指摘した分離の不要さの根拠は3点あります。

1. **initrd / initramfs の登場**: 現代の Linux は `/usr` を含む実ファイルシステムをマウントする前に、メモリ上の初期ファイルシステム（initramfs）でブートを完結できる。「`/usr` をマウントする前に必要なもの」を分けておく理由が消えた
2. **共有ライブラリの登場**: `/lib` と `/usr/lib` を独立に更新することは事実上不可能。両者のバージョンが一致しないと動かないので、分けておく意味がない
3. **ハードドライブの大容量化**: 1990年頃から100MB超えが普通になり、容量不足でディスクを分ける必要は消えた

つまり **「1970年代の実装上の都合がそのまま受け継がれて来ただけ」** というのが Landley の主張です。

## usrmerge — 21世紀の答え

<!-- 採否判断: ここが既存日本語記事の弱点を突くポイント。
     usrmerge を扱った日本語記事は少数。差別化に効く。
     一次情報: 
     - https://www.freedesktop.org/wiki/Software/systemd/TheCaseForTheUsrMerge/
     - https://wiki.debian.org/UsrMerge -->

実際、近年のメジャーディストリビューションは `/bin` と `/usr/bin` の **分離をやめつつあります**。これを **usrmerge** または **/usr merge** と呼びます [^usrmerge]。

[^usrmerge]: "The Case for the /usr Merge" — https://www.freedesktop.org/wiki/Software/systemd/TheCaseForTheUsrMerge/

仕組みは単純で、`/bin`, `/sbin`, `/lib`, `/lib64` を `/usr/bin`, `/usr/sbin`, `/usr/lib`, `/usr/lib64` への **シンボリックリンク** にしてしまうというものです。物理的な実体は `/usr` 配下に統一されます。

```bash
$ ls -la / | grep -E 'bin|sbin|lib'
lrwxrwxrwx   1 root root    7 ... bin -> usr/bin
lrwxrwxrwx   1 root root    7 ... sbin -> usr/sbin
lrwxrwxrwx   1 root root    7 ... lib -> usr/lib
lrwxrwxrwx   1 root root    9 ... lib64 -> usr/lib64
```

採用時期は以下の通りです：

| ディストリ | 採用時期 |
|------------|----------|
| Fedora | 17 (2012) |
| Arch Linux | 2012 |
| openSUSE | 2014 |
| Debian | 12 Bookworm (2023) で必須化 |
| Ubuntu | Debian と同期 |
| Solaris | 2005年から段階的に、Solaris 11 で完了 |

実は Solaris は Linux より前に同じことをしていました。

<!-- 採否判断: Debian の usrmerge 移行が紛糾した話を入れるか。
     LWN.net の記事に詳しい (https://lwn.net/Articles/890219/)。
     [選択肢A] 1段落で簡単に触れる
     [選択肢B] 削除（本筋から外れる）
     現案: A -->

Debian の移行は2016年から始まり、長い議論の末2023年にようやく完了しました。dpkg メンテナの強い反対があったり、ビルドデーモンが破綻したりと、シンプルに見える変更が大規模ディストリビューションでは大きな技術的・社会的問題になることを示した事例として記憶されています。

## ところで `/sbin` の `s` は？

<!-- 採否判断: 副題的に sbin にも触れるか。
     "bin の名前" の記事として完結性を高めるためには触れた方が良い。
     [選択肢A] 入れる
     [選択肢B] 削除
     現案: A、ただしコンパクトに -->

`/sbin` の "s" は **system** または **superuser** の略です。FHS では「システム管理に必要なコマンドを置くディレクトリ」と定義されています。具体的には `init`, `mount`, `fsck`, `shutdown` などの、root 権限を前提とするコマンドが置かれます。

ただし usrmerge の流れの中で、Fedora は `/usr/sbin` も `/usr/bin` に統合する方向で動いています。「root 用のコマンドを別フォルダに置く」のも、PATH の通し方で制御できれば不要、というのが現代的な見解です。

## 組み込み Linux / BusyBox の世界

<!-- 採否判断: 組み込みエンジニア向けの差別化ポイント。
     晴貴さんの背景に近い内容。
     BusyBox は1つのバイナリが複数のコマンドを兼ねるという特殊な設計。 -->

組み込み Linux でよく使われる **BusyBox** は、別の方法でこの問題を解決しています。BusyBox は **単一の実行ファイル** が `ls`, `cat`, `mv`, `cp` など多数のコマンドの実体を兼ねており、それぞれのコマンド名は BusyBox バイナリへのシンボリックリンクとして配置されます。

```bash
$ ls -la /bin/ls /bin/cat /bin/cp
lrwxrwxrwx 1 root root 7 ... /bin/ls -> busybox
lrwxrwxrwx 1 root root 7 ... /bin/cat -> busybox
lrwxrwxrwx 1 root root 7 ... /bin/cp -> busybox
```

BusyBox の場合、`/bin` と `/usr/bin` の分離は単に **シンボリックリンクの置き場** の違いでしかなく、実体は1つです。Yocto や Buildroot で構築する組み込み Linux では usrmerge と似た考え方を、ディスク容量削減という別の動機から先取りしていたとも言えます。

## まとめ

`/bin` の "bin" は **binary** の略です。これは単純です。

複雑なのは `/bin`, `/usr/bin`, `/sbin`, `/usr/local/bin` の分離で、これは **1971年に Ken Thompson と Dennis Ritchie が PDP-11 で2台のディスクを使い分けた** という実装上の事故が、後付けの理屈で正当化されながら50年以上引き継がれた結果です。

そして 21世紀に入り、Solaris、Fedora、Arch、Debian など主要ディストリビューションは **usrmerge** によってこの分離を解消しつつあります。1971年の事故の影響が、ようやく終わろうとしているところです。

シェルで `cd /bin` と打つとき、その背景にはディスク容量と50年の物語があります。

## 参考文献

- Rob Landley, "Understanding the bin, sbin, usr/bin, usr/sbin split" (busybox ML, 2010) — https://lists.busybox.net/pipermail/busybox/2010-December/074114.html
- "The Case for the /usr Merge" — https://www.freedesktop.org/wiki/Software/systemd/TheCaseForTheUsrMerge/
- Debian UsrMerge — https://wiki.debian.org/UsrMerge
- Filesystem Hierarchy Standard 3.0 — https://refspecs.linuxfoundation.org/FHS_3.0/fhs/
- "Unexpected fallout from /usr merge in Debian" (LWN.net) — https://lwn.net/Articles/773342/

<!--
================================================================================
最終チェックリスト
================================================================================
- [ ] タイトルの「1971年のディスク事故」表現を残すか変えるか
      → 候補: "PDP-11のディスク容量問題", "50年前の制約"
- [ ] /sbin の節を入れるか
- [ ] BusyBox の節を入れるか（組み込み色を出すか）
- [ ] 自分のシステムで実際に ls -la / の出力を貼って、usrmerge されているか確認する
- [ ] Debian 紛糾の話の長さ（現案は1段落、もっと詳しく書く手も）
================================================================================
-->