# pr-tree

PR の親子関係をツリー表示する CLI ツールです。

ブランチではなく **PR をノードの主体** とするため、ブランチがマージ・削除された後もツリーが断絶しません。

## 動作条件

| 依存 | バージョン | インストール方法 |
|------|-----------|----------------|
| Python 3 | 3.6+ | macOS 標準搭載 |
| git | - | 通常インストール済み |
| GitHub CLI (`gh`) | 2.0+ | `brew install gh` |

### gh の認証

```bash
gh auth login
```

## インストール

path の通ったディレクトリにシンボリックリンクを配置して下さい。

```bash
# ~/bin に配置する場合
ln -sf "$(pwd)/pr-tree" ~/bin/pr-tree
```

## 使い方

```bash
# カレントブランチのツリーを表示
pr-tree

# ブランチを指定
pr-tree <branch>

# HTML を生成してブラウザで開く
pr-tree --open [branch]

# ルートブランチを指定（デフォルト: develop）
pr-tree --root main [branch]
```

### オプション一覧

| オプション | 説明 |
|-----------|------|
| `--open` | HTML を生成してブラウザで開く |
| `--root <branch>` | ルートブランチを指定。指定するとリポジトリルートの `.pr-tree` に保存され、以降のデフォルトになる |

## 表示例

```
develop
└── [open] #101 L1 PR のタイトル
      feature/l1-branch
    ├── [merged] #110 兄弟 PR A
    │     feature/sibling-a
    ├── [open] #123 カレントブランチの PR タイトル   ← カレント（ハイライト）
    │     feature/current-branch
    │   └── [open] #130 子 PR
    │         feature/child
    └── [merged] #115 兄弟 PR B
          feature/sibling-b
```

PR に紐づく issue が存在する場合は issue 番号・タイトルを先頭に表示します。

```
├── #42 issue タイトル
    [open] #101 PR タイトル
      feature/branch
```

PR の状態はバッジで色分けされます。

## HTML ビュー（`--open`）

`--open` を付けると HTML を生成してブラウザで開きます。CLI 出力に加えて以下の機能が利用できます。

| 機能 | 説明 |
|------|------|
| merged / closed フィルタ | ページ上部のチェックボックスで merged・closed 状態の PR を表示 / 非表示できます |
| コミットハッシュ表示 | 各ノードに最新コミットハッシュ（7桁）を表示します |
| クリップボードコピー | ノードにカーソルを合わせると表示される `⎘` ボタンで、ブランチ名またはコミットハッシュをコピーできます |

| バッジ | 状態 |
|--------|------|
| `[open]` | オープン中 |
| `[draft]` | ドラフト |
| `[merged]` | マージ済み |
| `[closed]` | クローズ済み |

## branch-tree との違い

| | branch-tree | pr-tree |
|---|---|---|
| ノード | ブランチ名 | PR（番号 / タイトル / 状態） |
| 親子関係の定義 | `gh pr list --head <branch>` の `baseRefName` | `PR_child.baseRefName == PR_parent.headRefName` |
| マージ後の表示 | ブランチ削除でツリー断絶 | PR は残るため断絶しない |
| データ取得 | ブランチごとに API 呼び出し | `gh pr list --state all` で一括取得 |
| L2 以降の表示 | カレントの子のみ | **全子孫を網羅表示** |

## リポジトリごとのルートブランチ設定

初回に `--root` を指定すると、リポジトリルートに `.pr-tree` が作成されます。

```bash
pr-tree --root main
```

以降そのリポジトリでは `main` がルートブランチとして使われます。`.pr-tree` をコミットすればチームで共有できます。

## 仕組み

- `gh pr list --state all` で全 PR を一括取得しメモリ上でツリーを構築
  - 一括取得に含まれなかったブランチは `gh pr list --head <branch>` で個別取得して補完
- 親子関係は `PR_child.baseRefName == PR_parent.headRefName` で定義
- L0（develop）と L1（develop 直下）はカレントブランチへの祖先チェーンを辿って特定
- L2 以降は L1 配下の全子孫を再帰的に展開
- 紐づく issue は GitHub GraphQL API（`closingIssuesReferences`）で取得