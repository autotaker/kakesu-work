---
task_id: "TASK-0025"
status: pending
qa_agent: "qa-agent-terra-medium"
tested_commit: ""
candidate_commit: ""
candidate_tree: ""
merge_tree: ""
decision: pending
tested_at: "2026-07-20"
---

# TASK-0025 QA RESULT

## 対象

- 案 コミット/tree: 未提供。`git worktree list --porcelain` は `/home/ubuntu/git/agent-harness-work` の `main`（`fff3978ad56f6c974bbe3d40632217614e27d720` / tree `6617543cfd462bea63ebea5bd5f8ba3f94ce8f88`）だけを報告し、TASK-0025候補は存在しない。
- `main` / merge tree: 未マージ。上記commitは `task: allocate worktree TASK-0025` であり、`backlog.yaml`の状態と予定branch/worktreeだけを更新している。実装candidateまたはmerge treeではない。
- `merge_tree`はマージ後にMainが記録し、案 QAでは未設定とする:
- QA PLAN 改訂: 2（approved、`expectation_changed=false`）
- 環境: `/home/ubuntu/git/agent-harness-work`、NodeによるExplorer launcherを一度だけread-onlyで試行したが、launcher自身がread-only filesystem上のPATH alias/app-server初期化に失敗した（exit 1）。この環境所見はcandidate不在という結論に依存しない。

## 結果

| ケースID | モード | 対象案 コミット/tree | 結果 | 証跡（コマンド/テスト、環境/フィクスチャ、cache、exit、成果物 ダイジェスト、ネガティブ検出能力、テスト弱体化の有無） | 未実施/blocked理由 |
|---|---|---|---|---|---|
| QA-001 | `focused-rerun` | 未提供 | `pending` | candidate commit/tree、実装差分、DEVのcandidate-bound証跡がないため、分類field・移行・再承認fixtureを実checker経路で実行できない。`git worktree list --porcelain`、`git branch --all --verbose --no-abbrev`、`git log --all -- scripts/task/check-task.mjs scripts/task/development-process.test.mjs`を確認し、TASK-0025候補を発見できなかった。 | DEV candidate未提供 |
| QA-002 | `focused-rerun` | 未提供 | `pending` | product Done gateの正常/偽装negative fixtureを再実行する対象実装なし。 | DEV candidate未提供 |
| QA-003 | `focused-rerun` | 未提供 | `pending` | safety-contract正常系および必須証跡欠落matrixを実checker経路で再実行する対象実装なし。 | DEV candidate未提供 |
| QA-004 | `focused-rerun` | 未提供 | `pending` | TASK-0024 read-only regressionを候補実装に束縛できない。 | DEV candidate未提供 |
| QA-005 | `focused-rerun` | 未提供 | `pending` | candidate/tree、rerun/carry-forward、merge-tree fixtureを評価する候補なし。 | DEV candidate未提供 |
| QA-006 | `evidence-review` | 未提供 | `pending` | `check-task.mjs`およびprocess testはこのworktreeに存在せず、candidate差分、DEV evidence、`make check`、`make task-check`を独立監査できない。テスト弱体化の有無も判定不能であり、PASSにはしない。 | DEV candidate未提供 |

## 発見事項

| ID | FAIL分類 | 影響 | 差し戻し候補 | 内容 |
|---|---|---|---|---|
| QA-BLOCK-001 | `environment_issue` | Explorer補助探索のみ | Main / launcher | 許可された一回のExplorer launcher呼出しは、read-only filesystemでPATH aliases/app-server clientを初期化できずexit 1となった。追加のExplorer呼出しは行わない。これは実装candidateまたはDEV証跡の欠落を代替するものではない。 |

## main Agent判断

- 結論: `pending`
- 差し戻し先:
- revert / バグ化:
- 判断理由: Main所有。QAはcandidate未提供のため受け入れ判定を行っていない。

## 未実施項目

- QA-001〜QA-006。実装candidate、candidate tree、DEVのcandidate-bound HANDOVER、および同一candidateへのREVIEW/QA開始条件が未提供のため未実施。candidateが固定された後、QA PLAN revision 2の全caseをfocused-rerun/evidence-reviewで独立実行する。環境依存caseは計画上ない。

## Main-owned `qa_carry_forward` / 再実行判断

- 選択: `not-applicable`（candidate未提供。carry-forwardの前提となる旧QA PASSもない）
- `CF-1` 旧QA PASSと旧`candidate_commit`/`candidate_tree`の束縛: `not-applicable` / 証拠: 旧QA PASSなし
- `CF-2` 旧新案の全差分と差分ダイジェスト: `not-applicable` / 証拠: 新旧candidateなし
- `CF-3` 変更は実行されない誤字、空白、コメント、リンク、証跡メタデータだけで、製品挙動、ランタイム、テスト、Schema、設定、依存、生成物、外部公開契約または安全契約、受け入れ条件、QA_PLANの意味変更がない: `not-applicable` / 証拠: candidate差分なし
- `CF-4` 影響QAケース集合: `[]`。空でなければcarry-forwardせず該当ケースを再実行する。
- `CF-5` 独立レビュアーによる挙動、テスト、安全性、契約への影響なしの確認と、新案の`make check` PASS: `not-applicable` / レビュアー証拠、コマンド、結果: candidateなし
- `CF-6` QA FAIL、受け入れ条件/QA_PLAN変更、認証認可、秘密、sudo/PAM、IPC/Schema/設定/依存、並行性/ライフサイクル/persistence/エラー/fail-closed、テスト削除/弱体化、影響不明、証跡と評価対象の案/tree不一致が全て偽: `not-applicable`
- `CF-7` Main記録（旧新コミット/tree、全差分とダイジェスト、空の影響ケース集合、レビュアー/`make check`証拠、理由）: candidate提供後にMainが記録

## 結論

`pending`。candidate commit/tree、候補差分、およびDEVのcandidate-bound証跡が未提供のため、QA PLAN revision 2の受け入れ判定は開始していない。架空のPASS、未実行testの代替、またはcandidate不在をDEV起因のFAILとする判断は行わない。
