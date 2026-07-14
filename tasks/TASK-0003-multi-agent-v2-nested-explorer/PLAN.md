---
task_id: "TASK-0003"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_by: ""
approved_at: ""
approved_dev_profile: "sol-high"
approved_dev_profile_reason: "MultiAgentV2のnative nested launch、二つのrepository rootのadapter契約、sandbox初期化回避、起動証跡、失敗伝播を横断する権限境界変更であるため"
approved_dev_profile_risk_signals:
  - cross_cutting
  - sandbox_commit_ownership
  - dual_repository_root
  - configuration_contract
  - runtime_integration
planned_implementation_files: 3
planned_implementation_lines: 235
estimate_points: 2
---

# TASK-0003 PLAN

## DEV profile選定証跡

```yaml
profile: sol-high
model: gpt-5.6-sol
reasoning_effort: high
decision_rule: "高risk signal、横断性、または不確実性が一つでもあればsol-high"
reason: "native nested launchと既存のexplicit launcherを共存させ、adapter、runtime evidence、sandbox/commit境界を変更するため"
```

`luna-xhigh` は局所的・機械検証可能で、contract、sandbox、concurrency、migrationのrisk signalがない場合だけ許容される。本Taskはその条件を満たさない。DEV中に更なるrisk signalが出た場合もprofileは`sol-high`のままとし、範囲または既存Decisionの解釈を変える必要が出た場合はmain Agentへ停止・報告する。

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| PLANのnative Explorer起動 | `ACTION=plan`のpromptとMultiAgentV2 tool call fixtureで、PLANだけが固定`explorer` roleへ一問を渡し、要約・file referenceを同じ実行へ返すことを確認する。 | product `.codex/config.toml`、`agents/planner.toml`、`run-work-agent.mjs` |
| 実効contractと証跡 | native child launchの構造化traceから`gpt-5.6-luna`/`medium`/`read-only`、cwd、write scopeなし、child結果、`commit:null`をassertし、raw logや自己申告に依存しない。 | canonical contractとlaunch-evidence helper |
| Explorer境界 | 一問（trim済み・改行なし・500文字以下）、read-only、編集/Git write/scope拡大/再委譲なし、短い結果だけを固定入力検査・role設定・負例試験で確認する。 | Explorer role contract、routing validation |
| depth/thread/fan-out | root→PLAN→Explorerだけをdepth 2として許可し、Explorer→child、depth 3、3 thread、質問0・2件以上を実行前に拒否する。 | MultiAgentV2 configとdelegation test |
| canonical adapter/drift | canonical product TOMLのfeature flag、tool namespace、role registry、各role contract、depth/thread/no-childをdigestの入力にし、生成adapterと完全一致しなければFAILする。 | `agent-routing.mjs` generation/check |
| 管理sandboxでの実行 | native nested pathは別processの`codex exec`を呼ばず、childが`~/.codex` state DB又はin-process clientを初期化する書込みを要しない。管理sandboxでPLAN→Explorerの成功traceを受け入れ試験に残す。 | TASK背景、native runtime integration test |
| failureとcommit境界 | 起動前拒否、child失敗、親PLAN失敗を区別してfail closedにする。Explorer由来の差分/stage/commitは残さず、PLAN全体は子失敗を明示エラーとして終了し親rollbackの対象とする。 | existing parent validation/rollback contract |
| 既存経路との互換 | rootからの`run-explorer-agent.mjs` explicit launcherとPLAN以外のroleの契約は変更しない。native PLAN pathは限定追加であり、Decision置換の必要性はHANDOVERでWiki Agent向け候補として記録する。 | DECISION-0003、existing launcher tests |

## 関連Wikiと判断

