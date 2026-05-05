---
title: "なぜ `--version` と `-V` と `/V` があるのか — CLIオプション記法の系譜"
emoji: "🪶"
type: "tech"
topics: ["cli", "linux", "powershell", "posix", "c"]
published: false
---

<!--
================================================================================
ドラフト全体方針
================================================================================
独自性の出しどころ:
1. getopt() / getopt_long() の C 実装に踏み込む（晴貴さんの組み込み視点）
2. PowerShell の単一ダッシュ + 長オプションという「異質な」記法を歴史的に位置付ける
3. -V と -v の衝突実例を集める
4. POSIX / GNU / BSD / DCL の4系統で整理する

既存記事との差別化:
- 「-はPOSIX、--はGNU」で終わらせない
- 一次情報（POSIX 12.2、GNU Coding Standards）を出典として明示
- C のソースレベルで「実際に何が起きているか」を示す

構成案:
A. 歴史→各系統の解説→getopt実装→現代の混乱
B. 現代の混乱から入る→歴史を遡る
C. 系統別に並列解説→歴史で締める
→ 採用: A（時系列が読みやすい）
================================================================================
-->

## はじめに

<!-- 採否判断: 導入の入り方
     [選択肢A] 読者を疑問で引き込む（"これらは全部同じ意味なのに、なぜ書き方が違うのか？"）
     [選択肢B] いきなり結論
     現案: A -->

```bash
$ python --version
$ python -V
$ git --version
$ Get-Host | Select-Object Version  # PowerShell
```

これらは全部「バージョンを表示する」操作ですが、書き方が全然違います。`-V` と `-v` がツールによって意味が逆だったり、PowerShell では単一ハイフンで長いパラメータ名を書いたり。

これらの違いは思いつきではなく、**4つの異なる規約**が並立した結果です。本記事ではその系譜を整理します。

## 4つの系統

<!-- 採否判断: 概観の表を冒頭に置くかどうか。
     ここで全体像を見せておくと、以降の各論が頭に入りやすい。 -->

| 系統 | 短オプション | 長オプション | 代表例 |
|------|--------------|--------------|--------|
| POSIX / Unix | `-v` | （無し） | `ls -l`, `cat -n` |
| GNU 拡張 | `-v` | `--version` | `ls --color=auto` |
| BSD | `v`（ハイフン無し） | （無し） | `ps aux`, `tar xvf` |
| DOS / VMS DCL | `/V` | `/VERSION` | `dir /B` |
| PowerShell | `-V` | `-Version` | `Get-Process -Name foo` |

順に見ていきます。

## POSIX — 単一ハイフン + 1文字

<!-- 採否判断: POSIXの規定を一次情報から正確に引用する。
     出典: https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html
     "Utility Conventions" セクション 12.1, 12.2 -->

最も古典的な規約です。POSIX.1-2017 の "Utility Conventions" (12.1〜12.2 章) で正式に定義されています [^posix]。要点は以下の通り：

[^posix]: https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html

- オプションは `-` の後に **英数字1文字**
- 引数を取らないオプションは結合できる: `-abc` ≡ `-a -b -c`
- `--` は **オプションの終わり** を示す（これ以降はオプションとして解釈しない）

最後の `--` は地味に重要です。例えば `-foo` という名前のファイルを削除したいとき：

```bash
$ rm -foo                # エラー: 未知のオプション
$ rm -- -foo             # OK: -foo はファイル名として扱われる
```

この `--` の存在が「ハイフン2つ」記法の本来の起源で、長オプション `--version` は後発です。

<!-- 採否判断: ここで「-- が長オプションより先にあった」という事実を強調するか。
     POSIX 文化を知らない読者には新鮮な情報。差別化ポイント。 -->

## GNU 拡張 — ダブルハイフン + 単語

<!-- 出典: https://www.gnu.org/prep/standards/html_node/Command_002dLine-Interfaces.html
     GNU Coding Standards "Command-Line Interfaces" セクション。
     ここに「全てのプログラムは --version と --help をサポートすべき」と書かれている。 -->

GNU プロジェクトは 1980年代後半から、POSIX の単一文字オプションに加えて **長オプション** を導入しました。`--verbose`, `--output`, `--help` のように、人間が読める単語をハイフン2つで前置します。

GNU Coding Standards [^gnu] には次のように明記されています：

> All programs should support two standard options: '--version' and '--help'.

[^gnu]: https://www.gnu.org/prep/standards/html_node/Command_002dLine-Interfaces.html

