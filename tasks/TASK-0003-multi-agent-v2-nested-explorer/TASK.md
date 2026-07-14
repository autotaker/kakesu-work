---
task_id: "TASK-0003"
title: "MultiAgentV2のPLANからExplorerを起動可能にする"
status: draft
created_at: "2026-07-14"
---

# TASK-0003 MultiAgentV2のPLANからExplorerを起動可能にする

## 目的

`work-agent` から起動した PLAN Agent が、MultiAgentV2 の明示的な Explorer role を使って一件の限定的なリポジトリ調査を完了できるようにする。Explorer の Luna / `medium` / read-only / no-child 契約と、ロックを保持する launcher 親の scope検査・commit 権限を維持しながら、管理環境でも調査結果を PLAN の根拠に利用できる状態にする。

## 背景

TASK-0002 では、root -> role Agent -> Explorer の深さ2の調査、最大2 thread、一件の bounded question、Explorer の `gpt-5.6-luna` / `medium` / read-only / no-child を正規契約とした。製品リポジトリの `.codex/config.toml` は MultiAgentV2 の tool namespace と Explorer role を定義し、Planner role 定義も Explorer の子起動を許可している。

一方、運用リポジトリの生成 adapter には MultiAgentV2 の feature 契約が含まれていない。さらに現行の `run-work-agent.mjs` は、PLAN Agent への prompt で natural-language / custom-agent delegation を禁止し、別 process の `run-explorer-agent.mjs` を実行するよう指示している。この実行経路は、子の `codex exec` が `~/.codex` の state DB や app-server 初期化を必要とする環境では read-only 制約により起動できず、PLAN から Explorer を使えない。

そのため、調査の権限境界は緩和せず、PLAN の実行経路と生成 adapter を MultiAgentV2 の正規設定に整合させる必要がある。

## スコープ

### 対象

- `ACTION=plan` で起動される PLAN Agent から MultiAgentV2 Explorer を明示的に子起動する実行契約
- 製品リポジトリの正規 MultiAgentV2 設定と、運用リポジトリの決定的な生成 adapter・digest・drift検査の整合
- PLAN の調査指示、Explorer の bounded question・read-only・no-child 制約、簡潔な結果受け渡し
- MultiAgentV2 の正常系と、複数質問、深さ超過、fan-out、write / Git操作、再委譲を拒否する異常系の自動テスト
- 管理された sandbox で PLAN -> Explorer の一連の実行が完了することの受け入れ検証
- Explorer の利用方法、制約、起動証跡を説明する必要最小限の製品文書

### 対象外

- PLAN 以外のmain、DEV、QA、REVIEW Agent の Explorer 実行経路の一括変更
- Explorer のmodel、reasoning effort、read-only sandbox、一質問制約、no-child 制約の緩和
- Explorer による編集、stage、commit、scope拡大、実装方針や承認の決定
- ロール最大深さ2または最大2 threadを超える並列化と fan-out
- 利用者の global Codex 設定、`~/.codex` の書き込み権限、未文書化の model alias への依存
- PLAN / DEV / QA ゲート、運用リポジトリの共通ロック、launcher 親の scope検査・stage・hook・commit・事後検査の責務変更
- 製品機能、ドメインモデル、Wiki Schema の変更

## 受け入れ条件

- [ ] `work-agent` の `ACTION=plan` で起動した PLAN Agent が、MultiAgentV2 の Explorer role へ一件の bounded question を渡し、同じ PLAN 実行中に簡潔な根拠要約とファイル参照を受け取れる。
- [ ] PLAN -> Explorer の実効契約が `gpt-5.6-luna`、reasoning effort `medium`、read-only sandbox、write scopeなし、commitなしであることを、Agent の自己申告ではなく構造化された起動証跡または同等の runtime 証跡で検査できる。
- [ ] Explorer は調査対象 root の読み取りだけを行い、ファイル編集、Git 書き込み、scope拡大、raw log転送、子Agent起動を行えない。
- [ ] root -> PLAN -> Explorer を深さ2として許可し、Explorer -> child、深さ3、3 thread以上、0件または2件以上の質問を起動前に拒否できる。
- [ ] 製品リポジトリの MultiAgentV2 正規設定が運用リポジトリの生成 adapter へ決定的に反映され、feature、tool namespace、role、model、effort、sandbox、depth、thread、no-child の drift を検出できる。
- [ ] PLAN からの Explorer 起動は、子 Explorer が `~/.codex` へ書き込めることや、別の in-process app-server client を初期化できることを前提とせず、運用リポジトリの管理環境で実調査を完了できる。
- [ ] Explorer を起動した場合の成功、起動前拒否、子実行失敗を区別でき、失敗時に Explorer 由来の差分、stage、commitを残さない。PLAN 全体への失敗伝播方針は PLAN で明示する。
- [ ] PLAN Agent の `gpt-5.6-terra` / `medium`、`PLAN.md` のみの write scope、launcher 親のロック・scope検査・stage・hook・commit・rollback責務が維持される。
- [ ] PLAN 以外のロールの Explorer 契約と、root からの既存の明示 Explorer launcher 経路に回帰がない。互換範囲と移行方針は PLAN で明示する。
- [ ] 正常系・異常系の自動テスト、管理環境での PLAN -> Explorer 受け入れ証跡、既存検査と `make check` が PASS する。

