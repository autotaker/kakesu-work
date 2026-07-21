---
task_id: "TASK-0032"
change_class: product
status: approved
planner_agent: "planner-agent-terra-medium"
approved_dev_profile: "sol-high"
approved_dev_profile_reason: "project-scoped設定の削除がcanonical parser、fallback delegation制限、決定的adapter、drift検査、role契約文書を横断し、既定値への移行後もfail-closed境界を保つ必要があるため"
approved_dev_profile_risk_signals:
  - configuration_contract
  - cross_cutting
  - fallback_routing
approved_by: "main-agent-sol-high"
approved_at: "2026-07-22T08:09:25+10:00"
planning_reviewed_by: ""
planning_review_decision: "pending"
planning_reviewed_at: ""
classification_approved_by: ""
classification_approved_at: ""
classification_approval_reason: ""
planned_implementation_files: 2
planned_implementation_lines: 75
estimate_points: 1
---

# TASK-0032 PLAN

## 分類と境界

これは製品のproject-scoped設定、routing実装、生成adapter、テスト、および運用文書を変更する`product` Taskである。旧MultiAgentV2の明示有効化とproject-level registryを削除して、Codexの組み込み既定`agents.max_threads=6`、`agents.max_depth=1`へ委ねる。roleごとの`.codex/agents/*.toml`、そのrole名/model/effort/sandbox、top-level `model`・`model_reasoning_effort`・`sandbox_mode`、native spawnのrole分離、Main所有のGit境界、fallback launcherの固定model/effortは変更しない。

運用repositoryの`.codex/config.toml`は製品候補に含めず、製品`main`へのmerge後にMainだけが`make work-config-sync`で生成・commitする。Taskの許可外であるユーザー/managed設定、Codex runtime本体、既存Wiki/Decision、role TOMLの文言やlocal child policyには手を加えない。

## AC対応

TASKの条件本文を再掲せず、`planning input packet`のAC-IDに設計を対応させる。

| AC-ID | 設計判断 | 変更パス | 実施順序 | 失敗時の扱い |
|---|---|---|---|---|
| AC-1 | canonical project configは固定top-level 3 keyだけを保持し、`[features.multi_agent_v2]`、その2 key、および`[agents]`と全child tableを削除する。parserと正負fixtureはこれらの再導入を構造driftとして検出する。 | `.codex/config.toml`、`scripts/task/agent-routing.mjs`、`scripts/task/agent-routing.test.mjs` | 1–3 | prohibited section/keyが残る又は再導入される場合はcanonical parse/digest/render/checkを同期前にfail-closedし、設定を生成しない。 |
| AC-2 | project overrideを参照するrouting上の`2/2`前提を撤去し、fallback delegation validatorは未指定時のCodex既定値（depth 1/thread 6）を検査する。standalone role TOMLのrole-local制約は対象外として維持する。 | `scripts/task/agent-routing.mjs`、`scripts/task/agent-routing.test.mjs`、`docs/development/agent-roles.md` | 2–4 | depth 2のnested Explorer又は7 threadを許す、あるいは既定を別のproject値で代替するfixtureは失敗とする。既定値・runtime解釈に矛盾が出ればDEVを止めて`requirement_gap`としてMainへ戻す。 |
| AC-3 | deterministic adapterはcanonical contract digestと固定top-level設定だけを出力し、feature table、project `[agents]`、role registryを出力しない。CHECKは完全一致driftを引き続き拒否する。 | `scripts/task/agent-routing.mjs`、`scripts/task/agent-routing.test.mjs`、merge後の運用側`.codex/config.toml` | 2–3、merge後 | render結果に禁止table/keyがあればunit testを失敗させる。`CHECK=1` drift、sync/hook/post-check失敗は既存rollback/`commit:null`契約に従い次gateへ進めない。 |
| AC-4 | canonical sourceはstandalone role TOMLと`ROLE_CONTRACTS`による固定role contractのままとし、project registryなしでmodel/effort、Explorer no-child、fallback route/override拒否を検査する。文書は「project configとrole TOMLの二重正本」という旧説明を改める。 | `scripts/task/agent-routing.mjs`、`scripts/task/agent-routing.test.mjs`、`docs/development/agent-roles.md`、必要時の`docs/glossary.yml`/`docs/99-glossary-index.md` | 2、4–5 | role TOML欠落・fixed model/effort不一致・Explorer境界緩和・override受入れは既存の専用routing errorで失敗させる。文書変更がrole名/model/effort/sandboxやGit責務を変える差分になればPLANへ差し戻す。 |
| AC-5 | 正常・再導入・adapter drift・fallback既定値の正負fixtureを更新し、process suiteと全体checkで回帰を検出する。 | `scripts/task/agent-routing.test.mjs`、既存`make`検査 | 3、5–6 | focused test又は`make check`のFAILは原因を`implementation_defect`、`requirement_gap`、又は`environment_issue`として分類し、削除した設定を暫定復活させずMainが差し戻し先を決める。 |

