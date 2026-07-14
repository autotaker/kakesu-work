---
task_id: "TASK-0017"
title: "Linux外向き通信強制ブリッジを実装する"
status: ready
created_at: "2026-07-15"
---

# TASK-0017 Linux外向き通信強制ブリッジを実装する

## 目的

Linux P0 profileで、Work Agentのnetwork namespaceからmanaged DNSとmanaged HTTPS proxy以外の外向き通信を機械的に拒否し、許可通信をTASK-0016の統治判断へ強制捕捉する。

## 背景

CASB storeだけではCLIの直接接続を止められない。LinuxをMVP正本としてnamespace/firewall経路を固定し、Workspace network identityを強制点から与える必要がある。依存: TASK-0009、TASK-0016。

## スコープ

### 対象

- rootless Linux network namespace、Workspace識別、egress firewall、managed DNS/proxyへの唯一の経路と起動・回収。
- DNSのA/AAAAのみ、DNS/DoH/DoT/QUIC/raw/ICMP/生IP/private/link-local/metadata拒否、HTTPS proxyのcanonical request受け渡し。
- proxy/firewall/統治障害時のdefault denyと監査可能なblock。

### 対象外

- TLS本文検査、certificate interception、credential broker、非HTTP TLS/SNI L4 proxy。
- grant workflow（TASK-0018）、macOS/Windows profile、P1/P2隔離。

## 受け入れ条件

- [ ] sandbox processはnamespace内でmanaged DNSとHTTPS proxyだけに到達でき、外部IP/port・DNS bypass・UDP/QUIC・host/control socketへ直接到達できない。
- [ ] DNSは許可queryだけを統治へ正規化し、unknown FQDNを上流へ漏らさずchallenge経路へ渡す。
- [ ] proxyはWorkspace identityとcanonical HTTPS metadataをTASK-0016へ渡し、allow前には外側接続を作らない。
- [ ] setup/teardown/adapter障害はprofile提供または通信を拒否し、隔離・否定試験で再現できる。

## 検討すべき設計観点

- firewallが最終強制点、proxyはpolicy判定の迂回経路を持たない。IPv4/IPv6とDNS rebindingを同じdeny modelで扱う。

## 完成の定義

- [ ] 受け入れ条件を満たしている。
- [ ] 必要な実装、テスト、文書が完成している。
- [ ] 独立レビューと`make check`がPASSしている。
- [ ] マージ後QAが完了している。
- [ ] HANDOVERがWiki Agentに取り込みされている。

## 関連コンテキスト

### 意味 Wiki

- docs/07 §§2-5、docs/11 §2、docs/12 §5、docs/13 §§2,4。依存: TASK-0009、0016。
### 判断

- Linux P0だけを提供し、managed DNS/HTTPS proxy以外をdefault denyにする。
### 適用しなかった重要な判断

- TLS本文検査、credential broker、非HTTP TLS、OS横断profileを本Taskに含めない。
