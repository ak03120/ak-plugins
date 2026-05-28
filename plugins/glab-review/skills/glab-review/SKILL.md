---
name: "glab-review"
description: "GitLab Merge Request の差分の特定行に inline discussion または draft note を投稿し、複数の指摘を draft note としてキューに積んで bulk_publish でまとめて『Submit review』として publish するスキル。gitlab.com と self-managed インスタンス（例: gitlab.tokyo.optim.co.jp）の両方に対応。`glab api --field 'position[new_line]=N'` で position が null となり汎用 MR コメントとして silent failure する落とし穴を回避する。GitLab MR を行アンカー付きでレビューしたい、インラインコメントでスレッドを作成したい、draft でまとめてレビュー投稿したい、と頼まれたときに使う。"
---

<!-- markdownlint-disable MD013 -->

# glab-review — GitLab MR 行アンカー付きレビュー

MR の差分ビュー上の特定行にアンカーした discussion スレッドを投稿する。

本スキルは **self-contained** — ヘルパースクリプトを置かず、`glab api` 呼び出しと Bash から実行する小さなインライン Python ヒアドキュメントだけで完結する。各ブロックをコピーし、プレースホルダー（`<MR_URL>`、`<FILE_PATH>`、`<NEW_LINE>`）を埋めて実行する。

2 種類のワークフローを提供:

- **バッチ（複数指摘のレビューでの推奨）:** 各指摘を draft note としてキューに積み、GitLab の「Submit review」ボタンと同様に一括で publish する。著者への通知は N 件ではなく 1 件にまとまる。
- **即時投稿:** スレッドを 1 件すぐに投稿する。単発の指摘に向く。

## なぜこのスキルが必要か

GitLab の discussion / draft API は入れ子の `position` オブジェクト（`base_sha`、`head_sha`、`start_sha`、`new_path`、`new_line` …）を要求する。一見動きそうなアプローチがいくつか silent failure する:

| アプローチ | 結果 |
| --- | --- |
| `glab api --field "position[new_line]=217"` | 201 が返り note も作成されるが、`position` が `null`。スレッドは差分行ではなく汎用 MR コメントとして表示される。**silent failure — 使わないこと。** |
| `glab api --input <json> --field …` 併用 | HTTP 415: "The provided content-type '' is not supported." |
| `glab api --input <json> --header "Content-Type: application/json"` | ✅ **これを使う。** position が保持される。 |
| `curl -F "position[new_line]=217"`（multipart form） | これも動くが避ける — グローバルルールで `glab` を生 HTTP より優先するため。 |

以下のすべてのスニペットは `glab api` を JSON ボディで叩き、**レスポンスの `position` が null でないことを検証する**。API は un-anchored な note にも 201 を返すため、このチェックは必ず入れる。

## ツール: curl ではなく `glab` を使う

ユーザーのグローバルルール（`~/.claude/CLAUDE.md`）に従い、GitLab 関連は CLI（`glab`）を生 API 呼び出し（`curl`）より優先する。本スキルのすべてのスニペットは MR メタデータ、discussions、draft_notes、bulk_publish、draft ごとの publish、削除のすべてで `glab api` を使う。

`curl` への降格は `glab` が利用できない、または `glab` で表現できない場合だけ。今のところ必要になったことはなく、`glab api --input <file> --header "Content-Type: application/json"` で本スキルが必要とするすべてのエンドポイントを賄える。`curl` に手が伸びる前に、`glab api` の `--method`、`--header`、`--input` フラグで足りるか先に確認すること。

`glab api --hostname <host> <path>` はトークンを `glab auth` から取得するので、PAT のコピペは不要。`glab auth status` にホストが無ければ `glab auth login --hostname <host>` で追加する。

## 一時ファイルの扱い

本スキルは bash の `mktemp` で取得した一時ファイル（`$BODY` / `$DRAFT_JSON` / `$DISC_JSON`）を介してボディや JSON ペイロードを受け渡す。`mktemp` は OS 依存のパス（macOS / Linux は `$TMPDIR` 配下、Git Bash on Windows / WSL は `/tmp/` 相当）を返すので、Windows を含めて環境依存のパスをハードコードせずに済む。複数指摘を並列実行しても衝突しない利点もある。

## 0 — Setup（MR ごとに 1 回実行）

MR URL を shell 変数にパースし、`position` で必要となる 3 つの SHA を取得する。続くスニペットで `$HOST` / `$PROJECT_ENC` / `$MR_IID` / `$BASE_SHA` / `$START_SHA` / `$HEAD_SHA` を再利用する。

URL のパースは Python に委ねて、bash と zsh で同じ動作にしている（zsh は `BASH_REMATCH` を埋めない）。

