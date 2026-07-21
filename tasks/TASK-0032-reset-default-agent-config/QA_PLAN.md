---
task_id: "TASK-0032"
change_class: "product"
status: approved
qa_agent: "qa-agent-terra-medium"
approved_by: "main-agent-sol-high"
approved_at: "2026-07-22T08:09:25+10:00"
revision: 1
implementation_reviewed_at: ""
expectation_changed: false
expectation_change_approved_by: ""
---

# TASK-0032 QA PLAN — Reset project Agent configuration to defaults

## 方針

この計画はPLANを入力とせず、TASKの`planning input packet`だけからDEV開始前に作成する。条件本文や設計を再掲せず、各AC-IDへ観測を対応させ、各ケースへ`qa_execution_mode`を一つだけ理由付きで割り当てる。`evidence-review`はcandidate-bound証跡の独立監査、`focused-rerun`はhermetic・deterministic・上限付き フィクスチャで完全再現できる高リスクケース、`live-e2e`は実OS権限/auth（sudo/PAMを含む）、実配置、外部作用、実restart/ロールバック/クリーンアップ、環境固有integrationを必要とするケースに使う。条件が不明な場合は強いモードまたはblockedへfail-closedし、PASSで代替しない。

## 前提と環境

- QAはDEVがHANDOVERへ固定した同一`candidate_commit`と`candidate_tree`を照合し、REVIEWの結果を待たずに独立開始する。未固定、不一致、又はcandidate-boundな証跡のdigest不一致は全ケースをPASSにしない。
- 本計画の期待値はTASK packetのAC-1〜AC-5およびREF-1〜REF-4だけから固定した。`PLAN.md`を入力にも期待値の根拠にも用いない。期待値変更はない（`expectation_changed: false`）。
- QAが再実行する設定/adapter/routingの検査は、candidate treeを使う隔離されたworktreeで実行し、必要な作業ディレクトリを実行前後に記録する。既存のuser-level、system/managed設定は読取り・変更ともに対象外とする。
- 既定値`agents.max_threads=6`、`agents.max_depth=1`は、project-level `[agents]`を削除した状態でREF-3の未指定時既定へ委ねることを観測対象とする。ローカル環境に別の設定を足して既定値を模倣してはならない。
- 運用側生成物を更新する`make work-config-sync`、生成後のreload/restartと安全なロールバックは製品merge後のMain所有であり、実環境とcleanupが承認・隔離されるまで`live-e2e`はblockedのままとする。

## 受け入れ条件との対応