- [Codex Agent Model Routing](../../wiki/semantic/schemas/codex-agent-model-routing.md) と [DECISION-0003](../../wiki/decisions/DECISION-0003-codex-agent-model-routing.md) の固定model/effort、read-only、一問、depth 2、threads 2、no-child、trace検証、lock所有親のcommit責務を変更しない。既存Decisionの「明示launcher」はroot direct launcherを維持して満たす。PLAN内native explicit-role起動を同じ概念に含めるかは、このPLANでDecisionを暗黙に書換えずHANDOVERの判断候補として引き渡す。
- [Task Delivery](../../wiki/semantic/scripts/task-delivery.md) と [DECISION-0001](../../wiki/decisions/DECISION-0001-plan-dev-qa-process.md) に従い、PLAN承認、実装前QA計画、独立review、main merge、マージ後QAを維持する。現時点の`QA_PLAN.md`、`QA_RESULT.md`、`REVIEW_RESULT.md`はいずれもdraft/pendingであり、承認済みQA evidenceは存在しない。
- [Work Repository Boundary](../../wiki/semantic/schemas/work-repository-boundary.md) と [DECISION-0002](../../wiki/decisions/DECISION-0002-work-repository-and-wiki-ownership.md) に従い、product側TOMLを正本、work側`.codex/config.toml`をmain-owned生成adapter、Task証跡をwork repositoryに保つ。DEVはwork adapterを直接編集・commitしない。
- [QA FAIL Attribution](../../wiki/semantic/case-patterns/qa-fail-attribution.md) に従い、既存explicit launcherのstate DB/app-server初期化失敗は環境所見として分離する。今回のPlanner実行で得たtraceは`read-only`、`write_scope:"none"`、`commit:null`を示した一方、`~/.codex/state_5.sqlite` readonlyとin-process app-server初期化失敗によりchild exit 1となった。これはnative pathの必要性を裏付けるが、DEV defectの証明ではない。

## 設計

### 選択案

`ACTION=plan`に限り、work adapterがMultiAgentV2 feature/tool namespaceとplanner→explorer registryをcanonical設定から決定的に展開する。`run-work-agent.mjs`は外部`run-explorer-agent.mjs`のshell実行を指示せず、native tool namespaceの固定`explorer` roleだけを使う明示的・機械検査可能なPLAN promptを渡す。質問は最大一回で、launcher/runtime側のrequest validationがchain=`root,planner,explorer`、threads≤2、一問を検査する。

native launchのchild resultは、raw会話ではなく、role/profile/model/effort/sandbox/cwd/write scope、質問のdigest又はlength、result status、file references、`commit:null`だけを含む短い構造化evidenceとして親PLAN実行へ戻す。child非zeroまたはevidence不完全はPLAN child failureとして親をnonzeroにし、既存lock所有親の`validateChildOutcome`とrollbackが`PLAN.md`以外の差分、stage、commitを残さない。

canonical reader/digestはrole model/effort/sandboxだけでなく、`[features.multi_agent_v2]`、`tool_namespace`、project role registry、plannerのExplorer child registry、Explorerの`max_depth=0`/`max_threads=1`と禁止policyを読み検証する。adapter rendererは同じfeature・namespace・registryを明示し、checkは一文字のdriftでも拒否する。既存のroot direct `run-explorer-agent.mjs`は明示legacy-compatible入口として残し、PLAN以外へnative delegationを拡張しない。

### 代替案と不採用理由

- PLANが既存`run-explorer-agent.mjs`を別processで呼ぶ案は、管理sandboxでstate DB/app-server初期化書込みに失敗する実測があり、受け入れ条件の同一PLAN内完了を満たせないため不採用。
- generic/natural-language delegationでroleを選ばせる案は、role、質問数、depth、fan-outを機械的に固定できないため不採用。
- adapterへmodel値だけを複写する案はMultiAgentV2 feature、namespace、registryのdriftを見逃すため不採用。
- Explorerのwrite scope、質問数、child spawnを緩和する案はTASKとDECISION-0003に反するため不採用。
- existing root launcherを削除してnative pathだけにする案は既存explicit launcher経路を回帰させ、Decisionの意味を暗黙に変えるため不採用。

### 責務と不変条件

- main AgentだけがPLAN承認、adapter sync/governance commit、merge/revert、FAILの差し戻し先、Decision更新要否を判断する。
- Plannerは`PLAN.md`だけを編集し、Explorerを使うかを最大一回判断する。Explorerは方針、承認、実装を決定しない。
- native Explorerは`explorer`固定role以外を選べず、read-only/no child/write scopeなしで、短い根拠要約とfile referenceだけを返す。
- lock所有launcher親だけがscope検査、stage、hook、commit、post-check、rollbackを行う。ExplorerとPlanner childにGit authorityを与えない。
- PLAN以外のroleのnative Explorer実行は拒否し、root direct launcherのCLI contractは後方互換のままとする。

### 失敗時・移行・互換性

起動前の質問/chain/thread/role不正はchildを起動せず、構造化evidenceに拒否reasonと`commit:null`を残す。child失敗、禁止evidence、親scope/stage/hook/validation失敗は既存rollbackで開始前状態に戻し、PLANの成功として扱わない。native runtimeが管理sandboxで利用不能ならQA_RESULTでは環境・runtime integration failureとして再現条件を分けて記録し、main Agentが差し戻し先を決める。