## 検討すべき設計観点

- MultiAgentV2 の feature、tool namespace、role registry をどこで正本化し、生成 adapter・digest・drift検査で完全一致をどう証明するか
- `ACTION=plan` だけに Explorer の nested launch を許可し、generic delegation、他role起動、natural-language による不定なrole選択と区別する方法
- root = 0、PLAN = 1、Explorer = 2 の深さと thread上限を、MultiAgentV2 の実際の runtime semantics と launcher 契約の両方で検査する方法
- Explorer の一質問制約、質問長、調査 root、返却形式を、prompt だけでなく起動前検査とテストで保証する方法
- child の write、Git操作、scope拡大、再委譲を、sandbox、role設定、tool公開範囲、固定prompt、事後検査のどの層で防ぐか
- MultiAgentV2 の nested launch 証跡から model、effort、cwd、sandbox、write scope、child result、commit不在を raw log なしで検証する方法
- Explorer の必要性を PLAN Agent が判断して最大一回だけ使う方法と、成功、起動前拒否、子実行失敗を過度なraw logなしで観測・伝播する方法
- 既存の `run-explorer-agent.mjs` と PLAN の MultiAgentV2 経路の責務分担、後方互換性、移行順序、重複した契約の drift 防止
- DECISION-0003 が定めた「明示 launcher」と、PLAN 内の MultiAgentV2 明示role起動の整合性。実装経路の置換が判断の変更に当たる場合は、既存 Decision を暗黙に上書きせず、置換判断の候補を HANDOVER で Wiki Agent へ引き渡すこと
- プロセス初期化の環境制約と製品不具合を区別し、独立QAの FAIL を自動的に DEV へ帰責しない検証証跡の残し方

## 完成の定義

- [ ] 受け入れ条件を満たしている。
- [ ] 必要な実装、テスト、文書が完成している。
- [ ] 独立レビューと`make check`がPASSしている。
- [ ] マージ後QAが完了している。
- [ ] HANDOVERがWiki Agentに取り込みされている。

## 関連コンテキスト

### 意味 Wiki

- [Codex Agent Model Routing](../../wiki/semantic/schemas/codex-agent-model-routing.md): Explorer の固定model、read-only、一質問、深さ・thread上限、no-child、runtime trace による検証を本Taskの非交渉条件とする。
- [Development Task](../../wiki/semantic/concepts/development-task.md): 本Taskを、目的、受け入れ条件、証跡を持つ運用上の開発単位として扱う。
- [Task Delivery](../../wiki/semantic/scripts/task-delivery.md): PLAN承認、実装前QA計画、独立レビュー、mainへのマージ、マージ後QAの進行順を維持する。
- [Work Repository Boundary](../../wiki/semantic/schemas/work-repository-boundary.md): 製品側の正規設定と運用側の生成 adapter、共通ロック、Agent所有範囲を維持する。
- [QA FAIL Attribution](../../wiki/semantic/case-patterns/qa-fail-attribution.md): sandbox、state DB、app-server 初期化による失敗を環境・要件・実装・QA計画に分類し、自動的に DEV 不具合としない。

### 判断

- [DECISION-0001 PLAN DEV QA Process](../../wiki/decisions/DECISION-0001-plan-dev-qa-process.md): PLAN Agent に調査手段を追加しても、main Agentの承認、DEV・Reviewer・QAの独立性、マージ判断を委譲しない。
- [DECISION-0002 Work Repository and Wiki Ownership](../../wiki/decisions/DECISION-0002-work-repository-and-wiki-ownership.md): 運用証跡の正本、main一本運用、共通ロック、Wiki所有権を変更しない。
- [DECISION-0003 Codex Agent Model Routing](../../wiki/decisions/DECISION-0003-codex-agent-model-routing.md): Explorer の固定routing、bounded read-only 調査、深さ2、2 thread、no-child、trace検証を維持する。同 Decision が明記する明示 launcher と MultiAgentV2 経路の整合性は PLAN で明示的に判断する。

### 適用しなかった重要な判断

- なし。DECISION-0001〜0003の権限・進行・routing境界をすべて適用する。DECISION-0003 の明示 launcher を MultiAgentV2 の明示role起動で代替できるかは未決の設計論点であり、判断を不適用とみなして先行しない。
