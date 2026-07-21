---
task_id: "TASK-0032"
title: "Agent設定をデフォルト値へ戻す"
status: plan
created_at: "2026-07-22"
---

# TASK-0032 Agent設定をデフォルト値へ戻す

## `Planning input packet`（Main Agent所有）

このsectionをPlannerとQAへ渡す唯一の`planning input packet`とし、各内容をPLAN/QA_PLANへ複製しない。

### 目的

製品・運用リポジトリのproject-scoped `config.toml`から、旧MultiAgentV2の明示設定とproject-level `[agents]` overrideを削除し、現行Codexの組み込み既定値へ委ねる。

### 対象と対象外

#### 対象

- 製品側`.codex/config.toml`の`[features.multi_agent_v2]`とproject-level `[agents]` table全体の削除。
- 製品設定から生成される運用側`.codex/config.toml`でも同じ設定を出力しないようにするadapter生成・drift検査・テストの更新。
- project-level override削除後も`.codex/agents/*.toml`を正本とするcustom role契約と、fallback launcherのmodel/effort固定を維持するために必要な最小限の文書整合。

#### 対象外

- `.codex/agents/*.toml`の削除、role名・model・effort・sandbox設定の変更。
- top-level `model`、`model_reasoning_effort`、`sandbox_mode`の変更。
- user-level `~/.codex/config.toml`、system/managed config、Codex runtime本体の変更。
- PLAN/DEV/REVIEW/QAの分離、Main所有Git、fallback launcherの削除または緩和。

### 受け入れ条件

- [ ] AC-1: 製品側`.codex/config.toml`に`[features.multi_agent_v2]`、`hide_spawn_agent_metadata`、`tool_namespace`が存在しない。
- [ ] AC-2: 製品側`.codex/config.toml`にproject-level `[agents]` tableとその子tableが存在せず、未指定時の`agents.max_threads=6`、`agents.max_depth=1`というCodex既定値へ委ねる。
- [ ] AC-3: 運用側`.codex/config.toml`の決定的生成結果にもMultiAgentV2設定と`[agents]` tableが存在せず、`make work-config-sync CHECK=1`がdriftを検出できる。
- [ ] AC-4: standalone `.codex/agents/*.toml`、固定roleのmodel/effort検査、fallback launcherは維持され、project-level `[agents]` registryへ依存せず検査できる。
- [ ] AC-5: 設定生成・routingテストと`make check`がPASSし、削除した設定を再導入する回帰を検出する。

### 安定した参照

| 参照ID | 対象 | 固定改訂/ダイジェスト | 用途 |
|---|---|---|---|
| REF-1 | product `main` | `b9cb877fda35cbf428f04e09c2175f34387298a4` | Task開始点と現行実装 |
| REF-2 | `.codex/config.toml` | Git blob `ca1cb50ee6d8279a88de514f677b3d295e4903f6` | 削除対象の固定 |
| REF-3 | Codex Manual / Subagents・Config basics | SHA-256 `d09c5e514a6f152be14ad82753de4a987065f78f9b6534f920027e04d133bfb0`、2026-07-22取得 | standalone custom agents、既定`max_threads=6`/`max_depth=1`、`multi_agent=true`既定の確認 |
| REF-4 | DECISION-0004 | `wiki/decisions/DECISION-0004-multiagentv2-role-startup.md` | role分離とMain所有Gitの維持 |

### 依存状態

| 依存 | 状態 (`ready` / `pending`) | planning参照 | `ready`後に固定する値 |
|---|---|---|---|
| なし | `ready` | N/A | N/A |

### 許可パス

- `.codex/config.toml`
- `scripts/task/agent-routing.mjs`
- `scripts/task/agent-routing.test.mjs`
- `docs/development/agent-roles.md`
- `docs/glossary.yml`
- `docs/99-glossary-index.md`
- 運用側生成物: `.codex/config.toml`（製品merge後に`make work-config-sync`だけで更新）

### 完了経路preflight

