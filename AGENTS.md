# 運用リポジトリ規約

このリポジトリは`agent-harness`の開発状態と証跡の正本である。トピックブランチを作らず、`main`一本で運用する。

## 作業経路の分類

### 製品変更

製品コード、テスト、ランタイム/build設定、Schema、宣言済み製品依存の追加・削除・バージョン・設定、生成される製品入力または成果物、外部観測可能な挙動のいずれかを変更する場合は、Taskを起票し、完全な`PLAN → QA_PLAN → DEV → 独立REVIEW → マージ後QA`の証跡を保持する。

### 安全契約変更

製品成果物および宣言済み製品依存を一切変更しない場合に限り、セキュリティ/権限境界、脅威モデル、受け入れ条件、機能スコープ、依存方針またはTask順序、リソース上限、機能削減順、または必須開発統制の変更を安全契約変更とする。この経路ではTask、PLAN、TASK本文だけから先に作る独立QA_PLAN、独立した計画レビューを必須とする。製品実装がない場合は、製品DEV、`REVIEW_RESULT.md`、`QA_RESULT.md`のPASSを作らない。

### 純粋な証跡保守

次のすべてを満たす場合だけ、専用Task、PLAN、QA_PLAN、DEV Agent、QA Agent、カウント対象Lap、単独PRを省略できる。

- 変更がバックログ状態、計測算術、SLOC、時間、再試行、FAIL分類、証跡リンク、またはそれらの`append-only correction`だけである。
- 製品変更にも安全契約変更にも該当せず、挙動、ポリシー、受け入れ条件の意味を変えない。
- Main Agentが目的、出典、対象パス、算術または対応関係、除外、影響する検査、訂正またはロールバック方法のchecklistを、関連TaskのHANDOVERまたはcommit証跡へ残す。
- 独立レビュアー1名が出典、算術、状態遷移、ファイル間メタデータ、Schema/parser、差分スコープ、秘密情報不在を確認する。

公開済み証跡を書き換えず、既存の訂正方式で履歴を保存する。影響しない製品テスト群は繰り返さず、可能なら関連製品Taskのマージ後処理へ含める。分類に迷う場合、またはレビュアーが根拠の矛盾、受け入れ条件の意味変更、安全上の含意を発見した場合は軽い経路を停止し、安全契約変更へ再分類する。

## 書き込み規則

1. 書き込み前に`main`であることとcleanな作業ツリーを確認する。
2. 子Agentは製品リポジトリの`AGENTS.md`に従い、内部`agents.spawn_agent`を標準経路として`agent_type`でロールを選び、異種ロールでは`fork_turns="none"`を明示して一体ずつ起動する。親は編集開始前からコミット後検査まで`.locks/`の共通排他制御を保持し、直接並行編集しない。`agent_type`欠落、内部spawn利用不能、またはmodel/effort不一致を停止・証跡化した場合だけ、`make -C ../agent-harness work-agent TASK=... ACTION=...`または役割別の専用コマンドをfallbackとして使う。
3. 一度のコミットへ複数Taskまたは複数フェーズの無関係な変更を混ぜない。関連Taskのマージ後処理へ含める純粋な証跡保守は、同じTaskの証跡として扱う。
4. 各Agentは所有する証跡だけを変更する。
5. `backlog.yaml`のフェーズ変更、例外、FAILの差し戻し先はmain Agentが決定する。
6. Wiki配下の変更には`wiki/AGENTS.md`を追加適用する。
7. コミット前に共有pre-commit hookと`make -C ../agent-harness work-check`を通す。hookを迂回しない。

## 証跡所有者

- `TASK.md`: main AgentまたはTask起票者
- `PLAN.md`: Planner Agent、承認欄はmain Agent
- `REVIEW_RESULT.md`: Reviewer Agent
- `QA_PLAN.md`: QA Agent、期待結果と範囲の変更承認はmain Agent
- `QA_RESULT.md`: QA Agent、差し戻し判断はmain Agent
- `HANDOVER.md`: DEV Agent、QA Agent、main Agent
- `backlog.yaml`: main Agent
- 軽量checklist: main Agent（関連Taskの`HANDOVER.md`またはcommit証跡）
- `wiki/semantic/**`, `wiki/decisions/**`, `wiki/ingestions/**`, `wiki/index.json`: Wiki Agent
- `schemas/**`, `wiki/SCHEMA.md`, `wiki/AGENTS.md`: main Agent

## 禁止事項

- 製品コードをこのリポジトリへコピーしない。
- worktreeと生成HTMLをコミットしない。
- Task本文を`backlog.yaml`へ重複記録しない。
- FAILを自動的にDEV責任として記録しない。
- Wiki Agent以外が通常のWiki本文を直接保守しない。
