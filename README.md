# claude-agents-python

Python個人開発向けの Claude Code サブエージェント定義集です。実装・テスト・レビュー・ドキュメントといった役割を7体のサブエージェントに分担させ、人間はOrchestrator（指揮役）として判断だけに専念できるようにします。

役割ごとの設計意図や運用サイクルは [SETUP.md](./SETUP.md) を参照してください。このREADMEは導入手順に絞っています。

---

## 前提条件

| 項目 | 内容 |
| --- | --- |
| Claude Code | インストール済みであること（デスクトップアプリのCodeタブ、またはCLI） |
| 認証 | Claude サブスクリプション、または `ANTHROPIC_API_KEY` |
| 対象プロジェクト | Python。`git` 管理下だとレビュー系が本領を発揮します |

APIキーで使う場合は、シェルの設定ファイルに以下を追加します。

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

サブエージェントを並列で回すとトークン消費がまとまって増えます。常用するならサブスクリプション認証のほうがコストは読みやすくなります。

---

## インストール

### 手順1. 定義ファイルを配置する

配置先は2択です。両方に同名のファイルがある場合はプロジェクト側が優先されます。

**A. 全プロジェクトで使う（個人用）**

```bash
mkdir -p ~/.claude/agents
cp agents/*.md ~/.claude/agents/
```

**B. 特定プロジェクトだけで使う（gitで共有可能）**

```bash
cd /path/to/your-project
mkdir -p .claude/agents
cp /path/to/claude-agents-python/agents/*.md .claude/agents/
```

まずはAで入れて、プロジェクト固有の調整が必要になったらBに複製する流れが扱いやすいです。

### 手順2. CLAUDE.md を置く

プロジェクトルートに配置します。**このファイルは（組み込みの Explore と Plan を除く）全サブエージェントに読み込まれるため、共通規約はここに一元化します。**

```bash
cd /path/to/your-project
cp /path/to/claude-agents-python/CLAUDE.md.template ./CLAUDE.md
```

コピーしたら中身を自分の環境に合わせて編集してください（後述の「環境に合わせる」を参照）。

### 手順3. Claude Code を再起動する

**`agents` ディレクトリを新規に作った場合、再起動が必須です。** 起動時に存在しなかったディレクトリは監視対象に入らず、そのセッションでは認識されません。

既存の `agents` ディレクトリへのファイル追加・編集であれば、数秒で自動的に反映されます。

### 手順4. 動作確認

プロジェクトディレクトリで Claude Code を起動し、入力欄に `@` を打ちます。補完候補に `agent-librarian`、`agent-implementer` などが並べば認識されています。

実際に1体呼んで確かめます。

```
@agent-test-runner テストを回して結果を教えて
```

---

## 環境に合わせる

配置したままでも動きますが、以下2箇所は実環境に合わせてください。ここがズレていると、エージェントが存在しないコマンドを叩こうとします。

### 1. `CLAUDE.md` のツールチェイン

テンプレートは **uv + ruff + mypy + pytest** を前提にしています。異なる場合は書き換えてください。

| 前提 | 使っていなければ差し替える |
| --- | --- |
| `uv run <cmd>` | `poetry run` / `python -m` / 素の実行 |
| `ruff check` / `ruff format` | `black` + `flake8` + `isort` |
| `mypy .` | `pyright` |
| `pytest -q` | `nox` / `tox` 経由のコマンド |

Pythonの下限バージョン、`src/` レイアウトかフラットかも実態に合わせます。

### 2. `agents/implementer.md` の「実装後に必ずやること」

CLAUDE.md と同じコマンドに揃えます。ここだけ古いままだと、実装後の検証が空振りします。

### 任意の調整

- **`agents/documenter.md`** — docstringをNumPyスタイルで統一しているなら、テンプレート部分を差し替える
- **`agents/test-runner.md`** — nox / tox 経由でテストしているなら、その実行コマンドを明記する
- **各ファイルの `model:`** — コストを抑えたい場合は下げる（次項）

---

## モデルとコストの設定

各定義には役割に応じたモデルを指定してあります。

| エージェント | モデル | 理由 |
| --- | --- | --- |
| `debugger` / `reviewer` | opus | 判断の質が成果を左右する |
| `implementer` / `test-author` / `librarian` / `documenter` | sonnet | 手順が明確 |
| `test-runner` | haiku | 出力を要約するだけ |

