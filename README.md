# blog 運用手順

Quarto + Cloudflare Pages で運用しているブログ。公開先は <https://corocorohashioka.pages.dev>

GitHub は**バックアップ専用**で、公開とは無関係。`git push` しても公開内容は変わらない。

---

## 1. ディレクトリ構成

```
blog/
├── posts/                        # 記事本体（.qmd）
│   └── 2026-articles/
│       └── 0112_ontake/index.qmd
├── materials/                    # 写真・データ（posts/ と同じ階層構造）
│   ├── 2026-articles/
│   │   └── 0112_ontake/
│   │       ├── Ontake.gpx
│   │       └── ontake_point.geojson
│   └── shared/                   # 複数記事で使い回すデータ
│       └── shichoson/2019_isna/
├── docs/                         # quarto render の出力（自動生成・git 管理外）
└── _quarto.yml
```

`materials/` は `posts/` の階層をミラーする。記事 `posts/2026-articles/0112_ontake/` に対応する素材は
`materials/2026-articles/0112_ontake/` に置く。

---

## 2. 写真とデータの参照方法

**先頭の `/` の有無だけが違う。** どちらもプロジェクトルート（`blog/`）が起点。

| 用途 | 書き方 | サイトに公開されるか |
|---|---|---|
| 写真 | `![説明](/materials/2026-articles/0112_ontake/photo.jpg)` | **される** |
| データ | `read_sf("materials/2026-articles/0112_ontake/Ontake.gpx")` | **されない** |

```markdown
![剣ヶ峰からの眺め](/materials/2026-articles/0112_ontake/photo.jpg){width=50% fig-align="center"}
```

```r
line_data <- read_sf("materials/2026-articles/0112_ontake/Ontake.gpx", layer = "tracks")
```

理由: Quarto が `docs/` にコピーするのは Markdown から参照されたリソースだけ。R が読むファイルは
計算への入力でしかないため、公開されるのは計算結果（地図・グラフの HTML）のみ。
218MB のシェープファイルもアップロードされない。

R のパスに `/` を付けないのは、`_quarto.yml` の `execute-dir: project` により
R チャンクの作業ディレクトリがプロジェクトルートになっているため。

---

## 3. 記事を書いて公開する（メイン Mac）

```bash
cd ~/Documents/blog
quarto render
npx wrangler pages deploy docs --project-name=corocorohashioka --branch=main --commit-dirty=true
git add -A && git commit -m "記事を追加" && git push
```

1. `posts/2026-articles/<記事名>/index.qmd` を作る
2. 写真は `materials/2026-articles/<記事名>/` に置く
3. 上記3コマンドを実行

`quarto render` と `wrangler pages deploy` が**公開の本体**。`git push` はバックアップであり、
やらなくても公開はされる（ただしソースが失われるのでやること）。

---

## 4. git に入るもの・入らないもの

| git 管理下 | git 管理外（`.gitignore`） |
|---|---|
| `*.qmd`、`_quarto.yml`、`styles.css`、この README | `docs/`（出力）、`_freeze/`（キャッシュ） |
| | 画像 `*.jpg *.png *.gif *.webp *.heic` |
| | 地理データ `*.gpx *.geojson *.shp *.shx *.dbf *.prj` |

GitHub の 1リポジトリ 1GB 制限を避けるため、容量の大きいものは一切入れていない。

> **写真とデータのバックアップは git 任せにできない。** Time Machine や iCloud など別手段で確保すること。

---

## 5. 別デバイスで作業する

```bash
git clone https://github.com/corocorohashioka/blog.git
cd blog
# .qmd を編集
git add -A && git commit -m "本文を修正" && git push
```

clone されるのは `.qmd` などテキスト10ファイルのみ。**文章の編集はこれで完結する。**

### 別デバイスでできないこと

| 操作 | 可否 | 理由 |
|---|---|---|
| `.qmd` の文章を書く・直す | できる | テキストは全部入っている |
| `quarto render`（サイト全体） | **できない** | ontake がデータを見つけられずエラーになり、HTML が1枚も生成されない |
| ontake の記事を個別 render | **できない** | GPX・シェープファイルが無い |
| 外部データを使わない記事を個別 render | できる | `quarto render posts/.../index.qmd` |
| 写真入り記事の render | 注意 | **エラーにならず画像だけ壊れる**。`<img>` タグは出力されるが実体が無い |
| Cloudflare へのデプロイ | **やらないこと** | 画像やデータが欠けた `docs/` を公開してしまう |

### 推奨フロー

```
別デバイス: 文章を編集 → push
        ↓
メイン Mac: pull → quarto render → wrangler deploy
```

**公開は必ずメイン Mac から行う。** 写真・データ・レンダリング環境が揃っているのはメイン Mac だけ。

---

## 6. 注意点

- **ontake の記事は編集すると再実行される。** `freeze: true` はソースが変わると再実行するため、
  本文を1文字直しただけでも GPX とシェープファイルの読み込みが走る
- **写真は差し替えても `git status` に出ない。** gitignore 対象のため、変更の追跡は git ではできない
- **Cloudflare の制限**: 1ファイル 25 MiB、1デプロイ 20,000 ファイル。リポジトリ容量の上限は無い
- **存在しない URL はトップページが返る**（404 ではなく 200）。リンク切れの確認は目視で
