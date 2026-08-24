# Sasaki

AI 主体で開発を進めるためのオーケストレーション workspace。Claude Code を Leader、Claude の subagent を団員として動かす自律開発チームの「事務局」を担う。実装は配下の各プロジェクト (独立 git repo) が持ち、ここはチームの規約・知見・記録だけを管理する。

> 土台は友人 Touri の [ait913/Muraki](https://github.com/ait913/Muraki)。組織の型と記録の思想を借り、Amano 用に再構成した。団員は外部 CLI (Codex/Gemini) でなく Claude の Agent (subagent) ツールで回す点が違う。

## 思想

- **Leader が設計を書かない**: 意図理解 → 召集 → 統合判断に専念。設計は Architect、実装は Developer、テストは Reviewer に分離
- **設計 → 実装 → テストを役割で分離**: Architect が「実装で迷う余地のない」設計doc を書き、Developer は doc から実装し、Reviewer は **コードを見ず** doc だけからテストを生成する
- **ナレッジを残す**: 失敗・パターン・ツールの癖を `knowledge/` に蓄積。次セッション以降が grep で引ける
- **記録は内容の種類で振り分ける**: session (生ログ) から、事実は knowledge・繰り返す運用手順は skill・個人事情は memory へ。被らせない（「昇格」でなく種類で選ぶ）
- **ライブで書き換える**: 規約・仕様が変わったら**追記でなく置換**。矛盾の蓄積を防ぐ

## ロール

| ロール | 担当 | 役目 |
|---|---|---|
| **Leader** | Claude Code | 意図理解 → 団員召集 → 統合判断 |
| **Researcher** | `researcher` subagent | 設計**前**のリサーチ |
| **Architect** | `architect` subagent | 設計doc 執筆 (UI/UX 含む) |
| **Developer** | `developer` subagent | 設計doc から実装 |
| **Reviewer** | `reviewer` subagent | 設計doc から**テスト生成** → 走らせる |

団員の定義は [`.claude/agents/`](./.claude/agents/)。詳細な組織規約: [`CLAUDE.md`](./CLAUDE.md)

## ディレクトリ構成

```
Sasaki/
├── CLAUDE.md                       # 組織規約 (Claude Code が auto-load)
├── README.md                       # この本
├── .claude/agents/<role>.md        # subagent 定義 4 体
├── global/CLAUDE.md                # グローバル規約の原本 (~/.claude/CLAUDE.md に配置する)
├── global/portable-prompt.md       # 機械を持ち込めない環境用の起動プロンプト
├── knowledge/                      # クロスプロジェクト知見
│   ├── INDEX.md                    # 自動生成
│   ├── library/                    # ライブラリ・API知見
│   ├── pattern/                    # 設計パターン
│   ├── gotcha/                     # ハマりどころ
│   ├── tool-quirk/                 # ツール・モデルの癖
│   └── role/                       # ロール別 職業的習慣ノート
├── projects/                       # AI 管理プロジェクト本体 (各々独立 git repo)
│   └── <slug>/                     # 各 PJ (別 GitHub repo)
├── sessions/                       # セッションごとの作業記録
│   └── <yyyy-mm-dd>-<short-id>.md
├── scripts/                        # メタ運用スクリプト
│   └── gen-knowledge-index.py
└── worktrees/                      # 並列作業用 git worktree (gitignore)
```

## 新しい開発環境で使う

clone しただけでは動かない。Sasaki は `~/.claude/` 配下から Claude Code に読ませる作りなので、**repo 外の 3 点を、そのマシンの clone 先パスに合わせて繋ぎ直す**必要がある。

Claude Code に「Sasaki を clone した、使えるようにして」と言えば、この節を読んで実行できる。

1. **グローバル規約**: `global/CLAUDE.md` を `~/.claude/CLAUDE.md` に置く。これが「どのディレクトリでも Sasaki の Leader として動く」常駐設定の本体
2. **団員定義**: `.claude/agents/*.md` を `~/.claude/agents/` に symlink する (clone 先の絶対パスで)
3. **MCP 設定**: `.mcp.json` の `command` を、clone 先の `scripts/chrome-devtools-mcp.sh` の絶対パスに書き換える

### 原本マシンと副環境で置き方が違う

`global/CLAUDE.md` と `.mcp.json` は**絶対パスを埋め込む**方針 (団員をグローバル召集するため)。そのため配置方法を分ける:

| | 置き方 | 理由 |
|---|---|---|
| **原本マシン** (clone 先 = `/Users/amano/dev/Sasaki`) | symlink | repo が唯一の正。編集が即反映され drift しない |
| **副環境** (インターン先 PC 等) | copy して絶対パスを書き換え | パスが違うので symlink だと成立しない |

副環境で規約を編集したら、repo にコミットして原本マシンへ持ち帰る。

### 別途必要なもの (repo では賄えない)

- **SSH 鍵**: 各マシンで生成し GitHub に登録する
- **`projects/<slug>/`**: .gitignore 除外 (各 PJ は独立 repo)。必要な PJ は個別に clone する
- **skills**: `~/.claude/skills/` は Sasaki の管理外

## 記録の場所と責務

書く前に**どこに置くか**判断する。5 層あり責務は被らせない。詳細は [`CLAUDE.md`](./CLAUDE.md) の「記録の場所と責務」。

| 層 | パス | 読まれ方 |
|---|---|---|
| **memory** | `~/.claude/projects/.../memory/` | 全セッション auto-load |
| **session report** | `Sasaki/sessions/` | 必要時 grep |
| **knowledge** | `Sasaki/knowledge/`, `<project>/.knowledge/` | grep on-demand |
| **設計doc** | `<project>/.designs/<YYYYMMDD>-<feature>.md` | 該当作業時のみ |
| **skill** | `~/.claude/skills/` | 名前で自動起動 (繰り返す運用手順のカタログ) |

## 並列実行 (GitHub Flow + worktree)

複数機能 / 複数 Claude セッションが並列で動く前提。1 機能 = 1 ブランチ = 1 worktree。

```sh
git -C Sasaki/projects/<project> worktree add ../../worktrees/<project>-<slug> -b feature/<slug>
# ... 作業 ...
git -C Sasaki/projects/<project> worktree remove ../../worktrees/<project>-<slug>
git -C Sasaki/projects/<project> branch -d feature/<slug>
```

## ナレッジに目を通す

新規調査・新規設計の前に既存知見を引く:

```sh
grep -ril "<keyword>" Sasaki/knowledge/ Sasaki/projects/<slug>/.knowledge/ 2>/dev/null
cat Sasaki/knowledge/INDEX.md
```

INDEX 再生成: `python3 Sasaki/scripts/gen-knowledge-index.py`
