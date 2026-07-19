---
name: run-lap30
description: 30分を1周、1 Taskを最大2周としてPLAN、QA_PLAN、DEV、独立REVIEW、QA、PRマージを進め、固定スキーマのJSONLへ実績を継続記録する。30分タイムボックス、Lap30、後続Taskのパイプライン並列、開発時間の内訳計測、エージェント待ち時間の可視化、実績ベースのローリングウェーブ再計画を求められたときに使用する。
---

# Lap30を実行する

30分を完成期限ではなく、観測と安全な停止の周期として扱う。時間制約を理由に確認工程、受け入れ条件、テスト、エラー処理、可読性を削らない。

## 記録契約を読む

イベントを書き始める前に[固定JSON Schema](references/lap30-event.schema.json)を全文読む。このSchemaをLap30 JSONL v1の正本とする。

既定のログを運用リポジトリの`lap30/events.jsonl`とする。製品リポジトリへ実績ログを置かない。利用者が別の運用ログを指定した場合だけ保存先を変更する。

MainだけがJSONLへ追記する。子Agentは計測値と証拠をMainへ返し、ログを直接変更しない。

## 周回を準備する

開始前に依存PR、作業ツリー、ツール、権限、フィクスチャを確認する。失敗したpreflightを`not_started`として記録し、30分へ算入しない。

Task開始時に子Agentの状態を一覧し、完了、失敗、中断済みAgentから必要な証拠をMainが回収した後、利用可能なcleanup/release操作で終了Agentを解放して実行枠を空ける。activeなAgentや未回収の証拠を持つAgentは解放しない。ランタイムにcleanup操作がなければ、その制約をpreflight証拠へ記録し、ロール契約が一致する終了Agentを再利用する。空き枠を確認する前に新しいAgentを起動しない。

1周の成果を一文で定義する。未完でもゴールを縮小しない。

PLANとQA_PLANを別担当に所有させる。可能なら並行作成する。QA担当は最初にTASKだけから受け入れ条件、境界、失敗条件を作り、その後PLANと突合する。両方の承認前にDEVを開始しない。

## Taskを最大2周で閉じる

1 Taskに認めるカウント対象Lapは最大2周とする。preflightが`not_started`で終わった区間は数えない。Taskを30分境界でそのLapの主目的として宣言し、Task固有の`lap_started`を記録したときだけ1周として数える。

別TaskのLap途中から後続Taskを開始した区間は`partial interval`とし、その後続Taskの2周制限へ数えない。partial intervalは1 Taskにつき最初の1回だけ認め、作業時間、待ち時間、SLOC、再試行、FAILを通常どおり記録する。次の30分境界で、そのTaskを主目的とするカウント対象Lap 1を開始するか、停止するか、全ゲートを満たしていれば完了する。partial intervalの成果と証拠はLap 1へ引き継ぎ、境界到達を理由にresetしない。partial interval中に完了した場合は、カウント対象Lap 0、partial interval 1として完了証拠へ記録する。

カウント対象Lap 2の終了時に全必須ゲートとMain所有Gitまで完了していなければ、そのTaskを未完了として閉じる。Lap 3、同じTask IDのfresh cycle、Lap番号の付け替え、partial intervalによる延長を禁止する。FAIL、テスト、差分、SLOC、時間、待ち、再試行の証拠を保存してから、Mainが失敗Task所有の未コミット製品・テスト差分だけをresetする。

resetには専用worktreeをmerged baseから安全に作り直す方法、またはリポジトリで承認されたpath-scopedな復元方法を使う。`git reset --hard`で利用者、他Task、他Agentの変更を消さない。公開済み履歴、Lap30 JSONL、PLAN、QA_PLAN、REVIEW、QAのFAIL証拠を書き換えない。reset後は実測を正本としてバックログを再計画し、残作業を新しいTask ID、独立した受け入れ境界、見積り、最大2周のPLAN/QA_PLANへ分割する。元TaskはDoneにせず、未完了または`superseded`として追跡する。

## 周回を記録する

周回開始時に`lap_started`を追記する。各区間で`stage_started`と`stage_finished`を対にして追記する。10分と20分で`checkpoint`を追記する。失敗時は`failure`を追記する。

完了時は`lap_completed`、30分停止時は`lap_stopped`を必ず追記する。既存行を変更しない。誤記は元行を残し、`correction`を追記して`annotations.corrects_event_id`で対象を示す。

イベントの順序は`lap_id`内で単調増加する`sequence`にする。再試行ごとに`attempt`を増やす。同じ区間の実作業時間を`active_ms`、Agentや外部状態の待ち時間を`wait_ms`へ分離する。周回全体の経過時間は`elapsed_ms`へ記録する。

Schemaの全フィールドを毎行出力し、適用不能な値を`null`にする。未知のトップレベル項目を追加しない。例外的な補足だけを`annotations`へ文字列で格納する。継続集計する値を`annotations`へ逃がさず、Schema改訂で正式フィールドにする。

## 区間を進める

次の順序を守る。

1. `plan`
2. `qa_plan`
3. `dev`
4. `review`
5. `qa`
6. `git`

DEV、Reviewer、QAを別担当にする。REVIEW PASSとQA PASSの両方が揃うまでマージしない。FAILを消さず、`classification`で原因を分類する。

Mainは応答待ち中に読み取り専用調査、依存確認、作業ツリー準備、認証確認を進める。待ち時間の短縮を理由に順序や役割分離を崩さない。

依存関係、ファイル所有、Agent枠が衝突しない場合は、現在TaskのREVIEW、QA、Gitと、後続TaskのTASK-first QA_PLAN、PLAN、読み取り専用preflightをパイプライン並列してよい。後続TaskのPLANとQA_PLANも可能なら別担当で並行作成する。依存Taskのマージと後続TaskのPLAN/QA_PLAN承認が揃う前に後続DEVを開始しない。Main所有Gitと運用ログ書き込みは直列化し、並列化を理由にゲート、独立ロール、共通ロック、証拠記録を省略しない。

## 30分で停止する

30分時点で新しい作業を始めない。進行中の担当へ停止と現状報告を求め、利用可能な集中確認だけを行う。

未承認候補をマージしない。原則としてREVIEWとQAが通っていない候補をコミットまたはpushしない。リポジトリ規約がWIPコミットを明示的に許可する場合だけ例外とする。

`lap_stopped`へ、完了区間、現在区間、テスト結果、計画/実績SLOC、test LOC、最も時間を使った問題、次の行動を記録する。

## 実績から再計画する

最初の2〜3タスクだけを詳細化し、その後に`remeasurement`区間を置く。

再計測ではJSONLを正本として、PLAN、QA_PLAN、DEV、REVIEW、QA、Git、Agent待ちを分離集計する。計画/実績SLOC、test LOC、再試行、FAIL分類も比較する。

観測が少ない間は固定SLOC速度を仮定しない。観測済み周回から次の2〜3タスクだけを分割し、20%の時間余裕と次の再計測地点を設定する。

## ログの完全性を守る

OTP、TOTPシード、トークン、秘密鍵、認証情報、コマンド出力全文を記録しない。`summary`と`next_action`へ簡潔な非機密情報だけを書く。

追記前に各行が単独のJSONオブジェクトであること、改行を値に含めないこと、Schema v1に適合することを確認する。追記後にJSONL全行を再読し、JSONとして解釈できることを確認する。

Schemaを変更するときは`schema_version`を増やし、旧行を書き換えない。v1の意味を後から変更しない。
