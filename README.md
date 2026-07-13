# agent-harness-work

`agent-harness`の開発バックログ、Task証跡、開発用Wiki、Decisionを管理する運用リポジトリである。製品コードとはGit履歴を分離し、このリポジトリ自身はトピックブランチを作らず`main`一本で運用する。

## 配置

```text
backlog.yaml       EpicとTask状態の正本
project.yaml       製品リポジトリとの対応
tasks/             Taskごとの6証跡
wiki/semantic/     複数Taskで再利用する知識
wiki/decisions/    確定済みDecision
wiki/ingestions/   HANDOVER ingestの冪等証跡
worktrees/         製品リポジトリのworktree、Git管理外
viewer/index.html  生成ビューアー、Git管理外
```

## 共通コマンド

製品リポジトリの再利用可能なツールを使う。

```sh
make -C ../agent-harness work-check
make -C ../agent-harness work-agent TASK=TASK-0001 ACTION=plan PROFILE=planner
make -C ../agent-harness backlog-view
make -C ../agent-harness task-check TASK=TASK-0001
make -C ../agent-harness wiki-context TASK=TASK-0001 WIKI_CONTEXT_TARGET=task
make -C ../agent-harness wiki-ingest TASK=TASK-0001
```

すべての更新は`AGENTS.md`に従い、所有者が検査後に`main`へ直接コミットする。
