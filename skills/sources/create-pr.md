---
description: "現在のブランチ変更からPRを自動生成"
argument-hint: [issue-number]
allowed-tools: Read, Bash(git:*), Bash(gh:*), Edit
---

# create-pr

現在のブランチの変更内容を分析し、AIが適切なPRタイトルと本文を生成してプルリクエストを作成します。

**使用方法**:
- `create-pr` - 自動でIssue番号を検出してPR作成
- `create-pr 10` - Issue #10と紐づけてPR作成
- `create-pr #123` - Issue #123と紐づけてPR作成

## 言語ポリシー
- PRタイトル・本文・Summary / Changes / Test plan の箇条書きは必ず日本語で記述する（英語見出しはそのままでOK）。
- テスト結果や失敗理由も日本語で説明し、必要に応じて英語ログを引用する。
- `gh pr create` を実行する前に、生成した日本語テキストをユーザーに共有し確認を得る。
- フッターの `🤖 Generated with ...` は利用中のCLIに合わせて `[Codex CLI]` または `[Claude Code]` を選択する。

## 実行手順

### 0. Issue番号の取得（引数がある場合）

**重要**: ユーザーがIssue番号を指定した場合は、それを優先的に使用します。

- `create-pr 10` → Issue #10を使用
- `create-pr #123` → Issue #123を使用
- `create-pr` → 自動検出（ステップ5.5へ）

**Issue番号が指定された場合**:
1. 指定されたIssue番号を保存
2. ステップ5.5の自動検出をスキップ
3. 指定されたIssue番号でPRタイトル・本文を生成

### 1. ブランチチェック（最重要）

**最初に必ず現在のブランチを確認**：

```bash
!git branch --show-current
```

**master/main ブランチにいる場合の処理**：

1. コミットされていない変更があるかを確認：
```bash
!git status
```

2. 変更がある場合：
   - 変更内容から適切なブランチ名を提案（例: `fix/google-login`, `feat/add-feature`）
   - フィーチャーブランチを作成：
   ```bash
   !git checkout -b <branch-name>
   ```
   - その後、手順3に進む

3. 変更がない場合：
   - エラーメッセージを表示して終了
   - 「master/main ブランチでは直接PRを作成できません。フィーチャーブランチを作成してください。」

### 2. 差分チェック（フィーチャーブランチにいる場合のみ）

コミットされていない変更があるかを確認：

```bash
!git status
```

**重要**: コミットされていない変更がある場合は、`/commit-push` を使用してコミット＆プッシュを行います。

### 3. リモートブランチの確認

リモートブランチの状態を確認：

```bash
!git status
```

### 4. ベースブランチの確認

デフォルトブランチ（master または main）を確認：

```bash
!git remote show origin | grep "HEAD branch" || git symbolic-ref refs/remotes/origin/HEAD
```

### 5. 変更内容の確認

ベースブランチとの差分を確認（<base-branch> は master または main）：

```bash
!git diff <base-branch>...HEAD --stat
```

コミット履歴を確認：

```bash
!git log <base-branch>..HEAD --oneline
```

各コミットの詳細：

```bash
!git log <base-branch>..HEAD --format="%h %s%n%b"
```

### 5.5. Issue番号の決定（重要）

**Issue番号の決定優先順位**:

1. **引数で指定された場合（最優先）**
   - `create-pr 10` で指定された番号を使用
   - 自動検出をスキップ

2. **引数がない場合は自動検出**
   - 以下の順序で検索：

#### 検出方法1: ブランチ名から検出

現在のブランチ名を確認：

```bash
!git branch --show-current
```

ブランチ名のパターン例：
- `feature/add-dark-mode-123` → Issue #123
- `fix/login-error-#45` → Issue #45
- `123-add-feature` → Issue #123
- `issue-78-refactor` → Issue #78

#### 検出方法2: コミットメッセージから検出

コミットメッセージから Issue 番号を抽出：

