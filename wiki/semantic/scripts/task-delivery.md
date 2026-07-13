---
kind: script
title: Task Delivery
---

# Task Delivery

## Trigger

バックログから実施するTaskを選び、main AgentがPlanner Agentをアサインする。

## 標準進行

1. Planner AgentがWikiとDecisionを参照してPLANを作る。
2. main AgentがPLANを承認する。
3. QA Agentが実装前QA計画を作る。
4. main AgentがDEV、Reviewer、QA、ブランチ、worktreeを割り当てる。
5. DEV Agentが実装、テスト、文書、`make check`を完了する。
6. Reviewer Agentが独立レビューし、PASSを記録する。
7. main Agentが`--no-ff`で`main`へマージする。
8. QA Agentがマージ済み`main`を受け入れレビューする。
9. HANDOVERをWiki Agentがingestし、Taskを閉じる。

## 分岐

- QA計画の誤りはQAへ戻す。
- 実装不具合はDEVへ戻す。
- 要件または設計の不足はPLANへ戻す。
- 重大なマージ後不具合はrevertし、限定的不具合はバグTask化できる。

## 終了条件

受け入れ条件、レビュー、QA、HANDOVER、Wiki ingestが完了し、発見事項が解消または追跡されている。

## 関連

- [QA FAIL attribution](../case-patterns/qa-fail-attribution.md)
- [PLAN DEV QA process](../../decisions/DECISION-0001-plan-dev-qa-process.md)