すべての変数は `export` しており、続く Python ヒアドキュメントから `os.environ` 経由で参照できる。

```bash
MR_URL=https://gitlab.tokyo.optim.co.jp/<group>/<project>/-/merge_requests/<iid>

eval "$(MR_URL="$MR_URL" python3 <<'PY'
import os, re, sys, urllib.parse as u
m = re.match(r'^https?://([^/]+)/(.+)/-/merge_requests/(\d+)/?$', os.environ['MR_URL'])
if not m:
    print('echo "ERR: bad MR_URL — expected https://<host>/<group>/<project>/-/merge_requests/<iid>" >&2; false')
    sys.exit()
host, project, iid = m.groups()
print(f'export HOST={host}')
print(f'export PROJECT_PATH={project}')
print(f'export MR_IID={iid}')
print(f'export PROJECT_ENC={u.quote(project, safe="")}')
print(f'export MR_URL={os.environ["MR_URL"]}')
PY
)"

eval "$(glab api --hostname "$HOST" "projects/$PROJECT_ENC/merge_requests/$MR_IID" \
  | python3 -c "import json,sys; r=json.load(sys.stdin)['diff_refs']; [print(f'export {k.upper()}={v}') for k,v in r.items()]")"

echo "host=$HOST  project=$PROJECT_PATH  iid=$MR_IID"
echo "base=$BASE_SHA  head=$HEAD_SHA"
```

## ワークフロー A — バッチレビュー（推奨）

### A1 — draft を 1 件キューに積む（指摘ごとに繰り返す）

ボディは `mktemp` で確保した一時ファイル（`$BODY`）に書き出す。書き方は `cat <<'EOF'` でも `Write` ツールでもエディタでもよい。書いたら以下のブロックを実行する。

````bash
FILE_PATH=Shared/Views/LauncherView.swift
NEW_LINE=217
# OLD_LINE=...   # 変更なしのコンテキスト行ではこちらも設定する。削除行ではこちらだけ設定する

BODY=$(mktemp)
DRAFT_JSON=$(mktemp)

# 1. body — クォート付きヒアドキュメント（<<'EOF'）で markdown 内の `` や $vars を保持
cat > "$BODY" <<'EOF'
L217 で `uuid` が nil の時、dispatch 側 L1083 の `case .geoPoint(nil, _):` で
action が捨てられて callback が発火しません。

```diff
- case .geoPoint(nil, _):
+ case .geoPoint(nil, let action):
```
EOF

# 2. JSON ペイロードを安全に組み立てる（Python がエスケープを処理）。
#    $BASE_SHA などは §0 で export 済みなので os.environ から読める
export FILE_PATH NEW_LINE OLD_LINE BODY
python3 <<'PY' > "$DRAFT_JSON"
import json, os
pos = {
    "base_sha":      os.environ["BASE_SHA"],
    "start_sha":     os.environ["START_SHA"],
    "head_sha":      os.environ["HEAD_SHA"],
    "position_type": "text",
    "old_path":      os.environ["FILE_PATH"],
    "new_path":      os.environ["FILE_PATH"],
}
if os.environ.get("NEW_LINE"): pos["new_line"] = int(os.environ["NEW_LINE"])
if os.environ.get("OLD_LINE"): pos["old_line"] = int(os.environ["OLD_LINE"])
print(json.dumps({"note": open(os.environ["BODY"]).read(), "position": pos}))
PY

# 3. draft として POST し、position が保持されているか検証
glab api --hostname "$HOST" "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes" \
  --method POST --header "Content-Type: application/json" --input "$DRAFT_JSON" \
  | python3 -c "
import json, sys
n = json.load(sys.stdin)
pos = n.get('position') or {}
if pos.get('new_line') is None and pos.get('old_line') is None:
    print('ERR: draft created but position is missing — un-anchored!', file=sys.stderr)
    print('draft_id:', n.get('id'), file=sys.stderr); sys.exit(1)
