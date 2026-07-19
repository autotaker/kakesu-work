---
task_id: "TASK-0028"
status: draft
completed_at: ""
safety_checks:
  process_tests: pending
  contract_scope: pending
  docs_lint: pending
  make_check: pending
safety_checked_at: ""
safety_check_digest: ""
safety_candidate_tree: ""
safety_merge_tree: ""
---

# TASK-0028 HANDOVER

## 成果

- SQLite schema v2、active owner一意制約、明示lifecycle遷移、終端owner解放を実装した。
- state/event/terminal releaseを一transactionへ閉じ、競合・拒否・故意失敗でpartial writeを残さない。

## candidate-bound DEV証跡

- `candidate_commit`: `4e89916d3e63139c2163dd2d39ae912c2b6a6394`
- `candidate_tree`: `69339f5614629e33a0ecd67e731537970db41816`

| ケース ID | コマンド/テスト | 環境/フィクスチャ | cache条件 | exit | 成果物 ダイジェスト | 未実施理由 |
|---|---|---|---:|---:|---|---|
| QA-001〜008 | `go test -count=1 ./internal/control`; `go test -count=20 ./internal/control`; `go test -race ./internal/control`; `make check` | case別temporary file SQLite、2 Store/2 sql.DB | focusedはcache無効、反復20回 | 0 | 4-file SHA-256は主要変更に記載 | なし |

- QAへ渡すネガティブ検出証拠、テスト弱体化の有無を判定できる差分ダイジェスト: candidate binary diff SHA-256 `785087e0f91e6b2e87ed18b02e0566f1cdf478f5912700fe97432b531e1776cf`。既存test削除なし。

## 主要な変更

- `store.go`: schema v2 migration、active owner partial unique index、busy/locked typed conflict。SHA-256 `e6c292079d1ca9db1678797a803a9507b061e568457c95c820f3ea046e4d8ae4`。
- `lifecycle.go`: 7 state/13 edge、expected-state CAS、atomic event/release。SHA-256 `f7b4548ca2bdf2e1ce817678e3c4757199cef210324c2c1eae3dc98e695f0141`。
- tests: 全49 pair、owner保持/再利用、全terminal failure stage、v1 dirty migration、二接続競合。`store_test.go` `3c626f...be6`、`lifecycle_test.go` `5183ee...3b12`（DEV完了時digest、candidate差分を正本とする）。

## 検証結果

- focused PASS、20回反復PASS（28.412秒）、race PASS（5.514秒）、Main `make check` PASS、`git diff --check` PASS。
- production追加187 physical lines（store net +23、lifecycle 164）で承認上限240以内。

安全契約変更では`safety_checks`を`process_tests`、`contract_scope`、`docs_lint`、`make_check`の4項目だけとし、すべて`pass`を記録する。`safety_check_digest`は案 tree、merge tree、上記順の検査名と結果を`key=value`の改行区切りで正規化し、末尾改行を含めたSHA-256とする。第2親の案 treeとmerge treeもフロントマターへ記録する。製品用のREVIEW/QA PASS、製品用の完了HANDOVER、Wiki取込記録を代用証跡として作成しない。

## 判断

- DB制約を排他の正本とし、process mutex、自動retry、TASK-0029 versioned mutationは追加しない。
- 選択: `not-applicable | qa_carry_forward | focused-rerun | full-rerun`
- Main判断の旧新コミット/tree、全差分とダイジェスト、影響ケース集合、レビュアー/`make check`証拠、理由: TODO
- carry-forward時の`QA_RESULT.md` `CF-1`から`CF-7`: `not-applicable | complete | incomplete`
- 影響QAケース集合が空でない場合の再実行証拠: TODO
- `merge_tree`と案 treeの比較: `pending`

## 既知の制約と未解決事項

- なし

環境依存ケースがある場合、install/deploy/config生成、実権限、外部作用、実restart/ロールバック/クリーンアップのマージ後確認を省略しない。実環境または安全なクリーンアップが不明なケースはblockedとして残す。

## 運用上の注意

- なし

## Wikiへ引き渡す知識

### 再利用可能な知識

- SQLite partial unique indexでactive assignmentだけを一意化し、terminal transaction内で`released_at`を設定するとownerを安全に再利用できる。
- v1→v2 migrationは既存重複ownerでunique index作成に失敗した場合、DDLとversion更新を全rollbackする。

### 反例・失敗・注意点

- SQLiteは単一writerなので内部insert後barrierは相互待ちになる。二Storeを同時開始し、DB unique indexを唯一のarbitratorとして反復検証する。

### 更新候補ページ

- Control Task lifecycleとactive owner assignmentの永続不変条件。

## ブートストラップ例外

- 該当なし
