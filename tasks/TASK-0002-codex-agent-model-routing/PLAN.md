---
task_id: "TASK-0002"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_by: ""
approved_at: ""
approved_dev_profile: "sol-high"
approved_dev_profile_reason: "launcher、sandbox/commit所有権、兄弟2 repository root、project設定、起動証跡、テストを横断し、権限境界の契約変更を伴うため"
approved_dev_profile_risk_signals:
  - cross_cutting
  - sandbox_commit_ownership
  - dual_repository_root
  - configuration_contract
  - process_evidence
planned_implementation_files: 7
planned_implementation_lines: 760
estimate_points: 5
---

# TASK-0002 PLAN

## DEV profile選定証跡

```yaml
profile: sol-high
model: gpt-5.6-sol
reasoning_effort: high
decision_rule: "高リスクsignalが一つでもある、横断的、または判定不能ならsol-high"
reason: "launcher、sandbox/commit所有権、兄弟2 root、設定、証跡、テストを横断する契約・権限境界変更"
```

`luna-xhigh` は、対象が局所的で、機械的に検証可能で、契約・Schema・security・concurrency・migrationのいずれにも影響せず、曖昧さがない場合だけ選択する。上記のいずれかのsignal、横断性、曖昧さがあれば決定的に `sol-high` とする。DEV中に新しいsignalまたは不確実性を発見した場合、DEVは停止してPLAN証跡へ変更前profile、`sol-high`、signal、理由、main承認者・時刻を追記し、承認後に再起動する。降格は行わない。

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| 固定役割routing | routing単体試験でmain=`gpt-5.6-sol/high`、PLAN/QA/REVIEW=`gpt-5.6-terra/medium`、explorer=`gpt-5.6-luna/medium`を完全一致で確認する。 | canonical routing manifest |
| DEV routingとPLAN証跡 | `check-task` が承認済みPLANの `approved_dev_profile`、reason、risk signalsを検査し、`luna-xhigh`/`sol-high`以外、reason欠落、rule違反をFAILする。TASK-0002は上記 `sol-high` を使用する。 | PLAN frontmatterと選定証跡 |
| nestingとexplorer | role Agentだけが深さ2のexplorerを一質問で起動でき、root直下explorer、深さ3、explorer起点、複数質問、child spawnを起動前に拒否する。 | launcher policyと試験 |
| explorer権限 | explorer起動はread-only sandbox、write scopeなし、commit/spawn/scope拡大禁止の固定promptであり、出力は要約とファイル参照だけである。 | dry-run/evidence試験 |
| 両rootの設定 | `agent-harness` と `agent-harness-work` をcwdにした設定解決試験が同一canonical manifestと役割定義を返す。グローバル設定・未文書aliasは参照しない。 | project-local config fixture |
| parent-owned commit | 子stdinはclosed、子の成功後のみ、ロック保持中の親がdirty scopeを検査、stage、hook付きcommit、再検査する。子には`.git`書込みを与えない。 | process integration試験 |
| fail-closed互換性 | fixed roleへの`PROFILE`/`MODEL`/effort上書きはcanonical値と一致しなければ起動前FAILする。一致値のみ許容する。文書化したWiki/non-role legacy overrideだけを維持し、その正常・異常系を試験する。 | argument validation試験 |
| concise launch evidence | dry-runと実行でrole/profile、実model、effort、cwd、sandbox/write scope、allowed paths、child exit、親commit（なしも含む）を一行JSONで出力し、秘密値・raw logは保存しない。 | launch evidence試験 |

## 関連Wikiと判断

- Development Task と Task Delivery に従い、PLANは実装しない。PLAN承認、実装前QA計画、独立review、mainのみの`--no-ff` mergeは変更しない。
- Work Repository Boundary と DECISION-0002 に従い、製品コード・ランチャーは`agent-harness`、Task証跡は`agent-harness-work`に残す。共通lockは後者への書込み全期間を直列化する。
- DECISION-0001 の役割分離を固定routingとexplorerの限定委譲で強化する。QA FAILは QA FAIL Attribution に従い、自動的にDEV責任としない。
- Wiki Ingestion は完了後だけ適用する。本変更でWiki本文・Schemaは変更しない。

## 設計

### 選択案

