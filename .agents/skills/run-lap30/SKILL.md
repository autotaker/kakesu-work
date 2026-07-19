---
name: run-lap30
description: 製品Taskを原則1周、例外時も最大2周の30分でPLAN、QA_PLAN、DEV、独立REVIEW、QA、PRマージまで完了させ、固定スキーマのJSONLへ実績を継続記録する。30分タイムボックス、Lap30、1-Lap完了、後続Taskのパイプライン並列、開発時間の内訳計測、エージェント待ち時間の可視化、実績ベースのローリングウェーブ再計画を求められたときに使用する。製品変更を伴わない純粋な計測、status、証跡保守はカウント対象Lapから除外し、比例的な軽量経路へ送る。
---

# Lap30を実行する

30分をTaskの標準完成単位かつ安全な停止周期として扱う。1-LapでPLAN、DEV、独立REVIEW、QA、Main所有Gitまで完了することを標準とし、2-Lapは例外上限とする。時間制約を理由に確認工程、受け入れ条件、テスト、エラー処理、可読性を削らない。

リポジトリの`AGENTS.md`と開発契約を優先する。このスキルを厳しい契約の迂回に使わない。比例的な軽量経路が未定義なら、先に安全契約変更として承認する。

## カウント対象Lapかを分類する

製品コード、テスト、runtimeまたはbuild設定、Schema、依存、生成入力、外部観測可能な挙動のいずれかが変わるTaskをカウント対象Lapとする。このTaskには完全なPLAN、QA_PLAN、DEV、独立REVIEW、マージ後QAを適用する。

製品artifactを変更しなくても、securityまたはauthority境界、threat model、受け入れ条件、機能scope、依存、resource cap、機能削減順、必須開発統制を変える作業は安全契約変更とする。TASKとPLAN、TASK-firstの独立QA_PLAN、独立した計画レビューを必要とする。製品実装がないのにDEV、REVIEW、QAの製品PASSを記録しない。リポジトリがこの作業をTaskとして計測すると明示した場合だけpartial intervalまたは計画Lapとして記録する。

次のすべてを満たす作業だけを純粋な証跡保守とする。

- 製品コード、テスト、設定、Schema、依存、build入力、生成製品artifactを変更しない
- security、authority、threat model、受け入れ条件、機能scope、依存、cap、機能削減順、必須統制の判断を変更しない
- backlog status、計測算術、SLOC、時間、retry、FAIL分類、証跡link、またはそれらのappend-only correctionだけを更新する

純粋な証跡保守はカウント対象Lapにしない。専用Task ID、worktree、PLAN、QA_PLAN、DEV Agent、QA Agent、単独PRをその保守のためだけに作らず、関連製品Taskのpostmerge closureへ可能な限り含める。Mainが目的、出典、対象path、算術、除外、影響する検査、訂正方法の短いchecklistを作り、独立Reviewer 1名が出典、算術、status遷移、cross-file metadata、Schema/parser、diff scope、secret不在を確認する。影響しない製品test suiteを繰り返さない。

分類に迷う場合は安全契約変更とする。Reviewerが意味の変更、根拠の矛盾、安全上の含意を見つけた場合は軽量経路を止めて再分類する。既知の算術を再確認するだけのzero-SLOC Taskやsynthetic Lapを作らない。

## 記録契約を読む

イベントを書き始める前に[固定JSON Schema](references/lap30-event.schema.json)を全文読む。このSchemaをLap30 JSONL v1の正本とする。

既定のログを運用リポジトリの`lap30/events.jsonl`とする。製品リポジトリへ実績ログを置かない。利用者が別の運用ログを指定した場合だけ保存先を変更する。

MainだけがJSONLへ追記する。子Agentは計測値と証拠をMainへ返し、ログを直接変更しない。

## Lap外で周回を準備する

開始前に依存PR、作業ツリー、ツール、権限、フィクスチャを確認する。失敗したpreflightを`not_started`として記録し、30分へ算入しない。準備不足やAgent応答待ちのままカウント対象Lapを開始しない。

Task開始時に子Agentの状態を一覧し、完了、失敗、中断済みAgentから必要な証拠をMainが回収した後、利用可能なcleanup/release操作で終了Agentを解放して実行枠を空ける。activeなAgentや未回収の証拠を持つAgentは解放しない。ランタイムにcleanup操作がなければ、その制約をpreflight証拠へ記録し、ロール契約が一致する終了Agentを再利用する。空き枠を確認する前に新しいAgentを起動しない。

1周の成果を「必須ゲートを通過してPRをマージする」のように一文で定義する。未完でもゴールを縮小しない。

カウント対象の製品Taskでは、PLANとQA_PLANを別担当に所有させ、Lap開始前に可能な限り並行作成する。QA担当は最初にTASKだけから受け入れ条件、境界、失敗条件を作り、その後PLANと突合する。両方の承認前にDEVを開始しない。