なぜ単一ハイフンではなく **ダブル** ハイフンなのか。理由は POSIX との互換性です。`-verbose` と書くと、POSIX の慣習では `-v -e -r -b -o -s -e` という7つの短オプションの結合と区別がつかない。だから GNU は `--` を使って曖昧さを解消したわけです。

<!-- 採否判断: この「曖昧さ解消が動機」という説明は、
     既存記事では言われていても理由まで踏み込んでいない場合が多い。
     差別化に効く。 -->

## BSD スタイル — ハイフン無し

<!-- 採否判断: BSDスタイルに触れるか。
     現代の若い読者は ps aux の "aux" が何故ハイフン無しか知らない人が多い。
     雑学的価値はある。 -->

`ps aux`、`tar xvf archive.tar` のように、ハイフンを付けずにオプションを書く流派があります。これは BSD 系 Unix の伝統で、POSIX 以前の古い記法です。

現在は POSIX 互換のため `ps -aux` や `tar -xvf` も受け付けるツールが多いですが、両者は微妙に意味が違うことがあります。例えば Linux の `ps aux` は BSD 構文として解釈されますが、`ps -aux` は POSIX 構文として「ユーザー aux のプロセスを表示」と読まれる可能性があり、警告を出す実装もあります。

## DCL と PowerShell — `/` から `-` へ

<!-- 採否判断: ここが本記事の差別化ポイントの一つ。
     PowerShell の単一ダッシュ + 長オプション記法の歴史的位置を整理。
     ただし「PowerShellの -Param が DCL 由来」と断定するのは要注意。
     DCL は / を使う。PowerShellは - だが verb-noun の哲学はDCLっぽい。
     正確に書く: PowerShellの設計思想（verbose, verb-noun）は DCL 系譜だが、
     構文の - は DOS/Unix 折衷。 -->

DEC の VMS 上で動いていた **DCL (DIGITAL Command Language)** は、`/QUALIFIER` というスラッシュ前置の修飾子記法を使っていました [^dcl]。

[^dcl]: https://en.wikipedia.org/wiki/DIGITAL_Command_Language

```
$ DIRECTORY /SIZE /DATE
```

DOS / Windows のコマンドプロンプトもこの流れを汲み、`dir /B`, `xcopy /S /E` のようにスラッシュを使います。

PowerShell はこの伝統を受けつつも、Unix 系との折衷を図りました。**単一ハイフン + 長いパラメータ名** という独特の記法です：

```powershell
Get-ChildItem -Path C:\ -Recurse -Filter *.log
```

なぜスラッシュではなくハイフンなのか。Unix 文化との親和性、およびスラッシュが Windows のパス区切り文字と紛らわしいため、というのが定説です。ただし設計思想の根は DCL にあり、**動詞-名詞のコマンド名** (`Get-ChildItem`、`Set-Location`) や **冗長で読みやすいパラメータ名** はそのまま DCL の哲学を引き継いでいます。

<!-- 採否判断: PowerShell の - について踏み込みすぎると DCL 議論が長くなる。
     [選択肢A] 上記程度でサラッと
     [選択肢B] DCL 文化の解説をもう少し
     現案: A -->

## `-v` と `-V` 戦争

<!-- 採否判断: ここは小ネタとして楽しい節。
     既存記事ではあまり集約されていない。差別化に効く。 -->

長オプション `--version` は GNU Coding Standards で定まっていますが、短オプションは現場で混乱しています。

| ツール | `-v` | `-V` |
|--------|------|------|
| GCC | verbose | （無し） |
| Python | verbose | バージョン |
| Java | （無し） | バージョン |
| ssh | verbose（重ねがけで -vvv も可） | （無し） |
| curl | verbose | バージョン |
| less | バージョン | バージョン |
| grep | バージョン | （無し） |

<!-- 上記の表は要検証。執筆時に各ツールの --help を実際に走らせて確認すること。
     現バージョンで一覧化する必要あり。 -->

「`-v` が verbose、`-V` が version」というのが GNU 由来のなんとなくの慣習ですが、Python のように `-V` がバージョンを示すツールも多数あります。これは Python が古くから「`-v` は verbose のために予約」する文化を持っていたためです。

教訓: **本番スクリプトで `-V` か `-v` か迷ったら、`--version` を使う方が安全**。

## C の `getopt()` を読んでみる

<!-- 採否判断: ここが組み込みエンジニア向けの差別化最大ポイント。
     getopt のソースを読む or 簡単な使用例を書く or 自前実装してみる。
     [選択肢A] glibc の getopt.c の核を抜粋して解説（深い）
     [選択肢B] getopt() の使用例を書いて解説（中庸）
     [選択肢C] 自前で簡単な argc/argv パーサを書いてみる（実践的）
     現案: B + C のミックス -->

