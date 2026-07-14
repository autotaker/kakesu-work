---
task_id: "TASK-0002"
status: complete
reviewer_agent: "reviewer-agent-terra-medium"
reviewed_commit: "cfbf29973e9b35a24c85a5149f08940f1b2f6374"
decision: fail
make_check: fail
reviewed_at: "2026-07-14T11:37:32+10:00"
---

# TASK-0002 REVIEW RESULT

## 対象

- ブランチ: `task/TASK-0002-codex-agent-model-routing`
- コミット: `cfbf29973e9b35a24c85a5149f08940f1b2f6374`
- Task / PLAN / QA PLAN: 承認済みの`sol-high` PLANおよびrevision 1のQA PLANを照合した。

## 実行した検査

| コマンド | 結果 | 備考 |
|---|---|---|
| `node --test scripts/task/agent-routing.test.mjs scripts/task/development-process.test.mjs` | pass | 16件pass、0件fail。 |
| `make work-check WORK_ROOT=/Users/autotaker/git/agent-harness-work` | pass | Epic 1件、Task 2件、Wiki page 7件を検証。 |
| `node scripts/task/check-task.mjs --work-root /Users/autotaker/git/agent-harness-work --task TASK-0002` | pass | 現在のDEV gateを検証。 |
| `git diff --check 2af86dd..cfbf299` | pass | 空白エラーなし。 |
| `make check` | fail | `memory`の`hatchling`取得時にDNS解決できず終了。実装判定用の決定的Node試験は上記で完走したが、必須gateとしてはPASSにできない。 |

## 受け入れ条件の確認

| 条件 | 結果 | 根拠 |
|---|---|---|
| 固定role routing、DEV選定、explorer contract、adapter drift、evidence | pass | canonical TOML、routing実装、16件の決定的Node試験を照合。 |
| parent-owned commitと不正なchild変更の失敗処理 | fail | scope違反・child失敗時、親が変更を作業ツリーから復元せずlockを解放する（R-001）。QA-010と共有worktreeのclean開始条件を満たさない。 |
| 回帰ゲート | fail | `make check`が環境上のDNSエラーで未完走。 |

## 指摘

| ID | 重大度 | 状態 | 内容 | 根拠 |
|---|---|---|---|---|
| R-001 | P1 | open | `run-work-agent.mjs`と`run-wiki-agent.mjs`はchildのscope外編集、stage、nonzero終了などを検出した後、HEADが不変なら`git reset --mixed <beforeHead>`だけを実行する。mixed resetはindexを戻すだけで作業ツリーの不許可変更を残すため、失敗したchildが`backlog.yaml`等を編集するとdirtyなwork repositoryのままlockが解放される。次回launcherのclean前提を壊し、親がscope検査するという契約にも反する。失敗時にchild開始時の状態へ作業ツリーとindexを安全に復元し、その負例をsubprocess/integration testで証明する必要がある。 | `scripts/task/run-work-agent.mjs:107-114`、`scripts/task/run-wiki-agent.mjs:91-96`、`scripts/task/agent-routing.mjs:180-188`。既存試験は`validateChildOutcome`のthrowだけを確認し、実launcher後のclean状態を確認していない（`scripts/task/agent-routing.test.mjs:84-91`）。 |

## 残存リスク

- `make check`の失敗は`hatchling`取得時のDNS解決不能によるenvironment defectとして観測した。ネットワーク/依存取得が利用可能な環境で再実行が必要である。
- R-001解消後、QA-010のchild nonzero、scope外編集、hook/validation failureごとに、HEADだけでなくindex・working treeのclean復元とlock解放を統合試験で確認する必要がある。

## 結論

`fail`。R-001（P1）が未解消であり、必須の`make check`もPASSしていないため、main Agentはmergeへ進めない。
