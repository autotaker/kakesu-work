---
kind: case-pattern
title: QA FAIL Attribution
---

# QA FAIL Attribution

## 発生条件

実装前に作ったQA計画と、実装後に観測した動作の期待が一致しない。

## 典型的な誤り

QAのFAILを即座にDEV不具合と判断すると、QA計画の誤り、Taskの曖昧さ、環境問題が実装差し戻しへ混入する。

## 対策

FAILを実装不具合、QA計画不具合、要件不足、環境問題、回帰へ分類する。QA Agentは実装後に計画を再確認できるが、期待結果または範囲の変更はmain Agentの承認を要する。差し戻し先はmain Agentが根拠とともに決める。

## 適用限界

分類は責任追及ではなく修正すべき契約層を特定するために使う。複数原因がある場合、一つへ無理に圧縮しない。

## 関連

- [Task delivery](../scripts/task-delivery.md)