QA_PLANの各必須条件を、実行するテスト名または検査方法へ対応付ける。未対応行を残したままDEV readyまたはgate-readyと扱わない。計画、受け入れ条件、依存関係、preflightが揃い、先行Taskが安全な区切りへ到達した時点で、次の30分境界を待たずカウント対象Lapを開始する。

後続TaskのTASK-first QA_PLAN、PLAN、読み取り専用preflightを先行Taskの後半に準備し、単純な計画待ちを次Lapへ持ち込まない。依存Taskの結果で前提が変わった箇所だけをLap開始前に差分確認する。Lap外準備も作業時間であり、partial intervalとして計測する。目安のactive timeを5分以内とし、超える場合はTask境界または計画入力の不備を疑う。partial intervalを使って1-Lap完了率だけを良く見せない。

## 1-Lapで閉じる

1周の目安を次のように置く。Taskの性質に応じて調整してよいが、遅延を検知したら同じ工程を漫然と延長しない。

- 0〜5分: DEV開始、ReviewerとQAが受け入れ対応表を確認
- 5〜20分: DEVとテスト、Reviewerは利用可能な差分を読み取り専用で先行確認
- 20〜25分: 独立REVIEWと修正、QA実行
- 25〜30分: Main所有Git、マージ後確認、記録

Reviewerの先行確認は最終REVIEWを代替しない。DEV担当へ実装を指示したり、同じ担当が自己承認したりしない。完成差分に対する独立判定を必ず残す。

10分時点でDEVが始まっていない、20分時点でREVIEW可能な候補がない、または25分時点で必須ゲートが残る場合は、`checkpoint`へ原因と回復策を記録する。Mainは待機せず、依存確認、受け入れ対応表、差分スコープ、Git準備を進める。

局所SLOC見積りは圧縮目標にしない。可読な最小実装がソフト目標を少量超えても、それだけを理由に契約変更や再承認を挟まない。リポジトリのハードリミット、機能削減順、承認済み安全境界に抵触するときだけ停止して再計画する。外部契約や安全境界を変更しない計測修正のためだけに契約専用PR、zero-SLOC Task、または追加Lapを作らない。

## 例外時も最大2周で閉じる

1 Taskに認めるカウント対象Lapは最大2周とする。preflightが`not_started`で終わった区間は数えない。準備完了後にTaskをそのLapの主目的として宣言し、Task固有の`lap_started`を記録したときだけ1周として数える。

別TaskのLap途中から後続Taskを準備した区間は`partial interval`とし、その後続Taskの2周制限へ数えない。partial intervalは1 Taskにつき最初の1回だけ認め、TASK-first PLAN、QA_PLAN、読み取り専用preflightに限定する。DEVを開始する時点で、そのTaskのカウント対象Lap 1を開始する。partial intervalを未計上のDEV、REVIEW、QA、Git時間として使わない。準備の作業時間、待ち時間、再試行、FAILは通常どおり記録し、成果と証拠をLap 1へ引き継ぐ。

Lap 1で完了しなかった場合、Lap 2へ進む前に次のすべてを確認し、`lap_stopped`の`next_action`と`annotations`へ根拠を残す。

- 残件が具体的な1〜2個に限定されている
- 設計変更、要件調査、Task境界の再定義を必要としない
- 次Lapの最初の20分以内に修正候補をREVIEWへ渡せる
- PLANとQA_PLANの対応表が更新済みである

満たさない場合はLap 2で広いTaskを続行せず、失敗証拠を保存して未完了または`superseded`として閉じ、残作業を新しいTask IDへ分割する。Lap 2を標準所要時間、計画修正用の予備時間、または自動延長として扱わない。

カウント対象Lap 2の終了時に全必須ゲートとMain所有Gitまで完了していなければ、そのTaskを未完了として閉じる。Lap 3、同じTask IDのfresh cycle、Lap番号の付け替え、partial intervalによる延長を禁止する。FAIL、テスト、差分、SLOC、時間、待ち、再試行の証拠を保存してから、Mainが失敗Task所有の未コミット製品・テスト差分だけをresetする。

resetには専用worktreeをmerged baseから安全に作り直す方法、またはリポジトリで承認されたpath-scopedな復元方法を使う。`git reset --hard`で利用者、他Task、他Agentの変更を消さない。公開済み履歴、Lap30 JSONL、PLAN、QA_PLAN、REVIEW、QAのFAIL証拠を書き換えない。reset後は実測を正本としてバックログを再計画し、残作業を新しいTask ID、独立した受け入れ境界、見積り、最大2周のPLAN/QA_PLANへ分割する。元TaskはDoneにせず、未完了または`superseded`として追跡する。

## 周回を記録する

カウント対象周回の開始時に`lap_started`を追記する。各区間で`stage_started`と`stage_finished`を対にして追記する。10分と20分で`checkpoint`を追記する。失敗時は`failure`を追記する。

