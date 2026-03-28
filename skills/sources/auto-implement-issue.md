---
description: "GitHub issueを完全自動で実装（使用例: /auto-implement-issue 123）"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(npm:*), Bash(npx:*), Read, Write, Edit, Glob, Grep
---

# /auto-implement-issue

GitHub issueを読み取り、**完全自動で実装からPR作成、Issueコメントまで全て実行**します。

**使用方法**: `/auto-implement-issue <issue番号>`

例: `/auto-implement-issue 123`

⚠️ **注意**: このコマンドはユーザー確認を最小限にして全自動で実行します。重要な変更の場合は `/implement-issue` を使用してください。

## 実行フロー（全自動）

```
1. Issue取得 → 2. ブランチ作成 → 3. 実装 → 4. テスト → 5. 自動修正（必要時）
→ 6. コミット → 7. プッシュ → 8. PR作成 → 9. Issueコメント
```

## 実行手順

### ステップ0: ログディレクトリの準備（start-impl 互換）

`auto-implement-issue` でも start-impl と同じように進行ログ（plan/prompt）を残します。  
リポジトリ直下に `tmp/` ディレクトリが **すでに存在する場合** は、衝突を避けるため `temp/` に作成してください。

```bash
if [ -d tmp ]; then
  target_dir="temp/$(printf '%03d' NEXT_ID)_${ISSUE_SLUG}"
else
  target_dir="tmp/$(printf '%03d' NEXT_ID)_${ISSUE_SLUG}"
fi

!mkdir -p "$target_dir"
!cat <<'EOF' > "$target_dir/plan.md"
# 実装計画: ...
EOF
!cat <<'EOF' > "$target_dir/prompt.md"
# 実装セッション: ...
EOF
```

- `NEXT_ID` は既存ディレクトリを確認して採番。
- `ISSUE_SLUG` は `issue-123-xxxxx` のように一意化。
- 以降のステップではこの `target_dir` にメモを残す。

### ステップ1: Issue情報の取得

Issue番号を受け取り、詳細を取得：

```bash
!gh issue view [ISSUE_NUMBER]
```

以下の情報を自動抽出：
- タイトル
- 本文（実装内容）
- ラベル（feature/bug/refactor等）
- 担当者

進捗表示: `🔍 [1/9] Issue #123 を読み取り中...`

### ステップ2: ブランチ自動作成

現在のブランチを確認：

```bash
!git branch --show-current
```

master/main にいる場合、Issueから自動的にブランチ名を生成して作成：

```bash
!git checkout -b <type>/<description>
```

ブランチ名生成ルール:
- `feature/add-xxx` - 新機能
- `fix/xxx-error` - バグ修正
- `refactor/xxx` - リファクタリング

進捗表示: `🌿 [2/9] ブランチ作成: feature/add-dark-mode`

### ステップ3: 実装の実行（自動）

Issue内容から実装計画を立て、**ユーザー確認なしで自動実行**：

1. 必要なファイルを特定
2. 既存コードを確認
3. コードを実装（Read, Write, Edit ツール使用）
4. 必要に応じてライブラリをインストール

進捗表示: `⚙️ [3/9] 実装中... (3/5 ファイル完了)`

### ステップ4: テストとビルド（自動）

実装後、自動でテスト・ビルドを実行：

```bash
!npx tsc --noEmit
!npm run lint 2>/dev/null || echo "Lint skipped"
!npm test 2>/dev/null || echo "Test skipped"
!npm run build
```

進捗表示: `🧪 [4/9] テスト・ビルド実行中...`

### ステップ5: エラー自動修正（必要時）

エラーが発生した場合、**最大3回まで自動修正を試行**：

1. エラーメッセージを解析
2. 原因を特定
3. コードを修正
4. 再度テスト

修正試行回数を表示: `🔧 [4/9] エラー修正中... (試行 1/3)`