| ケースID | AC-ID | 観測方法 | `qa_execution_mode` / 理由 | 必要証跡 | fail-closed |
|---|---|---|---|---|---|
| QA-001 | AC-1 | candidateの`.codex/config.toml`をTOMLとして読み、`features.multi_agent_v2` table、`hide_spawn_agent_metadata`、`tool_namespace`の不在を確認する。削除対象を意図的に含むfixture/negative testが失敗することもrouting testで確認する。 | `focused-rerun` / 禁止設定の不在は設定回帰であり、隔離candidateで静的TOML検査とnegative fixtureを決定的に再実行できる。 | `candidate_commit`/tree、設定ファイルdigest、TOML key-path列挙、routing testのコマンド・fixture・cache・exit・出力digest、禁止keyを再導入したnegative検出、test削除/弱体化差分監査。 | candidate/tree不一致、TOML解析不能、対象keyの検査不能、negative検出なし、又は禁止keyが残存/許容ならFAILまたはblocked。DEV自己申告や単なるdiff閲覧でPASSにしない。 |
| QA-002 | AC-2 | candidateのproject設定に`[agents]`または子tableがないことをTOML key-pathで確認し、設定なしのcandidateからdefault値6/1に委ねる検査を独立実行する。top-level `model`、`model_reasoning_effort`、`sandbox_mode`が変わっていないことも固定REF-2との差分で確認する。 | `focused-rerun` / defaultへの委譲と許可された非対象設定の保持はhermeticな設定解析/adapter fixtureで完全再現できる。 | candidate/tree、REF-2との対象key差分、TOML key-path出力、default 6/1を明示するtest/fixtureの入力・期待値・exit・digest、negative（`[agents]`/子tableを戻す）検出、cache条件。 | `[agents]`/子tableの残存、6/1以外のproject override、defaultを別project設定で代用、非対象top-level設定の変更、又はdefault検査/negative fixtureの欠落はPASS不可。 |
| QA-003 | AC-3 | `scripts/task/agent-routing.test.mjs`をcandidateで独立実行し、adapterの決定的出力に禁止設定と`[agents]`がないこと、意図的driftで`make work-config-sync CHECK=1`が非zeroになること、同期済み入力ではzeroになることを観測する。 | `focused-rerun` / adapter生成とdrift検出はcandidate内のbounded fixtureでdeterministicに再現できる。 | candidate/tree、routing testコマンド・fixture・cache・exit・出力digest、生成済み運用config digest/TOML key-path、clean CHECK=1のexit、故意drift時のnonzero exitと検出メッセージ、fixture cleanup記録。 | adapter出力が禁止設定を生成、clean/drift双方のCHECK=1を観測できない、driftがzero、fixtureが実運用configを破壊、又はtest弱体化/証跡不足ならFAIL又はblocked。 |
| QA-004 | AC-4 | standalone `.codex/agents/*.toml`が存在し、固定roleのmodel/effortとfallback launcherの検査がproject-level registryなしで通ることをrouting testとcandidate差分で確認する。registryを復元せず、standalone roleの欠落・model/effort不一致を各negative fixtureで拒否できることを確認する。 | `focused-rerun` / role TOMLとlauncher routingは隔離fixtureでboundedかつdeterministicに検査でき、REF-4のrole分離/Git境界を観測できる。 | candidate/tree、role TOML一覧とdigest、roleごとのmodel/effort期待値、routing/launcher検査コマンド・fixture・cache・exit・digest、registry不在の証拠、欠落/不一致negative結果、差分のGit境界監査。 | role fileの削除・role/model/effort/sandbox契約変更、fallbackの緩和/削除、registryへの依存、negative拒否不能、又はcross-role起動/Git禁止境界を検査できない場合はPASS不可。 |
| QA-005 | AC-5 | candidateで設定生成・routing test、`make check`、`make task-check TASK=TASK-0032`、`git diff --check`を独立実行する。削除済み設定を戻す変更が少なくとも一つのrouting/adapter検査で失敗すること、およびテスト削除/期待値緩和がないことを差分とnegative testで確認する。 | `focused-rerun` / 必須検査はcandidate worktreeで上限付きに再実行でき、設定回帰はnegative caseで直接観測できる。 | candidate/tree、各コマンドの環境・cache・exit・出力digest、実行test名、negative結果、test/fixture差分、未実施理由、`make check`とtask checkの完全ログ参照。 | 必須コマンドの未実施/nonzero、候補と証跡不一致、禁止設定再導入を検出しない、test削除/弱体化、scope外の設定・依存変更、又は影響不明はFAIL又はblocked。 |
| QA-006 | AC-3, AC-5（マージ後環境依存） | Mainが製品merge後に専用ランチャー親/共通lock下で`make work-config-sync`を実行し、運用側生成configを更新してからCHECK=1を再実行する。承認済み隔離実環境でCodexをreload/restartし、standalone role起動と既定値委譲が有効で、禁止設定不在により起動/routingが壊れないことを観測する。 | `live-e2e` / 運用生成物、実reload/restart、実ランタイム設定とcleanupは環境固有であり、focused rerun/evidence reviewで代替できない。 | merge commit/treeとcandidate tree比較、lock所有者、sync前後config digest/TOML key-path、sync/CHECK=1のコマンド・exit・ログdigest、実環境識別・承認、reload/restart操作とruntime観測、role起動記録、rollback/cleanup結果。 | merge_tree不一致は影響を再評価するまでblocked。承認済み隔離実環境、restart/reload権限、安全なrollback/cleanup、又はruntime観測のいずれかがなければblocked。sync失敗、drift残存、禁止設定の再生成、runtime routing失敗はFAIL候補。 |