## 設計判断と関連Wiki

- REF-3が示す既定値をroutingの唯一のglobal delegation前提にする。`validateDelegation`はproject TOMLを読んで`2/2`を強制しない。root→Explorerはdepth 1として許可し、root→role→Explorerはdepth 2として拒否し、thread 6までを許可、7を拒否する。これはproject overrideの除去に伴う意図的なfallback挙動の変更である。
- role TOMLはcustom roleの正本として残る。通常roleの`[agents]`内`max_threads=2`、Explorerの`max_threads=1`/`max_depth=0`、Explorer registry、fixed promptは本Taskの対象外であり、`readCanonicalContracts`による検査を維持する。ただしproject registryをrole contractの入力・adapter出力として復活させない。
- `readCanonicalContracts`をadapterの共通入口に保ち、project configの禁止feature/table/keyを明示的に拒否する。これによりcanonical digest、adapter render、sync、`CHECK=1`が同じ不変条件で同期前に停止する。
- DECISION-0004の`agent_type`選択、cross-roleの`fork_turns="none"`、observed model/effort不一致時の停止、親のlock/stage/commit所有を変更しない。設定TOMLのsandbox宣言を実効権限の根拠にしないという判断も維持する。

### 代替案と不採用理由

- `.codex/config.toml`だけを削りrouting scriptの`2/2`とadapter registryを残す案は、実質的なproject overrideをコード生成へ隠し、AC-2/AC-3を満たさないため不採用。
- adapterにrole registryだけを残す案は、運用側生成物へproject-level `[agents]` tableを再導入するため不採用。
- role TOMLも一括削除・簡略化する案は、custom role contractとExplorerの独立no-child境界まで変更し、Taskの対象外であるため不採用。
- native runtimeをこの変更で設定し直す又はuser/managed configを更新する案は、project scopeと環境権限を越えるため不採用。実効化確認はmerge後の環境依存ケースとして分離する。

## 変更予定と見積もり

見積もり対象は実装コードと設定ファイルだけとする。テスト、文書、生成文書、merge後の運用adapterは除外する。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `.codex/config.toml` | configuration | 35 | MultiAgentV2 feature blockとproject `[agents]` registry全体を削除し、top-level固定3 keyを維持する。 |
| `scripts/task/agent-routing.mjs` | implementation | 40 | project overrideのparse/renderを除去し、禁止section/keyのfail-closed検査、built-in defaultに合わせたfallback delegation制限、registryなしadapter renderを実装する。 |
| `scripts/task/agent-routing.test.mjs` | test | - | config/adapter非出力、禁止設定再導入、既定depth/thread、role contractとadapter driftの正負fixtureを更新する。 |
| `docs/development/agent-roles.md` | documentation | - | standalone role TOML、Codex既定のglobal上限、nested Explorerの非許可、adapterの責務を現行contractへ最小同期する。 |
| `docs/glossary.yml` | generated documentation | - | 文書の用語sourceに差分が出た場合の決定的な更新。 |
| `docs/99-glossary-index.md` | generated documentation | - | 用語generatorが索引差分を出した場合だけ生成更新する。 |
| `../agent-harness-work/.codex/config.toml` | post-merge generated artifact | - | 製品merge後、Main所有`make work-config-sync`だけで生成し、本candidateのDEV変更に含めない。 |

```text
file_score = ceil(2 / 3) = 1
line_score = ceil(75 / 200) = 1
estimate_points = 1
```

## 実装順序

