---
task_id: "TASK-0017"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_dev_profile: "sol-high"
approved_by: ""
approved_at: ""
planned_implementation_files: 14
planned_implementation_lines: 1050
estimate_points: 8
---

# TASK-0017 PLAN

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| namespace/firewall | sandboxから許可proxy以外のconnectが失敗 | docs/07 §§2-3 |
| DNS capture | A/AAAAのみ正規化、DoH/生IP/rebinding負例 | docs/07 §5 |
| policy handoff | identity/canonical requestが0016 decisionへ到達 | docs/07 §§3,7 |
| fail closed | adapter/proxy/governance停止中の転送なし | docs/07 §3 |

## 関連Wikiと判断

- docs/07, 11, 12, 13。依存: TASK-0009、0016。Linux P0の経路固定とdefault denyを採る。

## 設計

### 選択案

Linux adapterがrootless network namespaceとWorkspace固有firewallを生成し、DNS/proxy listenerだけをallowする。DNSは統治へqueryを渡し、HTTPS proxyはcanonical metadata/identityで0016を照会し、allow結果後だけ接続する。

### 代替案と不採用理由

- 環境変数proxyだけ: direct socketを防げない。
- transparent proxyのみ: identity/hostnameの明示境界が弱い。
- TLS MITMやcredential injectionを同時実装: P0経路固定を超える別機能。

### 責務と境界

- execution Linux adapter: namespace/firewall/lifecycle。
- DNS/HTTPS bridge: canonicalizationとgovernance request、外部転送。
- governance: policy判断。TLS本文・認証情報・non-HTTP TLSは後続Task/phase。

### 不変条件

- sandboxから外部へ直結不可、DNS/proxy以外はdeny、allow前に外部接続なし、Workspace identityはsandbox強制点由来、障害はdeny、ホスト管理socket/credentialsを公開しない。

### 失敗時・移行・互換性

Linux capability/iptables-nft要件が満たせない環境はprofileを起動しない。起動途中/cleanup失敗は子processを回収し既存ruleを残さない。非Linuxはunsupportedとして明示する。

## 変更予定

| ファイル群 | 種別 | 概算行数 | 変更内容 |
|---|---|---:|---|
| core/internal/execution/linux/{namespace,firewall,identity,lifecycle}.go | implementation | 420 | P0隔離と回収 |
| core/internal/egress/{dns_proxy,https_proxy,canonicalize,governance_client}.go | implementation | 310 | 捕捉/正規化/handoff |
| core/internal/{app,execution}/**/*_test.go | test | 180 | isolation/bypass/failure |
| scripts/{linux-egress-fixture,verify-linux-profile}.sh | test tooling | 80 | Linux integration fixture |
| docs/runtime/linux-profile.md | documentation | 60 | 前提/非対応機能 |

実装対象14ファイル・1,050行。`ceil(14/3)=5`、`ceil(1050/200)=6`のため8pt。

## 実装手順

1. Linux profile preflight、namespace/identity/firewall lifecycleを作る。
2. managed DNS/HTTPS bridgeと0016 clientを接続する。
3. IPv4/IPv6/bypass/rebinding/障害の隔離試験を追加する。
4. Linux専用の起動診断を文書化し`make check`へ統合する。

## 検証計画

- direct TCP/UDP、DoH/DoT/QUIC、raw/ICMP、生IP/private/metadata、IPv6、proxy/governance停止、cleanup/restart。
- Linux integration fixture、Go tests、schema/tabletop、`make check`。

## 未解決事項

- CIでnamespace/firewallを許可するLinux runner capabilityをQA_PLANで固定する。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV開始を承認した。