## 境界・異常・回帰

- 高リスク、証跡欠落、ダイジェスト/コミット/tree不一致、テスト削除/弱体化、影響不明は`evidence-review` PASSにしない。
- `qa_carry_forward`は`QA_RESULT.md`の`CF-1`から`CF-7`を全て証明できる場合だけ許可する。影響QAケース集合が空でなければ該当ケースを再実行し、影響を限定できなければ全面再実行とする。QA FAIL、受け入れ条件/QA_PLAN変更、認証認可、秘密、sudo/PAM、IPC/Schema/設定/依存、並行性/ライフサイクル/persistence/エラー/fail-closed、テスト削除/弱体化、影響不明、証跡と評価対象の案/tree不一致では禁止する。
- `live-e2e`の環境または安全なクリーンアップが用意できない場合はblockedとして残す。
- 本Taskは設定・生成物・routingを変更するため、QA-001〜QA-006に影響する候補修正は`qa_carry_forward`の対象外である。Mainは影響ケースを再実行し、影響を限定できなければ全ケースを再実行する。
- 初期FAIL分類は、AC/REFと異なる候補挙動は`implementation_defect`（DEVへ）、TASK packetの矛盾・未定義は`requirement_gap`（Main/PLANへ）、本計画の手順・fixtureの誤りは`qa_plan_defect`（QAへ）、隔離環境・lock・reload/restart・cleanup不能は`environment_issue`（Main/環境へ）、既存設定/adapter/routingの破壊は`regression`（DEVへ）とする。最終分類はMainが証拠に基づき決定し、DEV起因と仮定しない。

## 実施不能時の扱い

- 未実施コマンド、環境、原因、残余リスクを記録し、別モードの結果でPASSにしない。FAIL候補の分類と復帰先はMainが判断する。
- QA-006がblockedのままなら、QA-001〜QA-005のPASSをもってTask全体のPASS、マージ後確認完了、又はruntime既定値の確認とはしない。

## 案とマージ後確認

- DEVは`candidate_commit`、`candidate_tree`、ケース ID、コマンド/テスト、環境/フィクスチャ、cache条件、exit、成果物 ダイジェスト、未実施理由をHANDOVERへ記録する。
- REVIEWとQAは同一案から相互のPASSを前提にせず独立に開始する。
- 修正後の`qa_carry_forward`はMainだけが`QA_RESULT.md`の`CF-1`から`CF-7`を全て記録して選択する。
- `merge_tree == candidate_tree`かつ環境依存ケースなしなら全面確認を省略できるが、環境依存ケースはマージ後もケース単位で確認する。

## 実装後の再確認

- [ ] 実装差分とレビュー結果を確認した。
- [ ] 操作手順を現行実装に合わせた。
- [ ] 期待結果または試験範囲の変更有無を確認した。
- [ ] 期待結果または範囲を変更した場合、main Agentの承認を得た。
- [ ] QA-006をmerge後の承認済み環境で実施した、又はblocked理由・環境/cleanup不足・残余リスクを記録した。

## 改訂履歴

| 改訂 | 日付 | 変更者 | 変更内容 | main承認 |
|---:|---|---|---|---|
| 1 | 2026-07-22 | qa-agent-terra-medium | TASK-firstの独立QA計画。AC-1〜AC-5を5件のfocused rerunと、merge後のwork-config-sync/reload/restartを1件のlive-e2eへ対応付け。期待値変更なし。 | `main-agent-sol-high` / 2026-07-22T08:09:25+10:00 |