移行は、(1) canonical parser/adapterの拡張、(2)PLAN限定native pathとnegative tests、(3)work adapter sync、(4)managed-sandbox acceptance、の順に行う。root direct launcher、legacy fixed-route validation、PLAN以外のroleは変更しない。native pathがDECISION-0003のexplicit launcherを実質置換すると判断される場合、DEVはHANDOVERへ「root explicit launcherを保持しつつPLAN native explicit-role pathをDecisionへ追記するか」の候補と根拠を記録する。Wiki本文・DecisionはこのTaskでは変更しない。

## 変更予定

見積もり対象はimplementation/configurationだけとする。test/documentationは見積もり対象外である。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `../agent-harness/.codex/agents/planner.toml` | configuration | 15 | PLAN限定のnative Explorer明示role、最大一問、no generic delegationを固定prompt/registryで明確化する。 |
| `../agent-harness/scripts/task/agent-routing.mjs` | implementation | 150 | MultiAgentV2 topologyのcanonical parse/digest、adapter render/check、PLAN-native delegation/evidence validationを追加する。 |
| `../agent-harness/scripts/task/run-work-agent.mjs` | implementation | 70 | `ACTION=plan`だけにnative fixed Explorer toolの指示とfailure propagationを追加し、external launcher指示を除く。 |
| `../agent-harness/scripts/task/agent-routing.test.mjs` | test | - | feature/namespace/registry drift、PLAN chain、question/fan-out/no-child、structured child evidenceを検査する。 |
| `../agent-harness/scripts/task/development-process.test.mjs` | test | - | PLAN限定性、親rollback、existing root launcher非回帰、adapter sync契約を検査する。 |
| `../agent-harness/docs/development/agent-roles.md` | documentation | - | PLAN native Explorerの利用条件、結果形式、root launcherとの互換境界を記載する。 |

implementation/configurationは3ファイル、235行。`file_score = ceil(3 / 3) = 1`、`line_score = ceil(235 / 200) = 2`。許容scale `1, 2, 3, 5, 8, 13` のうち2以上の最小値は2のため、`estimate_points: 2`とする。

## 実装手順

1. canonical product TOMLからMultiAgentV2 feature、namespace、role registry、planner child registry、Explorer no-child policyを読み、欠落・未知・不一致をfail closedにする。
2. 同じcanonical入力からwork adapterを生成し、feature/namespace/registryを含むdigest完全一致checkを追加する。adapter更新は既存のmain-owned sync launcherだけで行う。
3. `ACTION=plan`のpromptをnative fixed Explorer role用に限定し、質問数・chain・result format・failure propagationを明記する。外部Explorer launcherの実行を指示しない。
4. structured native-child evidenceを検査し、失敗を親failureへ伝播する。既存scope/stage/commit/rollback実装を再利用し、Explorerが差分を作れないことをassertする。
5. unit/integration fixtureで正常、質問0/2、長過ぎ/改行、depth 3、3 threads、Explorer child、write/Git、child failure、adapter drift、root launcher回帰を検証する。
6. docs、`make check`、`make -C ../agent-harness work-check`、shared hookを実行する。main Agentはadapter syncを別責務として行い、ReviewerとQAはmerge後のmanaged sandboxで再確認する。

## 検証計画

- `node --test scripts/task/agent-routing.test.mjs`でcanonical MultiAgentV2 topology、deterministic adapter/digest、PLAN→Explorer許可、root direct launcher保持、モデル/effort/read-only/no-child、質問・depth・threadの正負例を確認する。
- `node --test scripts/task/development-process.test.mjs`で`ACTION=plan`限定性、child evidence不備/失敗の親伝播、Explorer由来のstage/commitなし、scope/hook/validation failure rollbackを確認する。
- fixtureのnative runtime bridgeで、別`codex exec`をspawnせずmanaged read-only環境でもPLAN→Explorerが短いsummary/file referencesを返し、traceがLuna/medium/read-only/write scope none/`commit:null`であることを確認する。
- 実環境のacceptanceでは`make work-agent TASK=TASK-0003 ACTION=plan`相当をmain管理sandboxで一度実行し、成功、起動前拒否、child failureを区別したredacted traceをQA_RESULTへ残す。state DB/app-serverに起因する未実施は環境所見として分類しPASSへ置換しない。
- product `make check`、work `make -C ../agent-harness work-check`、adapter sync/check、独立review、main merge後QAを既存gate順に実行する。

## 未解決事項

- なし。DECISION-0003の文言へnative explicit-role pathを追加するかは、実装完了後にHANDOVERからWiki Agentへ渡す判断候補であり、DEV開始を妨げる未解決設計ではない。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV profile `sol-high`、選定理由、risk signalsを承認した。
- [ ] DEV開始を承認した。