1. canonical configを最小top-level構成へ縮約する設計を、role TOMLを変更せずに確定する。
2. `readCanonicalContracts`、adapter render、delegation validatorを更新する。禁止project overrideはcanonical parseで拒否し、adapterはfixed top-level値とdigestだけを出力する。
3. routing testを、禁止設定の再導入がparse/digest/render/sync CHECKの全入口でfail-closedになること、adapter完全一致drift、role固定contract、root→Explorer/深度2/6・7 threadの境界へ更新する。
4. `agent-roles.md`を、global defaultとstandalone role contractを区別する記述へ最小更新する。用語generatorによる必要な生成差分だけを含める。
5. focused routing/process/doc検査から`make check`まで実行し、candidate-bound証跡へコマンド、環境、cache、exit、digest、未実施理由を残す。
6. REVIEWとQAが同一candidateを独立評価し、Mainが`merge_tree == candidate_tree`を確認してno-ff mergeする。続いてMainが`make work-config-sync`、`CHECK=1`、運用repositoryの検査を実行し、adapterと実効化タイミングの環境依存確認を記録する。

## 失敗時・移行・互換性

- 削除対象のfeature/table/key又は`[agents.*]`がcanonical/adapterに残る・再導入される場合、同期前にrouting errorで停止する。DEVはwork adapterを手編集せず、正本とfixtureを修正して再検証する。
- 設定なしの実行環境でCodexがREF-3のdefaultを観測できない、又はrestart/reload手順が安全に確定できない場合は、`live-e2e`をblockedとして記録する。focused unit PASSや別環境の結果で代替しない。
- project defaultへの移行によりnested Explorerが利用できなくなるため、既存role promptの「Explorerへ一問」は権限付与ではなくruntime上限の範囲でのみ有効とする。role prompt変更が必要と判明した場合は許可path外のためMainがscopeを再評価し、PLAN/QA_PLANを再承認する。
- adapter同期、shared hook、lock、commit又はpost-checkの失敗は、専用launcherの開始前状態へのrollbackと`commit:null`を維持する。権限/lock競合は自動的にDEV不具合へ帰責せず`environment_issue`候補としてMainへ報告する。
- 受け入れ条件、設定/adapter出力、fallback contract、生成文書以外の差分、またはrole model/effort/sandboxの変更は計画外である。後付けallowlistにせず、Task/PLAN/QA_PLANの再承認又は別Taskへ分離する。

## 検証計画

- `node --test scripts/task/agent-routing.test.mjs`：canonical configとgenerated adapterにfeature/project `[agents]`/registryがないこと、再導入をparse/digest/render/`syncWorkAdapter(..., check: true)`前に拒否すること、固定role model/effortとExplorer no-childが維持されることを確認する。
- 同fixtureで、defaultのroot→Explorerとthread 6を許可し、nested depth 2とthread 7を拒否する。fallback override拒否、one bounded question、read-only/closed-stdin launcher traceも既存どおり確認する。
- `make work-config-sync CHECK=1`：現行生成物とのdriftを検出する能力を確認する。製品候補のadapter出力検査はtemp fixtureで行い、運用側の実同期はmerge後だけにする。
- `make test-process`、`make lint-docs`、`git diff --check`：parent-owned sync/rollback回帰、用語・索引freshness、文書と空白を確認する。generatorが差分を必要とする場合にのみ生成文書を更新して再実行する。
- `make check`、`make task-check TASK=TASK-0032`：製品変更の全build/test/lintとTask evidence schemaを確認する。
- merge後にMainが`make work-config-sync`、`make work-config-sync CHECK=1`、`make work-check`を実行する。承認済み環境でCodexのrestart/reload後にproject configなしの既定値とcustom role loadingを確認できる場合は、その観測とcleanupをTaskの環境依存ケースとして記録する。

## 未解決事項

- なし。runtime reload/restartの実行可否はcandidate完成後に環境依存ケースとして判定する。

## main Agentレビュー

- [x] TASKの全AC-IDへ設計判断、パス、順序、失敗時の扱いを対応させ、条件本文を複製していない。
- [x] project overrideをコード/adapterへ隠さず、Codex既定値とrole-local不変条件を区別している。
- [x] `sol-high` profile、risk signals、1 point見積もりが規則どおりである。
- [x] QA_PLANがTASK-firstで独立作成され、同一candidateからのQA/REVIEWとmerge後環境確認を実施できる。
- [x] dependency-ready reconciliationと完了経路preflightが完了している。
- [x] DEV開始を承認した。
