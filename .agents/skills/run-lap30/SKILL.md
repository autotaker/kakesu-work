---
name: run-lap30
description: 製品Taskを原則1回、例外時も最大2回の30分タイムボックスで完了させ、固定SchemaのJSONLへ実作業・待ち・再試行・FAILを記録する。Lap30、1-Lap完了、10分ごとのcheckpoint、後続Taskのパイプライン準備、30分停止、実績ベースの再計画を求められたときに使用する。純粋な計測・status・証跡保守はカウント対象から除外する。
---

# Lap30を実行する

30分を製品Taskの標準完成単位かつ安全な停止周期として扱う。1回で完了することを標準とし、2回目は条件付きの例外上限にする。時間を理由に受け入れ条件、安全ゲート、テスト、エラー処理、可読性を削らない。

## 正本を先に読む

次を正本とし、このskillへ内容を複製しない。

1. 現在のリポジトリと運用リポジトリの`AGENTS.md`
2. 運用リポジトリの`project.yaml`が指す製品リポジトリにある`.agents/skills/run-efficient-task-delivery/SKILL.md`
3. 製品リポジトリの`docs/development/`
4. イベントを書く場合は[lap30-event.schema.json](references/lap30-event.schema.json)全文

作業分類、PLAN/QA_PLAN、ロール分離、QA mode、候補拘束、軽微修正、Main所有Git、完了条件は上記正本へ従う。Lap30固有なのは開始条件、時計、記録、停止、再計測だけである。

## Lap開始前に準備する

カウントするのは製品変更Taskだけとする。安全契約変更を計測する場合はリポジトリが明示した計画区間だけ、純粋な証跡保守はカウントしない。

Mainは開始前に次を満たす。

- 同じ`planning input packet`から作られたPLANとTASK-first QA_PLANが承認済み
- `make task-preflight TASK=TASK-NNNN`、依存、権限、worktree、生成物、fixtureが確認済み
- ログの書込、Schema検証、`annotations.repository`が確認済み
- この回で完成させる成果を一文で固定済み

不足は`not_started`として時計外で解消する。依存待ちやAgent枠待ちのまま`lap_started`を書かない。依存前に固定不能な値は推測せず、製品側の`dependency-ready reconciliation`へ戻す。

後続TaskのPLAN、TASK-first QA_PLAN、読み取り専用preflightは、現在Taskの後半に準備できる。準備時間もpartial intervalとして記録するが、後続TaskのLap回数には数えず、DEVへ流用しない。

## 30分を進める

- 0〜5分: DEV開始。Reviewer/QAはAC-ID対応と観点を確認する。
- 5〜20分: DEVが実装・テスト・候補証跡を作る。Reviewerは読める差分を先行確認する。
- 20〜25分: 固定候補への独立REVIEWとリスク別QAを並行する。
- 25〜30分: Mainが修正経路、Git、tree/環境確認、証跡を閉じる。

10分でDEV未開始、20分で候補なし、25分で必須ゲート未完了なら`checkpoint`へ原因と回復策を書く。Mainは待機中に依存確認、差分scope、候補/tree、Git準備を進める。先行レビューは完成候補の独立判定を代替しない。

SLOCは圧縮目標にしない。ソフト目標の小超過だけで再承認しない。ハード上限、安全境界、Task scopeへ触れる場合だけ再計画する。

## 例外の2回目を制限する

1回目で完了しない場合、次を全て満たすときだけ2回目へ進む。

- 残件が具体的な1〜2件
- 設計、要件、Task境界を変えない
- 次回の最初の20分で修正候補を渡せる
- PLAN/QA_PLANの対応が更新済み

満たさなければ同じTaskを延長せず、証拠を残して残件を再分割する。2回目でも完了しなければ未完了または`superseded`として閉じ、Lap 3、改番、fresh cycle、partial intervalによる延長をしない。利用者や他Taskの差分を破壊するresetを行わない。

## JSONLを記録する

既定は運用リポジトリの`lap30/events.jsonl`とし、製品リポジトリへ置かない。MainだけがSchemaに従ってappendし、子Agentは作業時間、待ち時間、SLOC、再試行、FAIL分類の根拠だけを返す。

- 開始: `lap_started`
- 各区間: `stage_started` / `stage_finished`
- 10分、20分: `checkpoint`
- 失敗: `failure`
- 完了: `lap_completed`
- 30分停止: `lap_stopped`

`sequence`を単調増加させ、再試行は`attempt`を増やす。`active_ms`と`wait_ms`を分け、全体を`elapsed_ms`へ記録する。Schemaの全fieldを出し、適用不能値は`null`にする。既存行を変更せず、訂正は`correction`と`annotations.corrects_event_id`で追記する。

OTP、TOTP seed、token、秘密鍵、認証情報、コマンド出力全文を記録しない。追記前後に全行をSchema検証する。Schemaの意味を変える場合はversionを上げ、旧行を書き換えない。

## 30分で停止し再計測する

30分時点で新しい作業を始めない。未承認候補をmerge/pushせず、完了区間、現在区間、テスト、計画/実績SLOC、test LOC、最大の遅延原因、次の行動を`lap_stopped`へ残す。

2〜3 TaskごとにJSONLから、1回完了率、Task総active time、DEV/REVIEW開始時刻、計画・待機比率、2回目の理由、再計画原因、SLOC/test LOC、再試行、FAIL分類を再計測する。次の2〜3 Taskだけを詳細化し、20%の余裕はTask境界へ入れる。目安は1回完了率80%以上、DEV開始5分以内、REVIEW開始20分以内、計画・待機20%以下とし、未達時は安全ゲートではなく入力packet、Task境界、preflight、引き渡し、並列準備を直す。
