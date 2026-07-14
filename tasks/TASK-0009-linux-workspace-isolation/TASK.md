---
task_id: "TASK-0009"
title: "Linux P0 Workspace隔離とプロセス実行を実装する"
status: ready
created_at: "2026-07-15"
---

# TASK-0009 Linux P0 Workspace隔離とプロセス実行を実装する

## 目的

Linux P0プロファイルでTask Workspaceを隔離して実行し、ホストへの不要な読み書き・認証情報・IPC・直接外向き通信を拒否しながら、管理対象プロセスを確実に回収する。

## 背景

Workspaceはpolicyの適用主体であり、Task/Agentの自己申告で隔離を代替できない。`docs/07-governance.md`はLinux P0を初期正本とし、P0を全て強制できないOSプロファイルを提供しないとする。

## スコープ

### 対象

- Linux capability probeとP0 profile admission、workspace root生成、mount/PID/IPC/network namespaces、rootless/no_new_privs/capability drop、cgroup/PID cleanup。
- 明示allowlist以外のhost pathと管理socketを非公開にし、秘密情報をmount/environmentへ継承しない。
- proxy endpointのみ到達可能にするnetwork namespace/firewall設定、失敗時fail-closed、隔離試験。
- `empty` WorkspaceのみのP0実行adapterとworkspace-created境界イベント。

### 対象外

- `fork`/`shared_readonly`、snapshot/quota/seccomp/device制御などP1以降。
- macOS/Windows native profile、CASB proxy本体・ポリシー判定、一般的なコンテナorchestration。
- root helperによる昇格を前提とした代替経路。

## 受け入れ条件

- [ ] Linux P0 profileは必要なkernel機能を起動前にprobeし、一つでも不足すれば実行を開始せず理由を返す。
- [ ] sandbox processはTask workspaceと明示read-only runtime allowlist以外を読み書きできず、host credentialとCore/Governance管理socketを参照できない。
- [ ] sandbox processはrootless、`no_new_privs`、capability dropで起動され、子孫を含めてTask終了/失敗/cancel時に回収される。
- [ ] sandbox networkは管理proxy/DNS endpoint以外への直接接続を拒否し、強制点を構成できない場合はfail-closedする。
- [ ] workspace IDとTask IDを対応付けた生成・終了・隔離失敗の監査記録が残り、隔離侵害テストが自動化される。

## 検討すべき設計観点

- Linux P0とP1の境界、workspace IDをOS network identityへ結合する方法。
- mount/IPC/network/cgroupの失敗を部分隔離として続行しないfail-closed設計。
- process lifecycleと論理Workspace lifecycleを分離し、cleanupの冪等性を担保する。

## 完成の定義

- [ ] 受け入れ条件を満たしている。
- [ ] 必要な実装、テスト、文書が完成している。
- [ ] 独立レビューと`make check`がPASSしている。
- [ ] マージ後QAが完了している。
- [ ] HANDOVERがWiki Agentに取り込みされている。

## 関連コンテキスト

### 意味 Wiki

- `docs/00-kakesu.md` §§2.1–2.2, 4: Workspaceと外部境界。
- `docs/01-domain-model.md` §3: logical Workspaceと物理実行resourceの分離。
- `docs/07-governance.md` §§2–3: Linux P0表、workspace-scoped policy、network fail-closed。
- `schemas/draft-v0/execution-plane/workspace-created.schema.json`: `empty`とgrant非継承。

### 判断

- Decision: Linux専用P0 adapterを先に提供し、namespace/firewallの一部でも確立できない環境ではprofileをadvertiseしない。

### 適用しなかった重要な判断

- Docker依存だけの抽象化は必要なnetwork/IPC強制を検証不能にするため不採用。macOS/Windows best-effort profileもP0としては不採用。
