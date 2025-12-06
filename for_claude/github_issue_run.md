---
allowed_tools: Bash(git, gh), Read(*.md, *.js, *.ts, *.py, *.go, *.rb, *.java, *.php, *.css, *.html, *.json, *.yaml, *.yml), Write, Fetch(*)
description: "GitHub Issueを参照して開発を進める"
---

# GitHub Issue タスク実行コマンド

このコマンドは、GitHub Issueの番号を受け取り、Issueの内容に基づいて自動的に実装を行い、プルリクエストを作成し、Issueを更新します。

## 使い方

```
/github_issue_run {issue_number}
```

例:
```
/github_issue_run 42
```

## ワークフロー

### 1. 環境とリポジトリ情報の確認

まず、現在のGitリポジトリの情報を取得します:

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

### 2. Issue内容の取得と確認

GitHub CLIを使用してIssue情報を取得し、内容を確認します:

```bash
gh issue view {$ARGUMENTS} --repo $REPO
```

Issueのタイトル、本文、ラベル、アサイニーなどの情報を確認し、実装すべき内容を理解します。

### 3. プロジェクト構造の分析と基本設計

- プロジェクトのディレクトリ構造を確認
- 関連するファイルを特定
- 実装方針を決定
- 必要に応じて既存コードを参照

### 4. 実装計画の作成

TodoWriteツールを使用して、実装タスクの計画を作成します。

### 5. ブランチの作成

Issue番号を含むブランチ名を作成し、mainブランチ（またはデフォルトブランチ）から新しいブランチを切ります:

```bash
# ブランチ名の命名規則:
# - 機能追加: feature/{issue_number}_{簡潔な説明}
# - バグ修正: fix/{issue_number}_{簡潔な説明}
# - リファクタリング: refactor/{issue_number}_{簡潔な説明}

# 例:
git checkout -b feature/42_add_user_api
```

### 6. 実装

Issue内容に基づいて、必要なコードを実装します。

### 7. セルフレビュー

実装が完了したら、`/review`コマンドを使用してセルフレビューを実行します。

### 8. コミットの作成

レビューで問題がなければ、適切なコミットメッセージでコミットを作成します:

```bash
git add .
git commit -m "feat: {Issueの内容を要約} (#${issue_number})"
```

コミットメッセージの形式:
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

GitHub CLIを使用してDraft PRを作成します:

```bash
gh pr create \
  --repo $REPO \
  --draft \
  --title "{適切なタイトル} (#{issue_number})" \
  --body "## 概要
このPRはAIによって自動生成されました。必ず人間がコードレビューを行ってください。

## 関連Issue
Closes #{issue_number}

## 変更内容
- {変更内容のリスト}

## 確認事項
- [ ] コードレビュー完了
- [ ] テスト実行確認
- [ ] ドキュメント更新確認

---
🤖 このPRはClaude Codeによって自動生成されました。"
```

**重要**: PR本文の冒頭に必ずAIによる生成であることを明記してください。

### 11. GitHub Issueの更新

#### 11-1. コメントの追加

PRのリンクを含むコメントをIssueに追加します:

```bash
gh issue comment {issue_number} \
  --repo $REPO \
  --body "🤖 **AI自動実装完了**

レビュー依頼をお願いします。

**プルリクエスト**: {PRのURL}
**ブランチ**: {ブランチ名}

---
このコメントはClaude Codeによって自動投稿されました。
必ず人間によるコードレビューを実施してください。"
```

#### 11-2. ラベルの変更

Issueに「review-requested」ラベルを追加します（ラベルが存在する場合）:

```bash
gh issue edit {issue_number} \
  --repo $REPO \
  --add-label "review-requested"
```

### 12. 完了報告

最後に、以下の情報を含む完了メッセージを出力します:

```
✅ 実装完了しました。

**プルリクエスト**: {PRのURL}
**ブランチ**: {ブランチ名}
**Issue**: #{issue_number}

GitHub Issueへレビュー依頼のコメントを投稿しました。
```

## 注意事項

- **権限**: このコマンドはGitおよびGitHubへの書き込み権限を使用します。意図しない変更を防ぐため、実行前に必ずIssue内容を確認してください。
- **Issue内容の明確さ**: Issueの内容が不明確な場合や情報が不足している場合は、実装精度が低下する可能性があります。
- **タスクサイズ**: 小規模なタスクから始めることを推奨します。大規模なタスクは複数のIssueに分割することを検討してください。
- **人間によるレビュー**: 生成されたコードは必ず人間が確認してください。自動生成されたコードをそのままマージしないでください。
- **エラーハンドリング**: コマンド実行中にエラーが発生した場合は、エラー内容を確認し、必要に応じて手動で修正してください。

## トラブルシューティング

### ブランチ作成に失敗する場合
- リポジトリがクリーンな状態か確認（未コミットの変更がないか）
- デフォルトブランチが最新か確認

### PR作成に失敗する場合
- GitHub CLIが正しく認証されているか確認: `gh auth status`
- リポジトリへの書き込み権限があるか確認

### ラベル追加に失敗する場合
- 「review-requested」ラベルがリポジトリに存在するか確認
- 必要に応じて、リポジトリの設定でラベルを作成
