---
task_id: "TASK-0025"
status: draft
completed_at: ""
---

# TASK-0025 HANDOVER

## 成果

- `task-check`へ製品変更と安全契約変更の分類別完了gateを実装した。
- product gateを維持し、安全契約ではrename/copy、製品path、証跡欠落・偽装をfail-closedで拒否する。

## candidate-bound DEV証跡

- worktree: `/home/ubuntu/git/agent-harness-work/worktrees/TASK-0025-safety-contract-task-check`
- `candidate_commit`: `d3b52d8dfff90a614022ad1fa2b8b4267bd1e133`
- `candidate_tree`: `48ff3037f359d3adaead7b4bde73d3a0bde4557b`

| ケース ID | コマンド/テスト | 環境/フィクスチャ | cache条件 | exit | 成果物 ダイジェスト | 未実施理由 |
|---|---|---|---:|---:|---|---|
| QA-001〜QA-005 | `node --test scripts/task/development-process.test.mjs` | Node 22.23.1、一時Git fixture、39 test | local dependency/cache、外部networkなし | 0 | `52e4a04e7b103116ddfbb10ca55be967166dc6cfada427b7f9d2ed160901310b` | なし |
| QA-006 | `make check` | TASK-0025 product worktree | warm local build/cache、外部networkなし | 0 | candidate差分digestを参照 | なし |

- QAへ渡すネガティブ検出証拠、テスト弱体化の有無を判定できる差分ダイジェスト: `d43eb0856ad5bbe17c95c47be9320fb664eed28a8ca49890362a35140719d8dd`

## 主要な変更

- `change_class`の互換解釈、分類別Done gate、固定証跡field、tree/digest束縛を追加した。
- process fixtureを39 testへ拡張し、rename/copy、非no-ff、担当不一致、分類mirror、理由/時系列、検査集合、digest/tree不一致を検出する。

## 検証結果

- Focused test: 39/39 PASS。
- `make check`: process 50/50を含む全検査PASS。
- `git diff --check`、用語集同期、文書lint: PASS。

## 判断

- Reviewer初回FAILの4件は通常DEV経路で修正し、candidateを更新した。
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

- TODO

### 反例・失敗・注意点

- TODO

### 更新候補ページ

- TODO

## ブートストラップ例外

- 該当なし
