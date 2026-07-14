---
kind: schema
title: Codex Agent Model Routing
---

# Codex Agent Model Routing

## 問い

固定した役割割当、限定的な調査委譲、二つのrepository root間の設定同期を、誰が検証・書込み・commitするかを曖昧にせずにどう運用するか。

## 正本と固定割当

製品repositoryのproject-scoped TOMLを正本とする。mainは`gpt-5.6-sol/high`、PLAN・QA・REVIEWは`gpt-5.6-terra/medium`、Explorerは`gpt-5.6-luna/medium`でread-onlyとする。DEVは、明確で局所的かつ機械検証可能で、risk signalがない場合だけLunaを選べる。それ以外、または横断性、契約、Schema、security、concurrency、migration、不明点がある場合はSolを選ぶ。LunaからSolへのpromotionは履歴とmain Agentの承認を要し、降格しない。

固定roleと異なるmodel、profile、effortのoverrideは起動前に拒否する。互換overrideはfixed roleを経由しない、明示されたlegacy経路に限る。

## Explorer境界

Explorerは明示launcherから、一回につき一つのboundedなread-only質問だけを受ける。rootから直接、またはroot→role→Explorerで起動できるが、最大depthは2、最大threadsは2である。編集、Git操作、scope拡大、再委譲を行わず、簡潔な根拠要約とfile referenceだけを返す。read-only sandboxだけで完結せず、固定prompt、closed stdin、no-child設定、負例試験、launcher traceでこの境界を検証する。

## 生成adapterとcommit境界

work repositoryのadapterは正本から決定的に生成し、digest付き完全一致検査でdriftを拒否する。global設定、未文書include、model alias、Agentの自己申告を入力にしない。

専用sync launcherの親が共通lockを全工程で保持し、生成対象、scope検査、stage、shared hook、commit、post-checkを所有する。子Agentとgeneric governance経路はadapter同期やcommit authorityを持たない。child failure、scope・stage・commit drift、hook failure、pre/post-check failureでは、開始前のHEAD、index、worktree、untracked状態へrollbackし、`commit:null`を記録する。no-opとCHECKは書込みもcommitもしない。

## 検証と環境帰属

routingは正規TOML、launcher入力検査、決定的な正負試験、簡潔なlauncher traceで監査する。Explorerの境界は自己申告ではなくtraceで判定する。

独立QA環境でlock所有者の確認や製品build出力への書込みが許可されない場合、その未完走を環境所見として記録する。lock解放後または必要なwrite権限を持つmain実行環境で同じ必須commandが成功したとき、それは製品受け入れfailureや自動的なDEV帰責にしない。

## 関連

- [QA FAIL attribution](../case-patterns/qa-fail-attribution.md)
- [Codex agent model routing](../../decisions/DECISION-0003-codex-agent-model-routing.md)
- [TASK-0002 HANDOVER](../../../tasks/TASK-0002-codex-agent-model-routing/HANDOVER.md)
