# tech-articles

技術記事の執筆用リポジトリ。

## 構成

- `articles/` — 公開用 Markdown（Zenn の規約に合わせた配置）
- `posts/<記事名>/` — 記事ごとの作業フォルダ
  - `README.md` — 調査メモ・参考リンク
  - `draft.md` — 下書き
  - `snippets/` — 記事内で使うコードの検証用

## 運用

1. `posts/<記事名>/` で調査・執筆
2. 完成したら `articles/<記事名>.md` にコピー
3. 記事末尾に `posts/<記事名>/snippets/` への GitHub リンクを記載

## 記事一覧

- [x] wget-vs-curl
- [ ] cli-dash-options
- [ ] why-bin-folder
- [ ] trailing-comma
