---
task_id: "TASK-0030"
change_class: safety_contract
status: approved
qa_agent: "qa-agent-terra-medium"
approved_by: "main-agent-sol-high"
approved_at: "2026-07-20T09:38:13+09:00"
revision: 2
implementation_reviewed_at: ""
expectation_changed: true
expectation_change_approved_by: "main-agent-sol-high"
---

# TASK-0030 QA PLAN

## TASK-first baseline

この初版はTASK.mdだけから作成した。`qa` roleの正規契約
`gpt-5.6-terra` / `medium` は[`.codex/agents/qa.toml`](../../../agent-harness/.codex/agents/qa.toml)
と現在のQA割当で一致する。これは製品artifactを変更しない
`safety_contract`であり、製品DEV、製品candidate、`QA_RESULT.md`の製品PASSは対象外である。
TASK修正により、運用リポジトリの`run-lap30`と外部pathはこの製品merge treeで完成を検証できない
ため対象外とする。この除外は初版からのTASK-first期待値の意味変更であり、Main承認はpendingである。

## 共通の観測とfail-closed

全ケースは`evidence-review`とする。受け入れ真実は完成した規約・template・skill・検査設定の
差分と相互参照にあり、実OS権限、実配置、外部作用を要求しないためである。QAはPLANのPASSを
待たず、TASK-firstの基準と完成差分を独立に照合する。

- 証跡は評価対象tree、`git diff --check`、変更path一覧、各検索/検査のcommand・環境・exit、
  差分digest、未実施理由（あれば）とする。
- AC、許可path、検査、または規範間の意味が不明・欠落・矛盾する場合、`evidence-review`はPASSに
  しない。製品artifact又は宣言済み製品依存が差分に含まれた場合は安全契約経路を停止し、製品変更へ
  再分類する。
- 実権限、実配置、外部service、副作用、restart/rollback/cleanupが正しさに必要と判明した場合は
  `live-e2e`へ昇格する。隔離済み環境と安全なcleanupがなければblockedであり、別modeで代替PASSは
  作らない。

## 受け入れ条件との対応

| ケースID | AC-ID | 観測方法 | 期待する境界 | fail-closed |
|---|---|---|---|---|
| SC-001 | AC-1 | planning input packet の定義箇所を差分横断で読み、目的、非対象、条件ID、安定参照、依存状態、許可path、preflight、未決事項が一つの入力として参照されることを照合する。 | 同じ入力をPLAN/QA_PLAN又はskillへ再記述させず、依存の状態と未決事項を隠さない。 | 必須fieldの欠落、複数の正本、参照不能な入力、又は依存/許可pathの曖昧さはFAIL候補。 |
| SC-002 | AC-2 | TASK、PLAN/QA_PLAN template、関連規約の役割を比較し、条件ID対応表と各文書の所有情報を確認する。 | PLANは設計差分・範囲・順序・失敗時・見積りだけ、QA_PLANは条件IDごとの観測だけを所有する。 | TASK本文の複製、対応表の欠落、PLANとQA_PLANの期待値の二重正本はPASS不可。 |
| SC-003 | AC-3 | dependency state と reconciliation の記述を読み、依存前に確定できる計画と依存ready後の差分承認の接続を追跡する。 | dependency wait はactive planning計測から分離され、固定不能な値だけがreconciliation対象になる。 | 依存待ちをactiveとして記録する、依存前の推測を確定扱いする、又は差分承認なしにDEVへ進める経路はPASS不可。 |
| SC-004 | AC-4 | 10分超のactive planningに対する記録契約と停止/次行動を、packet、template、製品側開発規約で照合する。 | 指定された六分類を記録し、文章推敲だけの継続を抑止する。 | 10分超を無記録で許す、原因分類が欠落する、又は磨き込み継続を停止条件として扱わない場合はPASS不可。 |
| SC-005 | AC-5 | DEV開始前のpreflight checklistとcompletion経路を追い、checker、権限、依存、生成物、tree、Lap logの書込・Schema・repository annotationがそれぞれ検査可能か確認する。 | preflight未完了ではDEV gateが開かず、Lap logの開始確認は既存Schema/JSONLを変更せずに行う。 | 一項目でも自己申告だけ、検査不能、又はpreflight失敗後にDEV許可となる場合はPASS不可。 |
| SC-006 | AC-6 | 製品側`run-efficient-task-delivery`と開発文書の差分を読み、delivery skillが上位開発文書を再掲せず、実行時の判断と参照先へ縮約されることを検索する。 | 参照により重複を除去しても、分類、ロール分離、Main Git所有、安全gateを弱めない。 | 開発文書の実質的な再掲が残る、参照先がない、又は縮約により必須統制が消える場合はPASS不可。 |
| SC-007 | AC-7 | 許可pathと全文差分を検査し、既存Lap Schema/JSONL、製品gate、role separation、Main Git所有、安全境界に関するnegative検索を実施する。 | 新契約は既存統制と公開済み計測の意味を保存する。 | 製品コード/test/runtime/build/製品Schema/依存/生成artifact、Lap Schema/既存JSONL、又は統制を弱める差分はPASS不可。 |
| SC-008 | 対象外 | 製品candidateの許可pathと完了証跡を検査し、運用リポジトリの`.agents/skills/run-lap30/**`、その完成検査、及び外部ガバナンスcommitをTASK-0030の完了根拠に接続していないことを確認する。 | `run-lap30`の縮約は別Taskで扱い、本Taskは製品側の契約だけで閉じる。 | 外部pathの差分・検査結果・commitを本TaskのPASS又はmerge tree一致の条件にする場合は範囲逸脱としてPASS不可。 |

