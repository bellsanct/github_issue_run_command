# GitHub Issue Run Command

このリポジトリは、GitHub Issue を起点にブランチ作成から PR 作成までを自動化するコマンド / プロンプトの置き場です。Codex 用と Claude 用のサンプルを同梱しています。

## 注意事項 / 前提
- GitHub CLI (`gh`) をインストールし、`gh auth status` でログイン済みであること（`gh` が実行できないとすべてのフローが失敗します）。
- 作業するリポジトリは Git 管理下で、未コミットの変更がない状態で実行してください。
- 危険なコマンドは含めていませんが、実行結果は必ず確認してください。

## ディレクトリ構成
- `commands/`: 共通のサンプルコマンド
- `.codex/commands`, `.codex/prompts`: Codex が読み込む実体
- `for_codex/prompts/`: Codex 用の配布ファイル。`.codex/prompts/` にコピーして使います。
- `for_claude/commands/`: Claude 用の配布ファイル。`.claude/commands/` にコピーして使います。
- `.claude/`: Claude の設定用ディレクトリ（`settings.local.json` など）。

## セットアップと実行手順

### Claude で使う場合
1. `.claude/commands` ディレクトリを作成（未作成の場合）。
2. `for_claude/commands/github_issue_run.md` を `.claude/commands/github_issue_run.md` に配置。
3. Claude のコマンドとして `/github_issue_run {issue_number}` を呼び出します（Issue 番号は引数で指定）。

### Codex で使う場合
1. `.codex/prompts` が空の場合は作成。
2. `for_codex/prompts/github_issue_run.md` を `.codex/prompts/github_issue_run.md` に配置。
3. `.codex/commands` に含まれるサンプルコマンド（`echo`, `hello` など）はそのまま利用できます。必要に応じて `commands/` にあるファイルをコピーして編集してください。
4. Codex では `/prompts:github_issue_run` を実行し、案内に従って Issue 番号を対話形式で入力すると自動化が始まります。

## 参考コマンド
- `gh auth status` で GitHub CLI のログイン状態を確認
- `gh issue view <number>` で Issue 詳細を取得
- `git status` で作業ツリーのクリーン状態を確認
