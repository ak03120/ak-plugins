---
name: "conventional-commits"
description: "Conventional Commits 1.0.0 仕様 + Angular 規約由来の type 集合に従った短く簡潔なコミットメッセージを 1 行だけで作成し、GPG 署名付きでコミットする。type は feat / fix / chore / docs / style / refactor / perf / test / build / ci / revert から選び、必要なら scope と breaking marker (!) を付与する。本文・footer は使わず必ず 1 行で完結させる。コミットメッセージの作成、git のステージング状態の要約、コミット作成を任されたときに使う。push は依頼されない限り行わない。"
---

<!-- markdownlint-disable MD013 -->

# Conventional Commits コミット

ステージされた変更を Conventional Commits 1.0.0 仕様で簡潔にコミットする。コミットメッセージは短く保ち、GPG 署名を付け、push はユーザーから明示的に依頼されたときだけ行う。

本ドキュメント内のルールは3層に分かれる。混同しないこと。

- **[仕様]** … Conventional Commits 1.0.0 が MUST/SHOULD として規定するもの
- **[慣習]** … Angular 規約や Git の伝統的な慣習（仕様の MUST ではない）
- **[本スキル]** … 本スキル固有のローカル運用ルール

## 仕様

```text
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

- **[仕様]** 仕様は body / footer を任意で許容する。
- **[本スキル]** ただし本スキルではコミットメッセージを必ず 1 行で完結させ、body と footer は使わない。

### type の選び方

| type | 出典 | 用途 |
| --- | --- | --- |
| `feat` | **[仕様]** MUST | 新機能の追加（SemVer の MINOR と対応） |
| `fix` | **[仕様]** MUST | バグ修正（SemVer の PATCH と対応） |
| `docs` | **[慣習]** Angular | ドキュメントのみの変更 |
| `style` | **[慣習]** Angular | コードの意味を変えない整形（空白、フォーマット、セミコロン等） |
| `refactor` | **[慣習]** Angular | バグ修正・機能追加ではないコード変更 |
| `perf` | **[慣習]** Angular | パフォーマンス改善 |
| `test` | **[慣習]** Angular | テストの追加・修正 |
| `build` | **[慣習]** Angular | ビルドシステムや外部依存に関わる変更 |
| `ci` | **[慣習]** Angular | CI 設定・スクリプトの変更 |
| `chore` | **[慣習]** Angular | 上記いずれにも当てはまらない雑務 |
| `revert` | **[慣習]** 公式 FAQ の例示 | 過去コミットの取り消し。仕様の MUST ではなく、公式 FAQ で参考例として挙げられているのみ |

- **[仕様]** 仕様で MUST なのは `feat` と `fix` のみ。それ以外の type を採用するかはツール側の規約（commitlint 等）に依存する。本スキルでは Angular 規約由来の集合を採用する。
- **[仕様]** type と scope は case-sensitive に扱う。本スキルでは小文字に統一する。

### scope

- **[仕様]** 任意。コードベースのセクションを表す**名詞**を丸カッコで囲んで type の直後に置く。例: `feat(parser): ...`、`fix(api): ...`。
- 影響範囲が不明、または広範に及ぶ場合は省略する。

### breaking change

破壊的変更を表す**公式の方法は3通り**ある。

1. **[仕様]** subject の type/scope の直後に `!` を付ける（コロンの前）。例: `feat!: ...`、`refactor(api)!: ...`。
2. **[仕様]** footer に `BREAKING CHANGE: <説明>` を書く。`BREAKING-CHANGE:`（ハイフン形）も同義として認められる。
3. **[仕様]** 上記の両方を併用する。

`!` を付けたコミットは、たとえ footer がなくても破壊的変更を含意する。

- **[本スキル]** 1 行ルールを優先し、`!` のみを使う。`BREAKING CHANGE:` footer は使わない。subject だけで意図が伝わらない場合は、description を磨き直してから付ける。

### footer（参考）

本スキルでは footer を使わないが、公式仕様の規定を以下に示す（合流先のツールが要求した場合のため）。

- **[仕様]** footer は git trailer 形式の `word-token: value` または `word-token #value`。トークン内の単語はハイフン区切り（例: `Reviewed-by: ...`、`Refs: #123`）。
- **[仕様]** `BREAKING CHANGE` だけは例外で、トークン内に空白を許容する。`BREAKING-CHANGE:` も同義。
- **[仕様]** `BREAKING CHANGE` トークンは大文字必須。type と scope は case-sensitive。

## 作業手順

1. `git status` でステージ済み・未ステージの変更を把握する。
2. `git diff --cached`（必要なら `git diff`）で変更内容を確認する。
3. 直近のコミット履歴 `git log --oneline -10` でリポジトリのコミット文化を把握する。
4. 変更の主目的から type を1つだけ選ぶ。type、scope、breaking marker、その他の細則で判断に迷う場合は <https://www.conventionalcommits.org/> で公式仕様を直接参照する。
5. description は短く一文にまとめる。1 行で説明しきれない場合は、コミットを分割するか description を磨き直すことを優先し、複数行にはしない。
6. GPG 署名設定を確認してからコミットする。

## メッセージの書き方

- **[本スキル]** コミットメッセージは 1 行のみ。本文・footer は使わない。
- **[慣習]** description は命令形・小文字始まり・末尾ピリオドなしで書く。これは Conventional Commits 仕様ではなく Git の伝統的な慣習（[git-commit(1)](https://git-scm.com/docs/git-commit) や Tim Pope の commit message guide 由来）。
- **[慣習]** description は 50 文字以内が目安、最大でも 72 文字を超えない。これも Conventional Commits 仕様ではなく Git の慣習（`git log` の整形幅由来）。
- なぜではなく何が変わったかを書く。
- 1 行に収まらない、または複数の関心事が混ざっているときは、コミット分割を先に検討する。

### 良い例

```text
feat: add daily-report plugin manifest
fix(auth): reject empty refresh tokens
chore: init marketplace
docs(readme): clarify install command
refactor!: drop legacy v1 client
```

### 避けるべき例

- `update files`（type なし、内容も不明瞭）
- `fix: いくつかのバグを修正しました。`（複数件まとめ、敬体、末尾ピリオド）
- `feat: ✨ New feature 🎉`（emoji / gitmoji）
- 改行を含む複数行のコミットメッセージ（本文・footer を付ける運用）

## コミット実行

GPG 署名付きでコミットする。常に 1 行のメッセージのみを使う。

```sh
git commit -m "<type>[scope][!]: <description>"
```

## 安全上のルール

- push は依頼されたときだけ行う。
- `--no-verify` や `--no-gpg-sign` は使わない。
- amend は依頼されたときだけ行う。新規コミットを基本とする。
- 秘密情報やパス、トークンをコミットメッセージに含めない。
- ステージされていない変更を勝手に `git add` で取り込まない。範囲はユーザーに確認する。

## 参考

- Conventional Commits 1.0.0 仕様: <https://www.conventionalcommits.org/>