メインセッションはOpusで走らせ、そこで指揮に専念する想定です。

一括で下げたい場合は `~/.claude/settings.json` に置きます。

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "sonnet",
    "CLAUDE_CODE_SUBAGENT_MODEL_FORCE": "1"
  }
}
```

`FORCE` を立てると各定義の `model:` 指定は無視され、すべて上書きされます。定義側の指定を活かしたい場合は `FORCE` を外してください。

---

## 基本の使い方

`@` で明示的に指名するのがOrchestratorの基本動作です。エージェント名を自然言語で言うだけだと、委譲するかどうかはClaudeの判断に委ねられ、指揮権が曖昧になります。

```
@agent-implementer 承認した方針で実装して
@agent-reviewer 今回の変更をレビューして
```

独立した作業は並列化できます。

```
認証・DB・API の3モジュールを、それぞれ別のサブエージェントで並列に調査して
```

一連の流れは [SETUP.md](./SETUP.md) の「1サイクルの回し方」にまとめてあります。

---

## トラブルシューティング

**エージェントが `@` 補完に出てこない**
`agents` ディレクトリを新規作成した直後です。Claude Code を再起動してください。ファイルの配置先（`~/.claude/agents/` または `<project>/.claude/agents/`）と、フロントマターの `---` が正しく閉じているかも確認します。

**意図したエージェントに委譲されない**
`description` が委譲判断の材料です。「いつ使うか」を具体的に書き、積極的に使ってほしいものには「〜のときに積極的に使う」といった表現を入れます。確実に指名したい場合は `@` を使ってください。

**起動時に description が長すぎると警告が出る**
全エージェントの `description` 合計が15,000トークンを超えています。詳細は本文側に移し、`description` は「いつ使うか」の1〜2文に絞ってください。

**サブエージェントが禁止したはずのファイルを触った**
`tests/ を触らない` といった指示はプロンプトによる約束であり、100%の保証ではありません。確実に止めたい場合はフロントマターに `PreToolUse` フックを追加し、対象パスへの書き込みを終了コード2でブロックします（例は SETUP.md 参照）。まずはフックなしで運用し、実際に越境が起きた役割にだけ足すのが現実的です。

**トークン消費が想定より多い**
サブエージェントの報告はメインの会話に返るため、詳細な報告を返すエージェントを大量に並列実行すると、節約したはずの文脈を結局消費します。並列は独立した調査に限り、同時実行は数体に抑えてください（上限はデフォルトで同時20体、ネスト3層）。

**サブエージェントが前提を知らない**
非forkのサブエージェントは会話履歴を引き継がず、まっさらな文脈から始まります。必要な前提は依頼文に明示するか、恒久的なものは `CLAUDE.md` に書いてください。

---

## 知見を蓄積させる

`reviewer` と `librarian` には `memory: project` を設定してあります。`.claude/agent-memory/<エージェント名>/` に知見が溜まり、会話をまたいで参照されます。

gitにコミットすれば、マシンを変えても引き継がれます。

```bash
git add .claude/agent-memory
```

作業の区切りで明示的に促すと定着が早くなります。

```
@agent-reviewer 今回分かったプロジェクト固有のパターンをメモリに残して
```

---

## アンインストール

```bash
# 個人用
rm ~/.claude/agents/{librarian,implementer,test-author,test-runner,debugger,reviewer,documenter}.md

# プロジェクト用
rm -rf .claude/agents
```

`CLAUDE.md` と `.claude/agent-memory/` は必要に応じて別途削除してください。

---

## ファイル構成

```
claude-agents-python/
├── README.md              このファイル（導入手順）
├── SETUP.md               設計意図と運用サイクル
├── CLAUDE.md.template     プロジェクト規約のひな形
└── agents/
    ├── librarian.md       ライブラリ選定・API調査（読み取り専用）
    ├── implementer.md     実装（tests/ は触らない）
    ├── test-author.md     テスト作成（実装は触らない）
    ├── test-runner.md     テスト実行と要約（読み取り専用）
    ├── debugger.md        原因究明と最小修正
    ├── reviewer.md        コードレビュー（読み取り専用）
    └── documenter.md      docstring / README / CHANGELOG
```
