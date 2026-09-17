# Python個人開発向け サブエージェント構成

人間がOrchestratorに徹し、実作業を7体のサブエージェントに委譲するための定義一式です。

## 配置

```bash
# 全プロジェクトで使う場合
mkdir -p ~/.claude/agents
cp agents/*.md ~/.claude/agents/

# 特定プロジェクトだけで使う場合（gitで共有できる）
mkdir -p .claude/agents
cp agents/*.md .claude/agents/

# プロジェクト規約
cp CLAUDE.md.template ./CLAUDE.md   # 中身を自分の環境に合わせて編集
```

**agents ディレクトリを新規に作った場合は Claude Code を再起動してください。**
起動時に存在しなかったディレクトリは監視対象に入らないため、再起動しないと認識されません。
既存ディレクトリへのファイル追加・編集なら、数秒で自動的に反映されます。

## 7体の役割とモデル配分

| エージェント | 役割 | 書込権限 | モデル |
| --- | --- | --- | --- |
| `librarian` | ライブラリ選定・API調査・バージョン互換 | なし | sonnet |
| `implementer` | 実装（tests/ は触らない） | src のみ | sonnet |
| `test-author` | テスト作成（実装は触らない） | tests/ のみ | sonnet |
| `test-runner` | テスト実行と失敗の要約 | なし | haiku |
| `debugger` | 原因究明と最小修正 | あり | opus |
| `reviewer` | 読み取り専用レビュー | なし | opus |
| `documenter` | docstring / README / CHANGELOG | あり | sonnet |

モデル配分には意図があります。**判断の質が成果を左右する役割（debugger, reviewer）にopus、
手順が明確な役割（implementer, test-author）にsonnet、出力を要約するだけの役割（test-runner）にhaiku**
を割り当てています。メインセッションはOpusで走らせ、あなたはそこで指揮に専念してください。

すべてを一段階下げてコストを抑えたい場合は、各ファイルの `model:` 行を書き換えるか、
`~/.claude/settings.json` に以下を置きます。

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "sonnet",
    "CLAUDE_CODE_SUBAGENT_MODEL_FORCE": "1"
  }
}
```

（`FORCE` を立てると各定義の `model` は無視されます）

## 権限の分離が設計の中心

この構成の要は、役割分担そのものより **書き込み権限の非対称性** にあります。

- `implementer` は tests/ を触れない → テストを書き換えて通す事故が起きない
- `test-author` は実装を触れない → テストを実装に迎合させる事故が起きない
- `reviewer` と `test-runner` は `disallowedTools: Write, Edit` で読み取り専用 → 指摘と修正が混ざらない

単一のエージェントに全部やらせると、これらは静かに混線します。分けることで、
判断が必要な場面が必ずあなたのところに上がってきます。それがOrchestratorでいるということです。

## 1サイクルの回し方

```
1. あなた: 何を作るかを決める（ここは委譲しない）
   ↓ 必要なら
2. @librarian このユースケースに合うHTTPクライアントを比較して
   ↓ あなたが採用を決定
3. Plan mode（Shift+Tab 2回）で方針を固める
   ↓ あなたが計画を承認
4. @implementer 承認した方針で実装して
   ↓
5. @test-author 追加された機能のテストを書いて
   ↓
6. @test-runner テストを回して
   ↓ 失敗したら
7. @debugger この失敗を直して → 6に戻る
   ↓ 通ったら
8. @reviewer 今回の変更をレビューして
   ↓ あなたが指摘の採否を決定
9. @documenter docstringとREADMEを更新して
```

`@` の後にエージェント名を入力すると補完が出ます。`@` で明示的に指名するのが
Orchestratorの基本動作です。自然言語でエージェント名を言うだけだと委譲するか
どうかはClaudeの判断になり、指揮権が曖昧になります。

## 並列で回す

独立した調査は並列化が効きます。

```
認証・DB・API の3モジュールを、それぞれ別のサブエージェントで並列に調査して
```

ただしサブエージェントの結果はメインの会話に返るため、詳細な報告を返す
エージェントを大量に並列実行すると、節約したはずの文脈を結局消費します。
同時実行はデフォルトで20体まで、ネストは3層までです。

## 蓄積させる

`reviewer` と `librarian` には `memory: project` を設定してあります。
`.claude/agent-memory/<エージェント名>/` に知見が溜まり、会話をまたいで参照されます。
gitにコミットすれば、マシンを変えても引き継がれます。

意識的に使うなら、作業の区切りでこう言ってください。

```
@reviewer 今回のレビューで分かったこの プロジェクト固有のパターンをメモリに残して
```

## 調整すべき箇所

配布したまま使えますが、以下は自分の環境に合わせてください。

1. **`CLAUDE.md`** — ツールチェインのコマンド。uv / poetry / pip、ruff / black+flake8、
   mypy / pyright など、実際に使っているものに書き換える
2. **`implementer.md` の「実装後に必ずやること」** — 上と同じコマンドに揃える
3. **`documenter.md` のdocstringスタイル** — NumPyスタイルを使っているなら差し替える
4. **`test-runner.md`** — nox や tox を使っているなら、その実行コマンドを明記する

## 制約を機械的に強制したい場合

「tests/ を触らない」といった指示はプロンプトによる約束であり、100%の保証ではありません。
確実に止めたい場合は、フロントマターに `PreToolUse` フックを足して、
対象パスへの書き込みを終了コード2でブロックできます。

```yaml
hooks:
  PreToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/deny-tests-dir.sh"
```

まずはフックなしで運用し、実際に越境が起きた役割にだけ足すのが現実的です。

## 意図的に入れていないもの

- **planner** — 組み込みの Plan サブエージェントとPlan modeで足ります。方針決定は
  あなたが握るべき部分なので、専用エージェントを作ると委譲しすぎになります
- **explorer** — 組み込みの Explore がコードベース検索を担当します。`librarian` は
  外部ライブラリ調査に特化させ、役割の重複を避けています