**3回試行しても失敗した場合**: ユーザーに報告して中断

### ステップ6: コミット（自動）

既存のコミット履歴からスタイルを学習し、自動でコミットメッセージを生成：

```bash
!git log --oneline -n 10
```

コミット実行：

```bash
!git add .
!git commit -m "$(cat <<'EOF'
<type>: <タイトル>

<詳細な説明>

Closes #<ISSUE_NUMBER>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

進捗表示: `💾 [6/9] コミット完了`

### ステップ7: プッシュ（自動）

リモートに自動プッシュ：

```bash
!git push -u origin $(git branch --show-current)
```

進捗表示: `📤 [7/9] プッシュ完了`

### ステップ8: PR作成（自動）

コミット履歴とIssue内容から、自動でPRタイトルと本文を生成：

```bash
!gh pr create --title "<type>: <タイトル>" --body "$(cat <<'EOF'
## Summary
- Issue #<ISSUE_NUMBER> の実装

## Changes
- [変更内容を箇条書き]

## Test plan
- [x] 型チェック通過
- [x] Lint通過
- [x] ビルド成功

Closes #<ISSUE_NUMBER>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

進捗表示: `🔀 [8/9] PR作成完了: #456`

### ステップ9: Issueコメント（自動）

Issueに完了コメントを追加：

```bash
!gh issue comment [ISSUE_NUMBER] --body "✅ 実装完了しました。

PR: #<PR_NUMBER>

PRマージ時に自動的にクローズされます。

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

進捗表示: `✅ [9/9] 完了！Issue #123 → PR #456`

### 完了サマリー

最後に以下を表示：

```
🎉 全自動実装が完了しました！

Issue: #123 - ダークモード実装
ブランチ: feature/add-dark-mode
PR: #456

次のステップ:
1. PR をレビュー: https://github.com/...
2. 承認後、マージすると Issue が自動クローズされます
```

---

## エラーハンドリング

### 型エラー・Lintエラー

自動修正を最大3回試行：
1. エラー箇所を特定
2. コードを修正
3. 再テスト

### ビルドエラー

依存関係の問題の場合：
1. `npm install` を自動実行
2. 再ビルド

### PR作成エラー

ブランチが既に存在する場合：
1. 既存PRを確認
2. ユーザーに報告

### 致命的エラー

自動修正できない場合：
1. 進捗を保存（コミットまで）
2. エラー内容をユーザーに報告
3. 手動対応を依頼

---

## 使用例

### 例1: 新機能実装

```
/auto-implement-issue 123
```

→ Issue #123（ダークモード実装）を自動で実装

### 例2: バグ修正

```
/auto-implement-issue 45
```

→ Issue #45（ログインエラー）を自動で修正

---

## 通常版との違い

| 項目 | /implement-issue | /auto-implement-issue |
|------|-----------------|----------------------|
| 実装計画の確認 | ✅ ユーザーに確認 | ❌ 自動実行 |
| エラー修正 | 👤 手動 | 🤖 自動（最大3回） |
| コミットメッセージ | 👤 確認 | 🤖 自動生成 |
| PR作成 | 👤 確認 | 🤖 自動作成 |
| 推奨用途 | 重要な変更 | 小〜中規模の変更 |

---

## 注意事項

- **完全自動実行**: ユーザー確認を最小限にします
- **エラー自動修正**: 最大3回まで自動修正を試みます
- **重要な変更**: 大規模な変更は `/implement-issue` を推奨
- **セキュリティ**: 実装前にコードを自動レビューします
- **GitHub CLI認証**: `gh auth status` で事前確認

---

## トラブルシューティング

### 自動修正が失敗する場合

手動で修正するか、`/implement-issue` を使用してください。

### PRマージ後もIssueが開いたまま

コミットメッセージに `Closes #123` が含まれているか確認してください。

### 権限エラー

リポジトリへの書き込み権限を確認してください。