| 確認対象 | 結果 | コマンドまたは根拠 |
|---|---|---|
| 完了checker | ready | `make task-preflight TASK=TASK-0032`、`make task-check TASK=TASK-0032`、`make check` |
| 権限 | ready with escalation | workspace編集は可能。両リポジトリの`.git`書込みはmanaged approvalが必要であることをTask起票時に実測済み |
| 依存状態と参照 | ready | 依存Taskなし。REF-1〜REF-4を固定済み |
| 生成物の有無と更新方法 | ready | 運用側`.codex/config.toml`は製品merge後にMainが`make work-config-sync`で生成・commitする |
| 割当ワークツリー | ready | branch/worktree未存在を確認済み。PLAN/QA_PLAN承認後に`make worktree-create TASK=TASK-0032`を実行する |
| Lapログの書込・Schema・`repository annotation` | ready before Lap | `lap30/events.jsonl`はappend-only、Schemaは`.agents/skills/run-lap30/references/lap30-event.schema.json`、repository annotationは`project.yaml: repository_path=../agent-harness`を使う。PLAN/QA_PLAN承認前は`lap_started`を書かない |

### 未決事項

- なし

### `Dependency-ready reconciliation`

<!-- 依存ready時にMainがready参照、planning参照との差分、AC/設計/scope/QAへの影響、再承認結果を追記する。依存なし又は未readyならN/Aとする。 -->

- N/A

## 背景

現行設定は旧`[features.multi_agent_v2]`でtool namespaceとmetadata公開を明示し、project-level `[agents]`でthread/depth上限とrole registryを定義している。現行Codexはsubagent workflowを既定で有効化し、custom agentを`.codex/agents/*.toml`から読み込み、未指定時の`agents.max_threads`を6、`agents.max_depth`を1とする。ユーザー要望に従い、旧実験設定とproject overrideを除去して現行既定へ戻す。

## 検討すべき設計観点

- adapter生成が削除済みtableを再生成しないこと。
- standalone role TOMLをfallback routingの正本として維持し、project-level registry削除とrole契約削除を混同しないこと。
- 運用側生成物の更新を製品merge後の専用`work-config-sync`へ限定すること。
- 設定削除による再起動後の有効化タイミングを環境依存確認として扱うこと。

## 完成の定義

- [ ] 受け入れ条件を満たしている。
- [ ] 選択した`change_class`の完了経路と`make check`を満たしている。
- [ ] 製品変更の場合: 実装、テスト、文書、同一案の独立REVIEW/QA、`merge_tree`確認、環境依存ケース、Wiki取り込みが完了している。
- [ ] 安全契約変更の場合: 独立計画レビュー、契約検査、`no-ff merge`、案/merge tree一致が完了し、製品REVIEW/QA PASSやWiki receiptを代用証跡として作成していない。

## 関連コンテキスト

### 意味 Wiki

- [Codex Agent Model Routing](../../wiki/semantic/schemas/codex-agent-model-routing.md): project-scoped TOMLをrole routingの正本とし、固定roleのmodel/profile/effort overrideは起動前に拒否する。work repositoryのadapterは正本から決定的に生成し、digest付き完全一致検査でdriftを拒否する。
- [MultiAgentV2 Role Startup](../../wiki/decisions/DECISION-0004-multiagentv2-role-startup.md): `agent_type`でroleを選択し、異なるroleのchild起動では`fork_turns="none"`を明示する。設定上のsandbox宣言は実効権限の証明ではない。

### 判断

- 現行の判断はDECISION-0004であり、設定をデフォルトへ戻す変更は、role routing、cross-role起動、子AgentのGit禁止、親のlock/commit所有という既存の境界を維持しなければならない。

### 適用しなかった重要な判断

- [DECISION-0003 Codex Agent Model Routing](../../wiki/decisions/DECISION-0003-codex-agent-model-routing.md)はDECISION-0004によりsupersededであるため、現行判断の根拠としては採用しない。ただし不変の履歴として保持される。
