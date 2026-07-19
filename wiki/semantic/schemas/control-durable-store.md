---
kind: schema
title: Control Durable Store
---

# Control Durable Store

## 責務と境界

Control-owned Storeは`control.db`の唯一のwriterであり、root Taskのcurrent record、owner、workspace参照、contract snapshot、progress、作成eventを永続化する。接続時にはWAL、foreign keys、busy timeoutを設定する。

この境界はroot Taskの作成とその再open可能なread modelを扱う。ownerの単一割当・解放、lifecycle遷移、contract/progressの更新履歴、Schemaの意味検証、transport、Plane間原子性は別の責務である。

## Schema versionの受理条件

version tableは、認識済みのcurrent versionが**ちょうど一行**ある場合だけ既存Schemaとして受理する。未知version、重複行、異なるversionの混在は、値の一つが期待値でもfail closedにする。新規DBのmigrationは冪等に初期versionを作るが、既存DBの曖昧な状態を推測して修復しない。

この検査は「最新versionが含まれる」ことではなく、Storeが前提にできるSchema状態が一意であることを保証する。

## 原子的作成と返却値

作成は一つのtransactionに閉じる。成功時にはTask、owner、workspace、contract v1 snapshot、progress v0、`TaskCreated` sequence 1、`OwnerAssigned` sequence 2がそろい、失敗時にはそのいずれも残さない。storage conflictやmigration失敗はsilent overwriteや自動retryではなく、typed errorとして返す。

commit後に同じcancel可能contextでDB readをすると、commit済みなのに呼出元へerrorを返す場合がある。そのため、成功時のread modelはtransaction内で組み立て、commit成功後にそれを返す。commit後のreadを成功条件にしない。

## 適用限界

この設計は単一SQLite Store内の原子性だけを保証する。外部サービス、他Plane、配備手順、または後続のversioned recoveryを含む操作のatomicityを意味しない。

## 関連

- [TASK-0027 handover](../../../tasks/TASK-0027-control-durable-foundation/HANDOVER.md)
