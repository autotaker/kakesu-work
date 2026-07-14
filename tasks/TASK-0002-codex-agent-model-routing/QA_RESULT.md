---
task_id: "TASK-0002"
status: complete
qa_agent: "qa-agent-terra-medium"
tested_commit: "a0661057b56461c1a0e0f01a326d487b094e5ea1"
tested_work_adapter_commit: "27d04061f5f7858e59e61998d28c673d12e31b6c"
decision: fail
tested_at: "2026-07-14T12:31:37+10:00"
---

# TASK-0002 QA RESULT

## 対象

- product `main`: `a0661057b56461c1a0e0f01a326d487b094e5ea1`
- merge parents: `2514a41ce7cc420868960f43f5a1625ee4e06107` / `35fcaf209a65d17acd5d215298ff6175b0671572`（review済みcommitは第2親、親数は2）
- generated work adapter: `27d04061f5f7858e59e61998d28c673d12e31b6c`
- QA PLAN: revision 2、`expectation_changed=false`
- 環境: `qa-agent-terra-medium`、2026-07-14T12:31:37+10:00。両repositoryは検査開始時・終了時ともclean。

## 結果

| ケースID | 結果 | 証跡 | 備考 |
|---|---|---|---|
| QA-001 | `pass` | `node --test scripts/task/agent-routing.test.mjs`: 8/8 pass | canonical role routing完全一致。 |
| QA-002 | `pass` | routing DEV fixture pass、`make task-check TASK=TASK-0002 WORK_ROOT=/Users/autotaker/git/agent-harness-work`: pass | approved `sol-high`、reason/risk evidenceを検証。 |
| QA-003 | `pass` | routing DEV selection/promotion fixture、process DEV-profile fixture: pass | Luna eligibility、promotion、降格拒否を検証。 |
| QA-004 | `pass` | delegation fixture pass、Explorer dry-run JSON | depth=2、threads=2、一問制約、read-only launch contract。live Codex/Explorerは未呼出し。 |
| QA-005 | `pass` | Explorer launcher fixture pass | fixed CLI、closed stdin、write scopeなし、`commit:null`。 |
| QA-006 | `fail` | direct `node scripts/task/agent-routing.mjs --work-root /Users/autotaker/git/agent-harness-work --mode check`: pass; required `make work-config-sync WORK_ROOT=/Users/autotaker/git/agent-harness-work CHECK=1`: fail | required wrapperは既存lock ownerへの `process.kill(pid, 0)` が `EPERM`となり完走不能。 |
| QA-007 | `pass` | fixed-role override/legacy fixture pass | mismatch fail-closed、canonical redundancy/legacy routeを検証。 |
| QA-008 | `pass` | Explorer launcher、process closed-stdin fixture: pass | non-TTY、closed stdinを検証。 |
| QA-009 | `pass` | parent commit preconditions fixture: pass | child stage/commit、failure、scope driftを拒否。 |
| QA-010 | `pass` | rollback fixture: 6 subcases pass | child/scope/stage/commit/hook/validation failureでrollback・no commit。 |
| QA-011 | `pass` | concise/redaction fixture: pass | one-line evidence、secret-like value redactionを検証。 |
| QA-012 | `pass` | process gate fixture pass、`task-check`: pass、actual merge parent inspection | role separation/boundary gateとmainの2-parent `--no-ff` mergeを確認。 |
| QA-013 | `fail` | Node suites 29/29 pass、`make work-check WORK_ROOT=/Users/autotaker/git/agent-harness-work`: pass; `UV_OFFLINE=1 make check`: fail | product `.build/kakesu`の作成がsandboxで `operation not permitted`。必須regression gate未完走。 |

## 発見事項

| ID | FAIL分類 | 影響 | 差し戻し候補 | 内容 |
|---|---|---|---|---|
| QA-ENV-001 | environment defect | QA-006 | main Agent判断 | `.locks/work-repository.lock/owner.json`のPID 64065に対するliveness確認がsandboxで`EPERM`。adapter direct checkはPASSしたが、必須wrapper checkは完走不能。 |
| QA-ENV-002 | environment defect | QA-013 | main Agent判断 | product repositoryがこのQA sandboxのwrite許可外で、`UV_OFFLINE=1 make check`の`go build -o .build/kakesu`が`operation not permitted`。 |

## main Agent判断

- 結論: `pending`
- 差し戻し先: main Agent判断（environment defect。DEVへの自動帰責なし）
- revert / バグ化: main Agent判断
- 判断理由: QA-001〜005、007〜012とdirect adapter checkはPASSしたが、QA-006/013の必須commandが環境権限制約で完走していない。

## 未実施項目

- なし（全ケースを実行し、FAILはenvironment defectとして記録）。

## 結論

`fail`。期待値・QA_PLANは変更していない。main Agentはproduct書込み可能かつlock ownerを観測可能な環境で、`make work-config-sync ... CHECK=1` と `UV_OFFLINE=1 make check` を再実行して最終判断する。
