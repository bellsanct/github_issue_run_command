---
description: "GitHub Issue Runner for Codex (with argument parsing)"
---

# /github_issue_run コマンド（Codex 用）

このプロンプトは、`/prompts:github_issue_run 1` のような  
Codex の「引数が自動で渡らない環境」でも確実に issue 番号を取得し、  
GitHub Issue の自動処理（ブランチ作成〜実装〜PR）を行うための仕様書です。


ユーザー入力から `{issue_number}` を読み取り、これを `ISSUE_NUMBER` として扱います。

# 🔍 最初に行うこと（絶対必須）
**ユーザーが入力した全文を必ず解析して、ISSUE_NUMBER を抽出してください。**

解析対象例：
/prompts:github_issue_run 1
/github_issue_run 12
/prompts:github_issue_run 99

抽出ルール：

- 数字だけを抜き出す（最後の整数を ISSUE_NUMBER と認識）
- 数字が存在しない場合は、「Issue 番号が指定されていないため終了します」と返して終了する

# 📌 ISSUE_NUMBER の抽出例

| 入力 | 取得される ISSUE_NUMBER |
|------|----------------------------|
| `/prompts:github_issue_run 1` | `1` |
| `/github_issue_run 42` | `42` |
| `/prompts:github_issue_run    105` | `105` |

抽出後は **以降の全てのコマンド内で ISSUE_NUMBER を使う**。



---

# 🔒 前提条件

- GitHub CLI (`gh`) がインストール済みでログイン済みであること  
- 作業ディレクトリが GitHub リポジトリであること  
- Codex は `shell_command` により PowerShell コマンドを `!` 付きで実行可能  
- ファイル編集は **apply_patch ツール**を優先して使用すること  
- 危険コマンド（`rm -rf` など）禁止  
- 未コミットの変更がある場合は作業を中断すること  


---

# 🧭 処理の全体フロー

1. 前提チェック（Git 管理 / 作業ツリー / 認証）
2. Issue 情報を取得して要約
3. 作業ブランチ作成
4. 対応方針の設計（文章）
5. ファイル編集（apply_patch）
6. テスト / Lint / Build（存在する場合）
7. コミット作成
8. リモートへ push
9. PR 作成（Issue と自動リンク）
10. Issue へのコメント（任意）
11. 完了報告


---

# 🚀 手順詳細

## 0. 前提チェック

### 0-1. Git リポジトリである確認

git rev-parse --is-inside-work-tree

- 失敗したら「Git リポジトリではないため終了します」と説明して終了。

### 0-2. 作業ツリーに未コミットの変更がないか
git status --short

- 出力が空でなければ  
  「未コミットの変更があるため、自動処理は実行しません」と説明して終了。

### 0-3. リモートと gh 認証
git remote -v
gh auth status

- GitHub ではない or 認証エラーなら説明して終了。


---

## 1. Issue 情報を取得して要約

ユーザー入力から `ISSUE_NUMBER` を抽出。

Issue を取得：gh issue view ISSUE_NUMBER --json number,title,body,url,state,labels

次を日本語で要約：
- タイトル  
- 本文  
- ラベル  
- 状態  
- 対応に必要な作業（TODO リスト）

状態が `closed` の場合：
→ 「Issue が既に closed なので終了します」と説明して終了。


---

## 2. 作業ブランチの作成

### 2-1. 現在ブランチ確認
git branch --show-current

必要なら main 等へ移動：git checkout main

### 2-2. 最新化
git pull --ff-only

### 2-3. 作業ブランチ作成
#### ブランチ名の命名規則:
- 機能追加: feature/{issue_number}_{簡潔な説明}
- バグ修正: fix/{issue_number}_{簡潔な説明}
- リファクタリング: refactor/{issue_number}_{簡潔な説明}

git checkout -b <ブランチ名>

---

## 3. 対応方針の設計（文章）

まずリポジトリ構造の確認：ls

必要に応じて：ls src
必要に応じて：cat package.json

Issue の内容に基づいて：

- どのファイルを修正するか  
- どんなアプローチで実装するか  
- なぜその変更が必要か  

を文章で説明する。


---

## 4. 実装（apply_patch）

apply_patch を用いて次を行う：

1. 修正前後のコードを含む差分パッチを提示  
2. その修正が何を解決するのか説明  
3. 小さく安全なパッチを複数回に分けることを推奨  
4. 修正後、差分確認：git diff

※ 破壊的な変更を行わないよう注意する。

---

## 5. テスト / Lint / Build（存在する場合）

存在するコマンドのみ実行。

例（package.json がある場合）：
npm test
npm run lint
npm run build

失敗した場合：

- エラーログを要約  
- 必要であれば修正  
- 再実行  
- 大規模な問題であれば PR 本文に記載するためメモする  


---

## 6. コミットの作成

### 6-1. 変更確認
git status

### 6-2. ステージング
git add .
- または変更したファイルのみをaddする
git add path/to/file

### 6-3. コミット
メッセージ形式：fix: #ISSUE_NUMBER <短い説明>
コマンド例: git commit -m "fix: #42 prevent crash when user is null"

---

## 7. リモートへ push
git push -u origin <ブランチ名>


エラーが出た場合は理由を説明し、安全策を案内する。


---

## 8. PR を作成
コマンド例: 
gh pr create --title "Fix: #ISSUE_NUMBER <短い説明>" --body "<PR内容を説明する文章>"

可能ならラベル追加：gh issue edit ISSUE_NUMBER --add-label "review-requested"


存在しないラベルの場合は無理せずスキップ。

## 9. Issue へのコメント（任意）
gh issue comment ISSUE_NUMBER --body "<コメント内容>"

---

## 10. 完了報告（ユーザー向け）

最後に、以下をまとめて報告する：

- 対象 Issue 番号 & タイトル  
- 作成したブランチ名  
- 作成した PR の URL  
- 主な変更点  
- 実行したテスト  
- 懸念点・レビューしてほしい点  

