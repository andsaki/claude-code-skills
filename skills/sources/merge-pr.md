---
description: "PRをマージ（Issue自動クローズ対応）"
argument-hint: [PR番号]
allowed-tools: Bash(gh:*), Bash(git:*)
---

# merge-pr

指定したPRをマージします。PRに `Closes #123` が含まれていれば、GitHubの標準機能でIssueも自動クローズされます。

**使用方法**:
- `/merge-pr 100` - PR #100 をマージ
- `/merge-pr` - 現在のブランチのPRをマージ

## 実行手順

### 1. PR番号の取得

#### 引数がある場合
引数からPR番号を取得：
```bash
pr_number=100  # 例: /merge-pr 100
```

#### 引数がない場合
現在のブランチのPRを自動検出：
```bash
gh pr view --json number,state,title -q .number
```

検出できない場合はエラー表示して終了。

### 2. PR情報の確認

PRの詳細を取得：
```bash
gh pr view [PR_NUMBER] --json number,title,state,isDraft,mergeable,reviews,statusCheckRollup
```

表示する情報：
- タイトル
- 状態（OPEN/CLOSED/MERGED）
- Draft状態
- レビュー状態（Approved/Changes requested/Pending）
- CIステータス（Success/Failure/Pending）
- マージ可能か

### 3. マージ前チェック

以下を確認：

#### 3.1. PRが既にマージ済みか
```bash
state=$(gh pr view [PR_NUMBER] --json state -q .state)
if [ "$state" = "MERGED" ]; then
  echo "❌ このPRは既にマージされています"
  exit 1
fi
```

#### 3.2. PRがDraftか
```bash
is_draft=$(gh pr view [PR_NUMBER] --json isDraft -q .isDraft)
if [ "$is_draft" = "true" ]; then
  echo "⚠️  このPRはDraftです。マージしますか？ (y/N)"
fi
```

#### 3.3. レビュー状態の確認
```bash
reviews=$(gh pr view [PR_NUMBER] --json reviews -q '.reviews | map(select(.state == "APPROVED")) | length')
if [ "$reviews" -eq 0 ]; then
  echo "⚠️  Approveされていません。マージしますか？ (y/N)"
fi
```

#### 3.4. CIステータスの確認
```bash
checks=$(gh pr view [PR_NUMBER] --json statusCheckRollup -q '.statusCheckRollup[] | select(.conclusion == "FAILURE") | .name')
if [ -n "$checks" ]; then
  echo "⚠️  CI が失敗しています："
  echo "$checks"
  echo "マージしますか？ (y/N)"
fi
```

### 4. Issue自動クローズの確認

PR本文とコミットメッセージから関連Issueを検出：

```bash
# PR本文を取得
body=$(gh pr view [PR_NUMBER] --json body -q .body)

# Issueキーワードを検索
issues=$(echo "$body" | grep -iE "(close[sd]?|fix(e[sd])?|resolve[sd]?) #[0-9]+" | grep -oE "#[0-9]+" | sort -u)

if [ -n "$issues" ]; then
  echo ""
  echo "📋 以下のIssueがマージ時に自動クローズされます："
  echo "$issues"
  echo ""
fi
```

### 5. マージ方法の選択

リポジトリの設定とPRの状況に応じて適切な方法を提案：

```bash
# コミット数を確認
commit_count=$(gh pr view [PR_NUMBER] --json commits -q '.commits | length')

echo "マージ方法を選択してください："
echo "1. merge   - 全てのコミットを保持（コミット数: $commit_count）"
echo "2. squash  - 1つのコミットにまとめる（推奨）"
echo "3. rebase  - コミット履歴を直線化"
echo ""
```

**推奨ロジック**:
- コミット数が1つ → `merge` または `squash`（どちらでも同じ）
- コミット数が2-5個 → `squash`（推奨）
- コミット数が6個以上 → ユーザーに確認

デフォルトは `squash` を推奨。

### 6. マージの実行

選択された方法でマージを実行：

#### Squash Merge（推奨）
```bash
gh pr merge [PR_NUMBER] --squash --delete-branch
```

#### Merge Commit
```bash
gh pr merge [PR_NUMBER] --merge --delete-branch
```

#### Rebase Merge
```bash
gh pr merge [PR_NUMBER] --rebase --delete-branch
```

**重要オプション**:
- `--delete-branch`: マージ後にブランチを自動削除
- `--auto`: 全ての承認が得られたら自動マージ（オプション）

### 7. マージ確認

マージ後、以下を確認：

```bash
# PRの状態を確認
gh pr view [PR_NUMBER] --json state,mergedAt

# 関連Issueがクローズされたか確認
if [ -n "$issues" ]; then
  echo ""
  echo "📋 Issue クローズ状態を確認："
  for issue in $issues; do
    issue_num=$(echo "$issue" | sed 's/#//')
    state=$(gh issue view $issue_num --json state -q .state)
    echo "  Issue $issue: $state"
  done
fi
```

### 8. 完了メッセージ

```
✅ PR #100 をマージしました！

マージ方法: squash
ブランチ: feature/add-dark-mode → 削除済み

📋 自動クローズされたIssue:
  - Issue #123: CLOSED
  - Issue #45: CLOSED

🔗 https://github.com/owner/repo/pull/100
```

---

## マージ方法の比較

| 方法 | 説明 | 使用ケース |
|-----|------|----------|
| **merge** | 全コミットを保持 | 重要な履歴を残したい |
| **squash** | 1つにまとめる | 通常はこれ（推奨） |
| **rebase** | 直線的な履歴 | クリーンな履歴重視 |

## Issue自動クローズのキーワード

以下のキーワードがPR本文またはコミットメッセージに含まれていれば、マージ時に自動クローズ：

- `Closes #123`
- `Fixes #123`
- `Resolves #123`
- `Close #123`, `Fix #123`, `Resolve #123`
- 複数形も可: `Closed`, `Fixed`, `Resolved`

**例**:
```markdown
## 概要
ダークモードを実装しました。

Closes #123
Fixes #45
```
→ マージ時に Issue #123 と #45 が自動クローズ

## 注意事項

- マージ前に必ずCIが通っているか確認
- Draft PRは通常マージしない
- Approveがない場合は警告を表示
- ブランチは自動削除される（`--delete-branch`）
- Issue自動クローズはGitHubの標準機能（ワークフロー不要）
- `--auto`オプションを使う場合、承認要件を満たす必要あり

## トラブルシューティング

### マージできない場合

**コンフリクトがある**:
```bash
# ローカルで解決
git checkout feature-branch
git merge main
# コンフリクト解決
git push
```

**権限がない**:
- リポジトリの write 権限が必要
- Protected branch の設定を確認

### Issue が自動クローズされない

- PR本文に `Closes #123` があるか確認
- コミットメッセージに含まれているか確認
- 同じリポジトリのIssueである必要がある
- 別リポジトリの場合: `Closes owner/repo#123`

---

## 使用例

### 基本的な使い方

```
# PR #100 をマージ
/merge-pr 100

# 現在のブランチのPRをマージ
/merge-pr
```

### 実行の流れ

1. PR情報を確認
2. レビュー・CIステータスをチェック
3. 関連Issueを表示
4. マージ方法を選択（squash推奨）
5. マージ実行
6. Issueクローズを確認

---

**作成日**: 2026-03-15
**バージョン**: 1.0.0