CLI オプションの解析は、C 言語標準ライブラリの `getopt()` で行うのが伝統的です。POSIX が標準化しています。

```c
// snippets/getopt-demo/main.c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    int opt;
    int verbose = 0;
    char *output = NULL;

    while ((opt = getopt(argc, argv, "vo:")) != -1) {
        switch (opt) {
            case 'v':
                verbose = 1;
                break;
            case 'o':
                output = optarg;
                break;
            default:
                fprintf(stderr, "Usage: %s [-v] [-o file]\n", argv[0]);
                return 1;
        }
    }
    printf("verbose=%d, output=%s\n", verbose, output ? output : "(stdout)");
    return 0;
}
```

`"vo:"` という文字列がポイントです。文字 `v` は引数を取らないオプション、`o:`（コロン付き）は引数を取るオプションを意味します。`-o file.txt` も `-ofile.txt` も同じように処理されます。

長オプションを扱うには GNU 拡張の `getopt_long()` が必要です。これは POSIX には含まれません。

```c
#include <getopt.h>

static struct option long_options[] = {
    {"verbose", no_argument,       0, 'v'},
    {"output",  required_argument, 0, 'o'},
    {0, 0, 0, 0}
};

// getopt_long(argc, argv, "vo:", long_options, NULL) で解析
```

組み込み開発で newlib などの軽量 libc を使う場合、`getopt_long()` が無いことがあります。その場合は自前で argv をループするか、argtable や CLI11 のような外部ライブラリを使うことになります。

<!-- 採否判断: 自前パーサの例を書くか。
     書くと記事が長くなるが、組み込み読者には喜ばれる。
     [選択肢A] 削除（記事を短く）
     [選択肢B] snippets/ に置いて記事からはリンクのみ
     [選択肢C] 記事内に書く
     現案: B -->

自前で書く場合のサンプルは `snippets/getopt-demo/` に置いておきます。

## 余談 — `-` 単独はなんなのか

<!-- 採否判断: 余談として面白いネタ。
     入れると記事の幅が出るが、本筋からは外れる。
     [選択肢A] 入れる（雑学的価値）
     [選択肢B] 削除
     現案: A、ただしコンパクトに -->

`tar -czf - .` のように `-` をファイル名の位置に書くと、多くの Unix ツールで **標準入出力** を意味します。これは POSIX で明確に規定されているわけではなく、ツール毎の慣習です。`tar`, `cat`, `gzip`, `ssh` などほとんどの主要ツールがサポートしています。

```bash
# ローカルの tar をリモートに転送
tar cf - mydir | ssh remote 'tar xf - -C /target'
```

## まとめ

CLI のハイフン記法は **歴史の地層** です。

- **`-v`**: POSIX / Unix の伝統（1970年代から）
- **`--verbose`**: GNU の拡張（1980年代後半から）
- **`v` （ハイフン無し）**: BSD の遺産
- **`/V`**: DEC VMS / DOS の系譜
- **`-Verbose`**: PowerShell（DCL の哲学を - で表現した独自折衷）

新しくCLIツールを書くなら、**POSIX + GNU 形式** に従うのが最も無難です。Python なら `argparse`、Rust なら `clap`、Go なら `flag` か `cobra` が、これらの慣習をデフォルトでサポートしています。

## サンプルコード

`posts/cli-dash-options/snippets/getopt-demo/`

<!-- TODO: GitHub のパーマリンクに差し替え -->

## 参考文献

- POSIX.1-2017 Utility Conventions — https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html
- GNU Coding Standards: Command-Line Interfaces — https://www.gnu.org/prep/standards/html_node/Command_002dLine-Interfaces.html
- GNU Coding Standards: Option Table — https://www.gnu.org/prep/standards/html_node/Option-Table.html
- DIGITAL Command Language (Wikipedia) — https://en.wikipedia.org/wiki/DIGITAL_Command_Language
- "Conventions for Command Line Options" by Chris Wellons — https://nullprogram.com/blog/2020/08/01/

<!--
================================================================================
最終チェックリスト
================================================================================
- [ ] -v / -V の表は実際に各ツールを走らせて確認すること
- [ ] PowerShell セクションの DCL 言及の表現を再確認（"由来" と書くと厳密には誤り）
- [ ] getopt のサンプルコードを実機で確認
- [ ] BSD スタイルの節を残すか削るか
- [ ] 余談（- 単独）を残すか削るか
================================================================================
-->