`agent-harness/scripts/task/agent-routing.mjs` をrouting、role/depth、DEV profile、override検査、evidence payloadの唯一の正本にする。role launcherはこのmanifestを読んでCodex引数・prompt・sandboxを組み立てる。両repositoryのproject-local `.codex/config.toml` は同じcanonical manifestの所在を明示してrootをtrustする最小adapterとし、roleごとのモデル値を別管理しない。設定解決がmanifest外のprofile/aliasへfallbackしたらFAILする。

root(0) はmain orchestrationだけ、role Agent(1) はmain/plan/dev/qa/review、explorer(2) は一つのbounded read-only調査だけにする。explorerはrole Agentのみから起動可能で、固定のread-only prompt、write scopeなし、`spawn=false`、`commit=false`をmanifestと起動引数の両方で指定する。委譲は既定off、同時数は1、会話全文を証跡化しない。

`run-work-agent.mjs` は子Agentを`.git` write権限なしのsandboxで実行しstdinを`ignore`へ閉じる。成功後も子はcommitしない。親はlockを保持したまま、許可パス以外のdirty/untracked/staged変更を拒否し、許可変更のみstageし、既存pre-commit hook、`validate-work`、commit、commit後`validate-work`を順に実行する。scope拒否・hook失敗・検証失敗ではcommitせず、JSON evidenceにfailureと`commit:null`を残す。既存lock、hookの許可パス検査、製品/work repository境界は維持する。

固定roleの上書きはfail closedとする。`--profile`、`--model`、effort指定がroleのcanonical値と不一致なら子起動前にエラーにする。一致する明示値は冗長だが許可する。roleを持たない従来の汎用起動とWiki launcherの既存`PROFILE`/`MODEL` overrideは、文書化済みのlegacy経路に限定して保持し、role固定を経由しないことを明示・試験する。

### 代替案と不採用理由

- 子Agentへ`.git` writeを許しhookだけで制限する案は、commit権限とstage内容を子へ委譲し、sandbox境界を弱めるため不採用。
- 各launcher・各rootへmodel値を複写する案はdriftを検出しにくいため不採用。
- DEVを常にLunaへroutingする案は、本Taskのような横断的な権限・設定契約を過小評価するため不採用。
- rootからexplorerを直接起動、またはexplorerの再委譲を許す案は深さ・scope・token消費を統制できないため不採用。
- 全既存overrideを黙って維持する案は固定roleの非交渉条件を破るため不採用。

### 責務と境界

- main Agent: PLAN承認、DEV profile/promotion承認、branch/worktree、merge/revert、FAIL分類のみを判断する。
- Planner: 本PLANと機械検査可能なDEV選定証跡を所有する。承認欄はmainのみが更新する。
- DEV: 承認profileで製品worktreeを編集・検証する。profile昇格を自己承認しない。
- launcher parent: lock、scope検査、stage/hook/commit、構造化evidenceを所有する。sandbox childにはcommit authorityを渡さない。
- role child: allowed filesを編集・検証するだけで、commit/stage/lock解放を行わない。
- explorer: bounded read-only investigationと短い根拠要約だけを行う。
- Wiki: legacy overrideの文書化対象になり得るが、本文・SchemaはこのTaskでは変更しない。

### 不変条件

- work repositoryは`main`、clean開始、共通lock保持、action所有ファイルだけの一コミットである。
- product repositoryのTask branch/worktreeとwork repositoryの証跡正本を混同しない。
- parent commitは子exit=0、scope検査、hook、検証すべて成功時だけ発生する。
- role、model、effort、深さ、explorer権限はcaller引数で緩和できない。
- approved DEV profileとreasonはPLAN証跡に存在し、実行profileと一致する。

### 失敗時・移行・互換性

導入時は既存`work-agent`をrole-aware parent flowへ置換し、Wiki/non-role legacy launcherは明文化されたoverride検査を追加して維持する。fixed-roleの不一致override、未知DEV profile、PLAN証跡欠落、深さ違反、scope外差分、child nonzero、hook/validation失敗はfail closedにする。部分stage、commit、lock release後の再試行は行わない。rollbackはこのTaskの製品・work設定・文書を同一revertで戻し、旧launcherを復元する。既存commit履歴・Task証跡・Wikiを移行しない。

## 変更予定

