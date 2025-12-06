---
allowed_tools: Bash(git, gh), Read(*.md, *.js, *.ts, *.py, *.go, *.rb, *.java, *.php, *.css, *.html, *.json, *.yaml, *.yml), Write, Fetch(*)
description: "GitHub Issueを参照して開発を進める（Codex用）"
---

# GitHub Issue タスク実行コマンド（Codex）

このコマンドは、GitHub Issue の番号を受け取り、Issue の内容に基づいて実装を進め、プルリクエストを作成し、Issue を更新します。Claude 用コマンド（for_claude/github_issue_run.md）と可能な限り同じ流れ・挙動になるように構成しています。

## 使い方

```
/github_issue_run {issue_number}
```

例:
```
/github_issue_run 42
```

## ワークフロー

### 1. 環境・リポジトリ情報の確認

まず、現在の Git リポジトリ情報を取得します。

```bash
# リモートリポジトリのURLを取得
REPO_URL=$(git remote get-url origin)
echo "リモートURL: $REPO_URL"

# owner/repo 形式を抽出
REPO=$(echo $REPO_URL | sed -E 's/.*github\.com[:/]([^/]+\/[^.]+)(\.git)?$/\1/')
echo "対象リポジトリ: $REPO"

# カレントブランチを確認
CURRENT_BRANCH=$(git branch --show-current)
echo "現在のブランチ: $CURRENT_BRANCH"
```

Windows PowerShell を使用する場合は同等のコマンドに読み替えてください。

### 2. Issue情報の取得と確認

GitHub CLI を使用して Issue 情報を取得し、内容を確認します。

```bash
gh issue view {$ARGUMENTS} --repo $REPO
```

Issue のタイトル、本文、ラベル、アサイニーなどを確認し、実装すべき内容を明確化します。

### 3. プロジェクト構造の把握と基本設計

- プロジェクトのディレクトリ構造を確認
- 関連するファイルを特定
- 実装方針を決定
- 必要に応じて既存コードを参照

### 4. 実装計画の作成

Codex のプラン機能（TODO/plan）を用いて、実装タスクの計画を作成します。

### 5. ブランチの作成

Issue 番号を含むブランチ名を作成し、main（またはデフォルトブランチ）から切ります。

```bash
# ブランチ名の命名規則の例:
# - 機能追加: feature/{issue_number}_{簡潔な説明}
# - バグ修正: fix/{issue_number}_{簡潔な説明}
# - リファクタ: refactor/{issue_number}_{簡潔な説明}

# 例:
git checkout -b feature/42_add_user_api
```

### 6. 実装

Issue の要件に基づいて、必要なコードを実装します。

### 7. セルフレビュー

実装が完了したら、セルフレビューを実施します（レビュー項目の確認、差分の見直し、lint/test など）。

### 8. コミットの作成

レビューで問題がなければ、適切なコミットメッセージでコミットを作成します。

```bash
git add .
git commit -m "feat: {Issueの要点を要約} (#{issue_number})"
```

コミットメッセージ例:
- feat: 新機能
- fix: バグ修正
- refactor: リファクタリング
- docs: ドキュメント更新
- test: テスト追加・修正

### 9. ブランチのプッシュ

```bash
git push origin {ブランチ名}
```

### 10. プルリクエストの作成

GitHub CLI を使用して Draft PR を作成します。

```bash
gh pr create \
  --repo $REPO \
  --draft \
  --title "{適切なタイトル} (#{issue_number})" \
  --body "## 概要\nこのPRはAIによって自動生成されました。必ず人間がコードレビューを行ってください。\n\n## 関連Issue\nCloses #{issue_number}\n\n## 変更点\n- {変更点のリスト}\n\n## 確認事項\n- [ ] コードレビュー完了\n- [ ] テスト実行確認\n- [ ] ドキュメント更新\n\n---\n🛠 このPRはCodexによって作成されました"
```

重要: PR本文の冒頭に AI による生成である旨を明記してください。

PR の URL は `gh pr view --json url --jq .url` で取得できます。

### 11. GitHub Issue の更新

#### 11-1. コメントの追加

PR のリンクを含むコメントを Issue に追加します。

```bash
PR_URL=$(gh pr view --json url --jq .url)

gh issue comment {issue_number} \
  --repo $REPO \
  --body "🛠 **AI自動実装完了**\n\nレビューをお願いします。\n\n**プルリクエスト**: ${PR_URL}\n**ブランチ**: {ブランチ名}\n\n---\nこのコメントは Codex によって自動投稿されました。必ず人間によるコードレビューを実施してください。"
```

#### 11-2. ラベルの変更

Issue に「review-requested」ラベルを追加します（ラベルが存在する場合）。

```bash
gh issue edit {issue_number} \
  --repo $REPO \
  --add-label "review-requested"
```

### 12. 完了報告

最後に、以下の情報を含む完了メッセージを出力します。

```
✅ 実装が完了しました。

**プルリクエスト**: {PRのURL}
**ブランチ**: {ブランチ名}
**Issue**: #{issue_number}

GitHub Issue へレビュー依頼のコメントを投稿しました。
```

## 注意事項

- 権限: このコマンドは Git および GitHub への書き込み権限を使用します。意図しない変更を防ぐため、実行前に Issue 内容を確認してください。
- Issue 内容の明確さ: Issue の記述が不明確な場合は実装精度が低下する可能性があります。必要に応じて補足情報を収集してください。
- タスクサイズ: 可能であれば小さく分割して進めることを推奨します。
- 人間によるレビュー: 生成コードは必ず人間が確認してください。自動生成されたコードをそのままマージしないでください。
- エラーハンドリング: コマンド実行中にエラーが発生した場合は、エラーログを確認し、必要に応じて手動で修正してください。

## トラブルシューティング

### ブランチ作成に失敗する場合
- リポジトリがクリーンな状態か確認（未コミットの変更がないか）
- デフォルトブランチが最新か確認

### PR作成に失敗する場合
- GitHub CLI が正しく認証されているか確認: `gh auth status`
- リポジトリへの書き込み権限を確認

### ラベル追加に失敗する場合
- 「review-requested」ラベルがリポジトリに存在するか確認
- 必要に応じてラベルを作成

