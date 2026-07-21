---
task_id: "TASK-0032"
title: "Agent設定をデフォルト値へ戻す"
status: draft
created_at: "2026-07-22"
---

# TASK-0032 Agent設定をデフォルト値へ戻す

## `Planning input packet`（Main Agent所有）

このsectionをPlannerとQAへ渡す唯一の`planning input packet`とし、各内容をPLAN/QA_PLANへ複製しない。

### 目的

<!-- 達成したい結果を記載する。 -->

### 対象と対象外

#### 対象

- TODO

#### 対象外

- TODO
<!-- safety_contractの場合: 製品コード、test、runtime/build設定、Schema、製品依存、生成製品入力/成果物、外部観測可能な挙動を変更しない。 -->

### 受け入れ条件

<!-- AC-IDはTask内で一意かつ安定させ、観測可能な結果をここに一度だけ記載する。 -->

- [ ] AC-1: TODO

### 安定した参照

| 参照ID | 対象 | 固定改訂/ダイジェスト | 用途 |
|---|---|---|---|
| REF-1 | TODO | TODO | TODO |

### 依存状態

| 依存 | 状態 (`ready` / `pending`) | planning参照 | `ready`後に固定する値 |
|---|---|---|---|
| なし | `ready` | N/A | N/A |

### 許可パス

- TODO

### 完了経路preflight

| 確認対象 | 結果 | コマンドまたは根拠 |
|---|---|---|
| 完了checker | TODO | TODO |
| 権限 | TODO | TODO |
| 依存状態と参照 | TODO | TODO |
| 生成物の有無と更新方法 | TODO | TODO |
| 割当ワークツリー | TODO | TODO |
| Lapログの書込・Schema・`repository annotation` | TODO | TODO |

### 未決事項

- なし

### `Dependency-ready reconciliation`

<!-- 依存ready時にMainがready参照、planning参照との差分、AC/設計/scope/QAへの影響、再承認結果を追記する。依存なし又は未readyならN/Aとする。 -->

- N/A

## 背景

<!-- なぜ今必要か、現在の問題、関連する制約を記載する。 -->

## 検討すべき設計観点

- TODO

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