見積もり対象は実装コード、Schema、設定ファイルだけとする。テストと文書は見積もり対象外である。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `../agent-harness/.codex/config.toml` | configuration | 25 | product rootのproject-local routing adapterとroot policy。 |
| `.codex/config.toml` | configuration | 25 | work rootから同一canonical routingを解決するadapter。 |
| `../agent-harness/scripts/task/agent-routing.mjs` | implementation | 220 | canonical profiles、role/depth/override/PLAN evidence検査、evidence serializer。 |
| `../agent-harness/scripts/task/run-work-agent.mjs` | implementation | 210 | closed stdin、child sandbox、parent scope/stage/hook/commit/evidence flow。 |
| `../agent-harness/scripts/task/run-wiki-agent.mjs` | implementation | 70 | documented legacy non-role/Wiki override validationとevidence統合。 |
| `../agent-harness/scripts/task/check-task.mjs` | implementation | 110 | approved DEV profile/reason/promotion履歴のmachine validation。 |
| `../agent-harness/scripts/task/work-pre-commit.mjs` | implementation | 100 | parent-owned staging前提とallowed-path rejectionを保つhook検査。 |
| `../agent-harness/scripts/task/agent-routing.test.mjs` | test | - | routing、effort、override、depth、explorer、dual-root、evidenceの単体試験。 |
| `../agent-harness/scripts/task/development-process.test.mjs` | test | - | approved profileの正常/invalid値、promotion、parent commit、scope拒否、closed stdinの統合試験。 |
| `../agent-harness/Makefile` | build integration | - | routing testを`test-process`へ登録し、role launcher targetsを追加。 |
| `../agent-harness/docs/development/agent-roles.md` | documentation | - | profile表、explorer契約、override互換境界を記載。 |
| `../agent-harness/docs/development/development-process.md` | documentation | - | PLAN profile承認・promotion・parent commit手順を記載。 |
| `../agent-harness/AGENTS.md` | documentation | - | direct Agent commit禁止とlauncher-owned commitの運用規約を更新。 |

実装対象は7ファイル、760行。`ceil(7/3)=3`、`ceil(760/200)=4`、最小scale値は5のため、`estimate_points: 5` とする。

## 実装手順

1. canonical routing manifestと両root adapterを追加し、固定profile、DEV rule、depth、explorer contract、evidence schemaを定義する。
2. `check-task`にPLANのprofile/reason/risk/promotion検査を追加し、approved PLANからlauncherへのprofile受渡しを照合する。
3. role launcherをparent-owned commit flowへ変更する。child stdinを閉じ、child成功後にのみ親がscope検査、stage、hook、commit、post-checkを行う。
4. Wiki/non-role legacy経路を明示的に分離し、fixed-role override不一致を起動前FAILにする。
5. routing/unitとprocess/integration試験、Makefile、運用文書を更新する。
6. product `make check`、work `make -C ../agent-harness work-check`、hook付きTaskコミット、独立reviewを実施する。

## 検証計画

- `node --test scripts/task/agent-routing.test.mjs`：全role/DEV profileのmodel・effort、未知値、不一致override、dual-root、depth 0–2、explorer read-only/no-fan-out、一質問制約、evidence字段を確認する。
- `node --test scripts/task/development-process.test.mjs`：PLANの`sol-high`/`luna-xhigh`正常系、reason/risk欠落、未知profile、Luna rule違反、promotion履歴、scope外変更、child stdin closed、child failure、parent-only commit、hook failureを確認する。
- launcher fixtureでchildがcommitできず、親が許可ファイルだけをcommitし、commit hashをJSON evidenceへ載せることを確認する。commitなし・拒否時は`commit:null`であることを確認する。
- productで`make check`、work repositoryで`make -C ../agent-harness work-check`と対象Task検査を実行し、既存TASK-0001のgateを回帰させない。
- Reviewerは実際のdiff、PLAN/QA計画、evidenceの秘密値非含有、hookを迂回しないcommit経路を独立確認する。QAはmerge済みmainで上記受け入れケースを再実行する。

## 未解決事項

- Codex CLIがproject-local configからcanonical manifestを参照する正式な設定key/引数は、DEVが利用中CLIの`--help`と既存ローカル設定で確認して確定する。未文書のinclude/aliasに依存しない。確認できない場合は、root adapterを薄いlauncher設定にし、manifestをNode launcherで明示的に読んでfail closedにする。
- work rootの`.codex/config.toml`は運用リポジトリに属する。DEV開始前にmain Agentが、TASKの実装範囲としてその一ファイルをparent-owned work flowで変更する権限を確認する。許可されない場合は、work root adapterの代替を承認するまでDEVを開始しない。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV profile `sol-high`、選定理由、promotion規則を承認した。
- [ ] DEV開始を承認した。
