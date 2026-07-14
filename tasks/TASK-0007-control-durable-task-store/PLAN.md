---
task_id: "TASK-0007"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_by: ""
approved_at: ""
approved_dev_profile: "sol-high"
planned_implementation_files: 20
planned_implementation_lines: 1380
estimate_points: 8
---

# TASK-0007 PLAN

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| 原子的Task作成 | SQLite integration testでTask、contract、event、progress、owner参照が全て存在するか全てrollbackする | `docs/01`、`docs/11` |
| owner排他 | 同一Agentへの並行割当で1件だけ成功する | `docs/03` |
| lifecycle | table-driven testが許可遷移だけを受理し、違法遷移で不変である | `docs/02` |
| 復旧 | DB close/reopen後にcontract version、current progress、履歴、状態を再構成できる | `docs/05` §検証と失敗 |

## 関連Wikiと判断

- `docs/01-domain-model.md` §§1–4、`docs/02-task-lifecycle.md`、`docs/03-agent-lifecycle.md`。
- `docs/11-data-model.md`、`docs/13-technology-stack.md` §6、Task-0006のwire契約。
- `schemas/draft-v0/control-plane/{task-contract,task-event,task-command}.schema.json`。

## 設計

### 選択案

`control.db`にversioned migrationを導入する。Control repositoryがSQLiteへの唯一の書込み口となり、Task生成、contract更新、progress更新、status遷移をtransaction化する。`tasks`はcurrent状態、`task_contracts`/`task_events`/`task_progress_history`はappend-only、`task_progress_current`は再開read modelとする。Agentの`current_task_id`はpartial unique indexまたは同等のtransactional constraintで非終端Task一件を強制する。

### 代替案と不採用理由

- Event sourcingのみ: current read modelを毎回再生するとMVP再開経路を複雑化するため不採用。
- mutable contract一行: 旧契約と監査を失うため不採用。
- in-memory mutexだけのowner排他: restart/process競合を守れないため不採用。

### 責務と境界

- migration/DB opener、repository、lifecycle policy、ID/clock、Control application serviceを分離する。
- schema snapshot validatorは境界でcontract/eventを検証し、SQLはFK/unique/checkで集合不変条件を守る。
- CLI/transportはrepository直書きをせずapplication serviceを呼ぶ。

### 不変条件

- Taskは単一owner・単一workspaceを持つ。Agentの非終端ownerは最大1件。
- contract versionとprogress versionは単調増加、event sequenceはTask内で単調増加。
- 状態変更、current state、対応event、必要なprogress/historyは同じtransactionでコミットする。
- terminal Taskは非terminalへ戻さず、違法操作はDBを変更しない。

### 失敗時・移行・互換性

- migrationはtransaction適用・schema version記録・途中失敗rollbackを行う。DB corruption/open failureは起動をfail-fastにする。
- optimistic version競合はretry可能なconflictとして返し、勝手に上書きしない。
- draft-v0 contract revisionはsnapshotに保存する。旧DBの移行はMVP初回migrationのみで、後方互換APIは提供しない。

## 変更予定

見積もり対象は実装コード、Schema、設定ファイルだけとする。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `core/internal/control/db/open.go` | implementation | 80 | SQLite open/pragmas/migration runner |
| `core/internal/control/db/migrations/001_initial.sql` | schema | 180 | Agent/Task/contract/progress/event tables/indexes |
| `core/internal/control/db/migrations_test.go` | implementation | 90 | fresh/upgrade rollback tests |
| `core/internal/control/model.go` | implementation | 120 | domain records/status/errors |
| `core/internal/control/lifecycle.go` | implementation | 95 | explicit transition table |
| `core/internal/control/lifecycle_test.go` | implementation | 110 | allowed/denied transition cases |
| `core/internal/control/repository.go` | implementation | 160 | transaction port/read APIs |
| `core/internal/control/sqlite_repository.go` | implementation | 190 | SQLite implementation |
| `core/internal/control/sqlite_repository_test.go` | implementation | 160 | persistence/constraint tests |
| `core/internal/control/service.go` | implementation | 100 | application service replacement |
| `core/internal/control/service_test.go` | implementation | 120 | create/update orchestration |
| `core/internal/control/progress.go` | implementation | 70 | current/history versioning |
| `core/internal/control/progress_test.go` | implementation | 75 | reopen/history tests |
| `core/internal/control/contract.go` | implementation | 80 | snapshot/version validation |
| `core/internal/control/contract_test.go` | implementation | 75 | version/conflict tests |
| `core/internal/control/ids.go` | implementation | 45 | clock/ID injection |
| `core/internal/control/testdata/*.json` | fixture | 80 | contract/event fixtures |
| `core/internal/app/app.go` | implementation | 50 | DB-backed control construction |
| `core/go.mod` | config | 10 | SQLite driver |
| `core/README.md` | implementation | 40 | control.db operational boundary |

## 見積もり

```text
file_score = ceil(planned_implementation_files / 3)
line_score = ceil(planned_implementation_lines / 200)
estimate_points = 1, 2, 3, 5, 8, 13のうちmax(1, file_score, line_score)以上の最小値
```

## 実装手順

1. Task-0006のwire/schema fixtureを取り込み、domain modelとmigrationを確定する。
2. DB opener/migrationとSQL制約を実装し、fresh/reopen/rollbackを試験する。
3. repositoryと明示lifecycle tableを実装し、contract/progress/eventをatomicにする。
4. application serviceへ接続し、並行owner競合とcrash recovery試験を追加する。

## 検証計画

- `go test ./...`に加え、temp SQLiteでmigration/reopen/parallel contentionを実行する。
- schema fixture検証、`make check`、`git diff --check`。
- Task-0008には実DB-backed message Storeを渡せること、Task-0010にはatomic create portを渡せることをconsumer testで確認する。

## 未解決事項

- TaskStatusの全列挙と初期/終端辺は実装前に`docs/02`の正確な表をrepository testへ転記して確認する。新状態の追加は本Taskの対象外。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV開始を承認した。
