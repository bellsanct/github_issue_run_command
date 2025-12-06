---
name: github_issue_run
description: Run a GitHub Issue workflow (branch, optional PR/comment)
allowed_tools: Bash(git, gh), Read(*), Write, Fetch(*)
---

# GitHub Issue タスク実行コマンド（Codex）

GitHub Issue の番号を受け取り、以下のワークフローをガイドまたは自動実行します。

- リポジトリとデフォルトブランチの確認
- Issue 情報の取得（タイトル等）
- ブランチ作成（`feature/{issue}_{短縮タイトル}`）
- （任意）Draft PR 作成、Issue へのコメント、ラベル付与

Usage:

```
/github_issue_run {issue_number} [--apply]
```

Examples:

```
/github_issue_run 42
/github_issue_run 42 --apply
```

注意
- デフォルトはドライラン（実際の変更は行いません）。`--apply` を付けると実行します。
- Git と GitHub CLI の認証/権限が必要です。

```bash
set -euo pipefail

ISSUE_INPUT="$ARGUMENTS"

if [ -z "${ISSUE_INPUT:-}" ]; then
  echo "Usage: /github_issue_run {issue_number} [--apply]" >&2
  exit 1
fi

# 引数パース: 最初の数値を Issue 番号として取得、残りはフラグ
ISSUE_NUMBER="$(printf '%s' "$ISSUE_INPUT" | awk '{print $1}')"
FLAGS="$(printf '%s' "$ISSUE_INPUT" | cut -s -d' ' -f2-)"

case "$ISSUE_NUMBER" in
  ''|*[!0-9]*) echo "Error: issue_number must be numeric" >&2; exit 1;;
esac

APPLY=false
if printf '%s' "$FLAGS" | grep -q -- "--apply"; then
  APPLY=true
fi

echo "[info] Issue #: $ISSUE_NUMBER"

# 前提ツールの確認
if ! command -v git >/dev/null 2>&1; then
  echo "Error: git not found" >&2; exit 1
fi
if ! command -v gh >/dev/null 2>&1; then
  echo "Error: GitHub CLI (gh) not found" >&2; exit 1
fi

echo "[check] gh auth status"
if ! gh auth status >/dev/null 2>&1; then
  echo "Error: gh is not authenticated. Run 'gh auth login'." >&2
  exit 1
fi

# リポジトリの特定 (owner/repo)
REPO_URL="$(git remote get-url origin 2>/dev/null || true)"
if [ -z "$REPO_URL" ]; then
  echo "Error: not a Git repo or no 'origin' remote." >&2
  exit 1
fi
REPO="$(printf '%s' "$REPO_URL" | sed -E 's#.*github\.com[:/]([^/]+/[^.]+)(\.git)?$#\1#')"

if ! printf '%s' "$REPO" | grep -q '/'; then
  echo "Error: failed to parse origin remote as owner/repo: $REPO_URL" >&2
  exit 1
fi

echo "[info] Target repo: $REPO"

# デフォルトブランチの特定
DEFAULT_BRANCH="$(git remote show origin 2>/dev/null | awk '/HEAD branch/ {print $NF}')"
if [ -z "$DEFAULT_BRANCH" ]; then
  # fallback
  DEFAULT_BRANCH="main"
fi
echo "[info] Default branch: $DEFAULT_BRANCH"

echo "[step] Fetch issue #$ISSUE_NUMBER"
ISSUE_TITLE="$(gh issue view "$ISSUE_NUMBER" --repo "$REPO" --json title --jq .title)"
ISSUE_URL="https://github.com/$REPO/issues/$ISSUE_NUMBER"
if [ -z "$ISSUE_TITLE" ]; then
  echo "Error: failed to fetch issue title." >&2
  exit 1
fi
echo "[info] Issue title: $ISSUE_TITLE"

# ブランチ名の生成
slugify() {
  printf '%s' "$1" \
    | tr '[:upper:]' '[:lower:]' \
    | sed -E 's/[^a-z0-9]+/-/g; s/^-+|-+$//g; s/-{2,}/-/g' \
    | cut -c1-32
}
SHORT_TITLE="$(slugify "$ISSUE_TITLE")"
BRANCH="feature/${ISSUE_NUMBER}_${SHORT_TITLE}"
echo "[plan] Branch: $BRANCH"

echo "[plan] Actions:"
echo "  - Create branch from origin/$DEFAULT_BRANCH"
echo "  - (optional) Create Draft PR"
echo "  - (optional) Comment issue with PR link and branch"
echo "  - (optional) Add 'review-requested' label if exists"

if [ "$APPLY" != true ]; then
  echo "[dry-run] No changes applied. Re-run with --apply to execute."
  exit 0
fi

echo "[exec] Fetch and create branch"
git fetch origin "$DEFAULT_BRANCH" --quiet || true
git checkout -b "$BRANCH" "origin/$DEFAULT_BRANCH"

# ここで実装を行うフェーズ（ユーザー操作想定）。最低限、ブランチを push できるよう
# 変更が無い場合に備えてノートファイルを用意（初回のみ）
NOTE_PATH="for_codex/github_issue_run.md"
if [ ! -f "$NOTE_PATH" ]; then
  mkdir -p "for_codex"
  printf '%s\n' "# Task notes for issue #$ISSUE_NUMBER" > "$NOTE_PATH"
fi

git add -A
if ! git diff --cached --quiet; then
  git commit -m "chore: initialize task (#$ISSUE_NUMBER)"
fi

echo "[exec] Push branch"
git push -u origin "$BRANCH"

echo "[exec] Create Draft PR"
PR_TITLE="WIP: $ISSUE_TITLE (#$ISSUE_NUMBER)"
PR_BODY=$(cat <<'EOT'
## 概要
このPRはAIによって支援され作成されました。人間によるレビューをお願いします。

## 関連Issue
Closes #{issue_number}

## 変更点
- 初期セットアップ

## 確認事項
- [ ] コードレビュー
- [ ] テスト実行
- [ ] ドキュメント更新

---
🛠 このPRはCodexによって作業ガイドのもとで作成されました。
EOT
)

# {issue_number} を置換
PR_BODY="${PR_BODY//\#{issue_number}/#$ISSUE_NUMBER}"

PR_URL="$(
  gh pr create \
    --repo "$REPO" \
    --title "$PR_TITLE" \
    --body "$PR_BODY" \
    --draft \
    --head "$BRANCH" \
    --base "$DEFAULT_BRANCH" \
    --fill 2>/dev/null | tail -n1
)"

if [ -z "$PR_URL" ]; then
  # fallback: 取得
  PR_URL="$(gh pr view --repo "$REPO" --json url --jq .url 2>/dev/null || true)"
fi

echo "[info] PR: ${PR_URL:-<unknown>}"

echo "[exec] Comment issue with PR link"
COMMENT_BODY=$(cat <<EOT
🛠 自動実行（Codex）

レビューをお願いします。
PR: ${PR_URL:-N/A}
Branch: ${BRANCH}

---
このコメントはCodexによって自動投稿されました。
EOT
)

gh issue comment "$ISSUE_NUMBER" --repo "$REPO" --body "$COMMENT_BODY"

echo "[exec] Attempt label add: review-requested"
gh issue edit "$ISSUE_NUMBER" --repo "$REPO" --add-label "review-requested" 2>/dev/null || true

echo "[done] Completed. Summary:"
echo "  Repo   : $REPO"
echo "  Issue  : #$ISSUE_NUMBER ($ISSUE_TITLE)"
echo "  Branch : $BRANCH"
echo "  PR     : ${PR_URL:-N/A}"
echo "  Issue  : $ISSUE_URL"
```

