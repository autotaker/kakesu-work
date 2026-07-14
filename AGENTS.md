# 運用リポジトリ規約

このリポジトリは`agent-harness`の開発状態と証跡の正本である。トピックブランチを作らず、`main`一本で運用する。

## 書き込み規則

1. 書き込み前に`main`であることとcleanな作業ツリーを確認する。
2. 子Agentは製品リポジトリの`AGENTS.md`に従い、内部`agents.spawn_agent`を標準経路として`agent_type`でロールを選び、異種ロールでは`fork_turns="none"`を明示して一体ずつ起動する。親は編集開始前からコミット後検査まで`.locks/`の共通排他制御を保持し、直接並行編集しない。`agent_type`欠落、内部spawn利用不能、またはmodel/effort不一致を停止・証跡化した場合だけ、`make -C ../agent-harness work-agent TASK=... ACTION=...`または役割別の専用コマンドをfallbackとして使う。
3. 一度のコミットへ複数Taskまたは複数フェーズの無関係な変更を混ぜない。
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
- `wiki/semantic/**`, `wiki/decisions/**`, `wiki/ingestions/**`, `wiki/index.json`: Wiki Agent
- `schemas/**`, `wiki/SCHEMA.md`, `wiki/AGENTS.md`: main Agent

## 禁止事項

- 製品コードをこのリポジトリへコピーしない。
- worktreeと生成HTMLをコミットしない。
- Task本文を`backlog.yaml`へ重複記録しない。
- FAILを自動的にDEV責任として記録しない。
- Wiki Agent以外が通常のWiki本文を直接保守しない。