```bash
!git log <base-branch>..HEAD --format="%s %b" | grep -oE "#[0-9]+" | head -1
```

または `Closes`, `Fixes`, `Resolves` パターンを探す：

```bash
!git log <base-branch>..HEAD --format="%s %b" | grep -iE "(close[sd]?|fix(e[sd])?|resolve[sd]?) #[0-9]+"
```

#### 検出結果

検出されたIssue番号がある場合：
- PRタイトルに「(#123)」を追加
- PR本文に「Closes #123」を自動追加
- Issueの内容を確認して、PR本文のSummaryに反映

検出されなかった場合：
- 通常通りPRを作成（Issue連動なし）

#### Issue情報の取得（番号が検出された場合）

```bash
!gh issue view [ISSUE_NUMBER]
```

Issueのタイトルと本文をPRに反映：
- PRタイトル: `<type>: <Issueタイトル> (#123)`
- PR本文のSummary: Issueの本文から要約

### 6. 既存PRのスタイル確認（オプション）

リポジトリの既存PRを確認してスタイルを把握：

```bash
!gh pr list --limit 5
```

### 7. PRタイトルと本文の生成

以下の方針でPRの内容を生成：

**タイトル**:

**Issue番号が検出された場合**:
- `<type>: <Issueタイトル> (#123)`
- 例: `feat: ダークモード実装 (#123)`
- Issueのタイトルを活用して簡潔に

**Issue番号が検出されなかった場合**:
- 変更の本質を簡潔に表現（50文字以内推奨）
- 既存のPRスタイルに従う
- プレフィックス（feat:, fix:など）は既存スタイルに合わせる

**本文**:

**Issue番号が検出された場合**:
```markdown
## Summary
- Issue #123 の実装
- [Issueの本文から要約した変更内容]

## Changes
- [主要な変更点の詳細]

## Test plan
- [ ] 型チェック通過
- [ ] Lint通過
- [ ] テスト通過
- [ ] ビルド成功
- [ ] 動作確認完了

Closes #123

🤖 Generated with [Codex CLI](https://openai.com/codex)
```

**Issue番号が検出されなかった場合**:
```markdown
## Summary
- [変更内容の概要（箇条書き、2-5項目）]

## Changes
- [主要な変更点の詳細]

## Test plan
- [ ] 型チェック通過
- [ ] Lint通過
- [ ] テスト通過
- [ ] ビルド成功

🤖 Generated with [Codex CLI](https://openai.com/codex)
```

### 8. プッシュ確認

リモートにプッシュされているか確認し、未プッシュのコミットがある場合はプッシュ：

```bash
!git push -u origin $(git branch --show-current)
```

### 9. PR作成

```bash
!gh pr create --title "PRタイトル" --body "$(cat <<'PRBODY'
## Summary
- 変更内容1
- 変更内容2

## Changes
変更の詳細説明

## Test plan
- [ ] テスト項目1
- [ ] テスト項目2

🤖 Generated with [Codex CLI](https://openai.com/codex)
PRBODY
)"
```

ベースブランチを指定する場合：

```bash
!gh pr create --base <base-branch> --title "PRタイトル" --body "..."
```

### 10. PR URLの表示

作成されたPRのURLを表示し、ユーザーに確認を促す。

---

## 注意事項

- **必ずフィーチャーブランチで作業**: master/mainブランチでは直接PRを作成できません
- master/mainブランチにいる場合は、自動的にフィーチャーブランチを作成します
- PR作成前に必ずユーザーに生成したタイトルと本文を確認してもらう
- **コミットされていない変更がある場合は、確認なしで自動的に `/commit-push` を実行します**（フィーチャーブランチにいる場合のみ）
- ベースブランチ（master または main）を自動検出して正しく設定する
- GitHub CLIが認証されていることを確認（`gh auth status`）
- Draft PRとして作成する場合は`--draft`フラグを追加