純粋な証跡保守のために架空のTask ID、`lap_id`、`lap_started`を作らない。関連製品Taskがまだ開いている場合は、そのTaskの`git`またはpostmerge closureの証拠として記録して`lap_completed`前に閉じる。関連Taskが完了済みなら、リポジトリで承認された軽量checklistとcommit証拠を使い、Schemaに存在しない保守eventをJSONLへ無理に押し込まない。公開済みLapを再開、改番、またはLap 2化しない。

完了時は`lap_completed`、30分停止時は`lap_stopped`を必ず追記する。既存行を変更しない。誤記は元行を残し、`correction`を追記して`annotations.corrects_event_id`で対象を示す。

イベントの順序は`lap_id`内で単調増加する`sequence`にする。再試行ごとに`attempt`を増やす。同じ区間の実作業時間を`active_ms`、Agentや外部状態の待ち時間を`wait_ms`へ分離する。周回全体の経過時間は`elapsed_ms`へ記録する。

Schemaの全フィールドを毎行出力し、適用不能な値を`null`にする。未知のトップレベル項目を追加しない。例外的な補足だけを`annotations`へ文字列で格納する。継続集計する値を`annotations`へ逃がさず、Schema改訂で正式フィールドにする。

## 区間を進める

カウント対象の製品Taskでは次の順序を守る。

1. `plan`
2. `qa_plan`
3. `dev`
4. `review`
5. `qa`
6. `git`

DEV、Reviewer、QAを別担当にする。REVIEW PASSとQA PASSの両方が揃うまでマージしない。FAILを消さず、`classification`で原因を分類する。

Mainは応答待ち中に読み取り専用調査、依存確認、作業ツリー準備、認証確認を進める。待ち時間の短縮を理由に順序や役割分離を崩さない。

依存関係、ファイル所有、Agent枠が衝突しない場合は、現在TaskのREVIEW、QA、Gitと、後続TaskのTASK-first QA_PLAN、PLAN、読み取り専用preflightをパイプライン並列する。後続TaskのPLANとQA_PLANも別担当で並行作成する。実行環境や依存関係により並列化できない場合は、その理由を記録する。依存Taskのマージと後続TaskのPLAN/QA_PLAN承認が揃う前に後続DEVを開始しない。Main所有Gitと運用ログ書き込みは直列化し、並列化を理由にゲート、独立ロール、共通ロック、証拠記録を省略しない。

## 30分で停止する

30分時点で新しい作業を始めない。進行中の担当へ停止と現状報告を求め、利用可能な集中確認だけを行う。

未承認候補をマージしない。原則としてREVIEWとQAが通っていない候補をコミットまたはpushしない。リポジトリ規約がWIPコミットを明示的に許可する場合だけ例外とする。

`lap_stopped`へ、完了区間、現在区間、テスト結果、計画/実績SLOC、test LOC、最も時間を使った問題、次の行動を記録する。

## 実績から再計画する

最初の2〜3タスクだけを詳細化し、その後に`remeasurement`区間を置く。再計測の結果反映は可能な限り直前の製品Taskの25〜30分にあるMain所有Gitとpostmerge closureへ含め、既知の計測値を同期するためだけの独立TaskやPRにしない。

再計測ではJSONLを正本として、PLAN、QA_PLAN、DEV、REVIEW、QA、Git、Agent待ちを分離集計する。計画/実績SLOC、test LOC、再試行、FAIL分類も比較する。親Task完了後に純粋なstatusまたは算術修正が必要になった場合は、追加Lapではなく軽量checklistと独立Reviewer 1名で閉じる。

観測が少ない間は固定SLOC速度を仮定しない。観測済み周回から次の2〜3タスクだけを1-Lap完了可能な境界へ分割し、次の再計測地点を設定する。20%の余裕はTask境界と見積りへ含め、Lap 2を予定バッファにしない。

再計測では1-Lap完了率、partial intervalを含むTask総active time、DEV開始までの時間、REVIEW開始までの時間、計画・待機時間の比率、Lap 2使用理由、同一原因の再計画回数を確認する。目安として1-Lap完了率80%以上、Lap外準備5分以内、DEV開始5分以内、REVIEW開始20分以内、計画・待機時間20%以下を狙う。未達なら安全工程を削らず、Task境界、先行準備、受け入れ対応表、並列化の不足から改善する。

## ログの完全性を守る

OTP、TOTPシード、トークン、秘密鍵、認証情報、コマンド出力全文を記録しない。`summary`と`next_action`へ簡潔な非機密情報だけを書く。

追記前に各行が単独のJSONオブジェクトであること、改行を値に含めないこと、Schema v1に適合することを確認する。追記後にJSONL全行を再読し、JSONとして解釈できることを確認する。

Schemaを変更するときは`schema_version`を増やし、旧行を書き換えない。v1の意味を後から変更しない。