print(f\"OK draft {n['id']}  {pos.get('new_path')}:{pos.get('new_line') or pos.get('old_line')}\")
"
````

### A2 — キューの中身をサニティチェック

```bash
glab api --hostname "$HOST" "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
if not data: print('(no pending drafts)'); sys.exit(0)
print(f'{len(data)} pending draft(s):')
for d in data:
    pos = d.get('position') or {}
    path = pos.get('new_path') or pos.get('old_path') or '(general)'
    line = pos.get('new_line') or pos.get('old_line') or '-'
    note = (d.get('note') or '').replace('\n',' ')[:80]
    print(f\"  id={d['id']}  {path}:{line}  {note}\")
"
```

### A3 — すべての draft を 1 件のレビューとして publish

まず `bulk_publish` を試す（=「Submit review」ボタン相当、著者への通知は 1 件）。サーバが 500 を返す場合（self-managed インスタンスで散発的に発生）は、各 draft を個別に publish するフォールバックに切り替える。

```bash
if ! glab api --hostname "$HOST" \
      "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes/bulk_publish" \
      --method POST > /dev/null 2>&1
then
    echo "bulk_publish failed → per-draft fallback" >&2
    glab api --hostname "$HOST" "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes" \
      | python3 -c "import json,sys; [print(d['id']) for d in json.load(sys.stdin)]" \
      | while read DID; do
            glab api --hostname "$HOST" \
                "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes/$DID/publish" \
                --method PUT > /dev/null \
                || echo "  ERR: draft $DID failed to publish" >&2
        done
fi
echo "published — refresh the MR's Changes tab to see the inline threads"
```

### A4 — publish 前に個別の draft を削除

```bash
glab api --hostname "$HOST" \
    "projects/$PROJECT_ENC/merge_requests/$MR_IID/draft_notes/<DRAFT_ID>" \
    --method DELETE
```

「レビュー全体を破棄する」エンドポイントは存在しない — キューに積んだレビューを破棄したい場合は draft を 1 件ずつ削除する。

## ワークフロー B — 即時投稿

単発の指摘では draft 段階をスキップする。形は同じだが以下が異なる:

- POST 先が `…/draft_notes` ではなく `…/discussions`
- ボディフィールドが `note`（draft）ではなく `body`（discussion）
- 1 回の呼び出しごとに可視のスレッドが 1 件作成され、それぞれが個別の通知を発する

```bash
FILE_PATH=Shared/Views/LauncherView.swift
NEW_LINE=217

BODY=$(mktemp)
DISC_JSON=$(mktemp)

cat > "$BODY" <<'EOF'
コメント本文…
EOF

export FILE_PATH NEW_LINE OLD_LINE BODY
python3 <<'PY' > "$DISC_JSON"
import json, os
pos = {
    "base_sha":      os.environ["BASE_SHA"],
    "start_sha":     os.environ["START_SHA"],
    "head_sha":      os.environ["HEAD_SHA"],
    "position_type": "text",
    "old_path":      os.environ["FILE_PATH"],
    "new_path":      os.environ["FILE_PATH"],
}
if os.environ.get("NEW_LINE"): pos["new_line"] = int(os.environ["NEW_LINE"])
if os.environ.get("OLD_LINE"): pos["old_line"] = int(os.environ["OLD_LINE"])
print(json.dumps({"body": open(os.environ["BODY"]).read(), "position": pos}))
PY

glab api --hostname "$HOST" "projects/$PROJECT_ENC/merge_requests/$MR_IID/discussions" \
  --method POST --header "Content-Type: application/json" --input "$DISC_JSON" \
  | python3 -c "
import json, os, sys
d = json.load(sys.stdin); n = d['notes'][0]; pos = n.get('position') or {}
if pos.get('new_line') is None and pos.get('old_line') is None:
    print('ERR: discussion created but position is missing — un-anchored!', file=sys.stderr); sys.exit(1)
print(f\"OK discussion {d['id']} note {n['id']}  {pos.get('new_path')}:{pos.get('new_line') or pos.get('old_line')}\")
print(f\"  link: {os.environ['MR_URL'].rstrip('/')}/diffs#note_{n['id']}\")
"
```

publish 済みの note を削除する（後始末や訂正のため）:

```bash
glab api --hostname "$HOST" \
    "projects/$PROJECT_ENC/merge_requests/$MR_IID/discussions/<DISCUSSION_ID>/notes/<NOTE_ID>" \
    --method DELETE
```

## 行種別ごとのバリアント（両ワークフロー共通）

`NEW_LINE` / `OLD_LINE` は 1 始まりの **ソースブランチ上のファイル行番号** で、`grep -n` の出力と一致する。`@@ -191,7 +213,30 @@` のような hunk オフセットでは*ない*。

| 差分の行種別 | 設定する変数 | 例 |
| --- | --- | --- |
| 追加行（新規コード） | `NEW_LINE` のみ | `NEW_LINE=217` |
| 削除行（削除されたコード） | `OLD_LINE` のみ | `OLD_LINE=50` |
| コンテキスト行（変更なし） | 両方 | `NEW_LINE=100 OLD_LINE=95` |

## 推奨されるエンドツーエンドのレビューワークフロー

1. **MR メタデータと差分を読む** — `glab mr view <iid>` および `glab mr diff <iid>`。
2. **既存の discussion を読み**、bot や人間のコメントと重複する指摘を避ける —
   `glab api projects/$PROJECT_ENC/merge_requests/$MR_IID/discussions`。
   著者が返信ですでに対応している論点は蒸し返さない。
3. **Setup（§0）を 1 回実行**。
4. **指摘ごとに** 一時ファイル（`$BODY`）にボディを書いて §A1 を実行。
5. **§A2** でキューをサニティチェック。
6. **§A3** で 1 件のレビューとして送信。
7. **ユーザーにサマリを返す** — `(file:line, タイトル)` のリストと MR の Changes タブへのリンクを添える。

## 前提条件

- 対象ホストに認証済みの **`glab` CLI**（`glab auth status` で確認）。
  グローバルルールに従い、GitLab 関連は `curl` に降格する前に必ず `glab` を使う。
  必要なら `glab auth login --hostname <host>` でホストを追加。
- `python3`（標準ライブラリのみ — JSON 整形と URL エンコードにインラインで使う）。
- `mktemp`（一時ファイル作成。macOS / Linux / Git Bash on Windows / WSL に同梱）。

## 落とし穴

- **silent な un-anchored ノート。** `glab api --field "position[new_line]=N"`
  は 201 を返すが、レスポンスの `position` は `null` — スレッドは差分行ではなく
  汎用 MR コメントとして表示される。上記すべてのスニペットは応答を確認し、
  `position.new_line` / `position.old_line` が欠落していればエラーで停止する。
  **このチェックは絶対に省略しない。**
- **行番号はソースブランチ上のファイルの行番号。** 差分には
  `@@ -191,7 +213,30 @@` と表示されるが、これは hunk アンカー。`new_line` には
  MR のソースブランチ上のファイルにおける実際の行番号を渡す必要がある。
  ローカルにチェックアウトしたブランチに対して `grep -n <pattern> <file>` を打つのが最速。
- **`bulk_publish` は self-managed GitLab で 500 を返すことがある。**
  gitlab.tokyo.optim.co.jp で観測（毎回ではなく散発的 — インスタンスや
  ペイロードに依存する模様）。§A3 の draft 個別フォールバックは今のところ全ケースで動作している。
- **draft はユーザー単位。** §A2 で見えるのは *自分の* draft のみ — 他のレビュアーの
  キューに積まれた draft はこの API では見えない。
- **`glab mr note` は行アンカーできない。** 汎用 MR コメントしか投稿できないので、
  インラインスレッドを意図しているときに使ってはいけない。
- **`old_path` と `new_path` は同じ文字列にする** — base と head の両方に存在する
  ファイル（通常はこちら）の場合。スニペットでは両方をセットしている。
- **MR URL には `/-/merge_requests/<iid>` を含める** — `-/` 区切りが必要。
  MR 概要ページのブラウザアドレスバーからコピーする。
- **ボディはクォート付きヒアドキュメント（`<<'EOF'`）で書く。** クォート形にすることで
  markdown 本文内のバッククォート、`$vars`、コードフェンスを shell が展開しない。
  JSON 構築用の Python ヒアドキュメントも *クォート付き* (`<<'PY'`) — 値は
  shell 補間ではなく環境変数経由で渡しているのでエスケープ問題は起きない。

## トラブルシューティング

| 症状 | 原因 / 対処 |
| --- | --- |
| `ERR: … position is missing — un-anchored!` | un-anchored なノートが作成された。削除する（`…/discussions/<DID>/notes/<NID> --method DELETE` または `…/draft_notes/<ID> --method DELETE`）。原因は通常、SHA が誤っているか、head 版のファイルに該当行が存在しない、のいずれか。 |
| `400 Note {:line_code=>["can't be blank"]}` | `position` が不正 — 通常、`new_line` が差分 hunk に含まれない行を指していて、かつ `old_line` を指定していない。変更なしのコンテキスト行に対しては `NEW_LINE` と `OLD_LINE` の両方を設定する。 |
| §0 で `bad MR_URL` | URL に `/-/` が無いか、末尾の `/merge_requests/<iid>` が欠けている。ブラウザからコピーし直す。 |
| `glab api` から `404` | URL のホストが `glab auth` のエントリと一致しないか、プロジェクトパスが間違っている。`glab auth status` とグループ/プロジェクト部分を確認。 |
| `bulk_publish failed → per-draft fallback` | 一部の self-managed インスタンスで既知の挙動。フォールバックでひと通り publish されるはず。特定の draft が個別 publish でも失敗した場合はキューに残るので、§A2 で一覧して再試行・編集・削除を決める。 |
| 誤ったものを publish してしまった | §A1/§B の出力に `draft_id` / `discussion_id` / `note_id` が出るので、それを上記の DELETE 呼び出しに渡して削除する。 |