## 完成時の対象検査

完成差分に合わせて次を実行し、SC-001〜SC-007へ対応付ける。PLANが追加の専用検査を定義する場合は、
TASK-first期待値を変えずにこの集合へ加える。

```sh
git diff --check <planning-base>...<planning-candidate>
git diff --name-only <planning-base>...<planning-candidate>
rg -n 'planning input packet|dependency-ready reconciliation|active planning|preflight' AGENTS.md docs .agents
rg -n 'Repository `AGENTS.md` and development contracts take precedence|\[.*\](.*docs/development' .agents/skills/run-efficient-task-delivery/SKILL.md
make check
make task-check TASK=TASK-0030
```

`make check`と対象規約検査は安全契約の検証証跡であり、製品QA PASSの根拠にはしない。FAILはログと
再現性を添えて`implementation_defect`、`qa_plan_defect`、`requirement_gap`、`environment_issue`、
`regression`のいずれかへ分類候補を示し、DEV起因と推定しない。

SC-008のnegative checkは、製品candidateの変更path・PLAN・HANDOVERを読み、
`../agent-harness-work/.agents/skills/run-lap30/**`、外部repositoryのcommit/tree、又は同skillの完成検査を
検証対象として列挙していないことを確認する。外部pathを横断して検査・変更・clean判定をしない。

## PLAN突合と実施不能時

- PLAN完成後、AC-ID、許可path、preflight、対象検査、依存ready reconciliationをこの基準と突合する。
  期待値または範囲を変える必要があれば、理由を改訂履歴へ記録し、Main承認前にPASSにしない。
- 規範の探索範囲、完成tree、又は検査環境が確定できないときはblockedとし、未実施理由・影響case・
  再現条件を残す。安全契約の計画レビューを製品QA PASSで置換しない。
- 既存Lap Schema/JSONL及び公開済みTask証跡は遡及変更しない。Mainは安全契約の承認candidate treeと
  no-ff merge treeを照合し、不一致ならSC-001〜SC-007の影響を再評価する。

## 改訂履歴

| 改訂 | 日付 | 変更者 | 変更内容 | main承認 |
|---:|---|---|---|---|
| 1 | 2026-07-20 | QA Agent | TASK-first初版。AC-1〜AC-7の静的観測、fail-closed境界、対象検査を定義。 | `pending` |
| 2 | 2026-07-20 | QA Agent | TASK修正を反映し、SC-006を製品側delivery skillと開発文書の参照検査へ変更。運用側`run-lap30`・外部pathをSC-008 negative checkで対象外化した。 | `main-agent approved` |
