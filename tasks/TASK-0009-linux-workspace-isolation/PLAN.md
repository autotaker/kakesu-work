---
task_id: "TASK-0009"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_by: ""
approved_at: ""
approved_dev_profile: "sol-high"
planned_implementation_files: 18
planned_implementation_lines: 1170
estimate_points: 8
---

# TASK-0009 PLAN

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| profile admission | required Linux capabilitiesを欠くhostでsandbox起動が拒否される | `docs/07` §2 |
| filesystem/IPC | host secret・管理socket・非allowlist pathへのアクセス試験が失敗する | `docs/07` P0表 |
| process lifecycle | cancel/failure後にprocess group/cgroup内の子孫が残らない | `docs/07` P0表 |
| network | proxy/DNS以外へのdirect routeが拒否され、強制失敗時にworkloadを起動しない | `docs/07` §§3–5 |

## 関連Wikiと判断

- `docs/00-kakesu.md` §§2.1–2.2, `docs/01-domain-model.md` §3。
- `docs/07-governance.md` §§2–5のLinux P0表とfail-closed要件。
- `docs/13-technology-stack.md`のGo Core/Execution boundary、Task-0006 wire schema。

## 設計

### 選択案

Go Execution PlaneにLinux専用`P0WorkspaceManager`を置く。profile probeがuser/mount/PID/IPC/network namespaces、cgroup v2、必要なfirewall backendを確認する。createはprivate runtime/work root、秘密値を除いたenvironment、mount namespace/read-only runtime allowlist、PID/IPC namespace、network namespace、rootless/no_new_privs/capability drop、cgroupを準備してからexecする。どの強制点も失敗したらworkloadを開始せず全resourceをcleanupする。networkはproxy/DNS endpointだけをnamespace内で許可する。

### 代替案と不採用理由

- container runtimeへの丸投げ: P0証拠とidentity bindingを明示できないため不採用。
- partial isolation fallback: P0を満たさないので不採用。
- macOS/Windows adapter: 初期Linux P0の範囲外。

### 責務と境界

- profile probe/admission、workspace layout、namespace/mount/network setup、process/cgroup lifecycle、audit recordを別packageにする。
- Controlはlogical workspaceを所有し、Executionは物理resourceを所有する。Governanceはpolicy/enforcementを所有し、本Taskはproxyそのものを実装しない。

### 不変条件

- `workspace_id`は一つのnamespace/cgroup/audit identityへ対応し、Task/Agent変更でpolicy主体を変えない。
- workspace外書込み、host credential、host IPC/management socket、直接外向き通信、privilege escalationはdefault deny。
- isolate/create/execが一つでも失敗すればcommandをexecしない。cleanupは冪等で子孫全体に及ぶ。
- P0は`empty`だけで、grant継承は空、fork/shared-readonlyは許可しない。

### 失敗時・移行・互換性

- capability probe失敗はunsupported profileとして明示し、best-effortで起動しない。
- mount/network/cgroup途中失敗はreverse-order cleanup、cleanup失敗は監査しretry可能なorphanとして残す。
- process exitはaudit/eventを記録し、cancel/timeout/shutdownは同じkill/reap pathを使う。既存workspace形式の移行はない。

## 変更予定

見積もり対象は実装コード、Schema、設定ファイルだけとする。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `core/internal/execution/linux/profile.go` | implementation | 75 | P0 capability probe/admission |
| `core/internal/execution/linux/profile_test.go` | implementation | 60 | probe failure tests |
| `core/internal/execution/linux/workspace.go` | implementation | 130 | empty root/layout/create cleanup |
| `core/internal/execution/linux/mounts.go` | implementation | 105 | mount namespace/allowlist |
| `core/internal/execution/linux/namespaces.go` | implementation | 115 | user/PID/IPC namespace setup |
| `core/internal/execution/linux/network.go` | implementation | 120 | namespace/firewall/proxy-only routing |
| `core/internal/execution/linux/process.go` | implementation | 105 | exec/no_new_privs/cap drop |
| `core/internal/execution/linux/cgroup.go` | implementation | 75 | cgroup tracking/reap |
| `core/internal/execution/linux/audit.go` | implementation | 60 | lifecycle/violation records |
| `core/internal/execution/linux/cleanup.go` | implementation | 70 | idempotent reverse cleanup |
| `core/internal/execution/linux/integration_test.go` | implementation | 160 | fs/ipc/net/process isolation tests |
| `core/internal/execution/service.go` | implementation | 65 | manager lifecycle wiring |
| `core/internal/execution/workspace.go` | implementation | 75 | platform-neutral port/models |
| `core/internal/execution/testdata/allowlist.json` | fixture | 25 | runtime mount policy |
| `core/internal/execution/testdata/proxy-routes.json` | fixture | 25 | allowed endpoints |
| `core/cmd/kakesu/main.go` | implementation | 35 | Linux profile config bootstrap |
| `core/go.mod` | config | 10 | Linux syscall/cgroup dependency |
| `schemas/draft-v0/execution-plane/workspace-created.schema.json` | schema | 40 | P0 empty/profile binding validation |

## 見積もり

```text
file_score = ceil(planned_implementation_files / 3)
line_score = ceil(planned_implementation_lines / 200)
estimate_points = 1, 2, 3, 5, 8, 13のうちmax(1, file_score, line_score)以上の最小値
```

## 実装手順

1. Linux P0 acceptance matrixをtestable probe・isolation casesへ変換する。
2. platform-neutral portとLinux probe/workspace/cgroup primitivesを実装する。
3. mount/PID/IPC/user/network isolationとproxy-only firewallを失敗時cleanup込みで接続する。
4. process supervisor/auditを追加し、host-path/credential/socket/network/descendant testsを実行する。
5. Task-0010 consumerが`empty` workspaceをprepare/cleanupできる契約を確定する。

## 検証計画

- Linux runner上でrootless profile admission、negative filesystem/IPC/credential tests、direct network reject/proxy allow test、cancel descendant reapingを実行する。
- unit testsはprivileged host依存をmockし、integrationはP0機能を実測する。機能不足をPASS扱いにしない。
- `go test ./...`、schema validation、`make check`、`git diff --check`。

## 未解決事項

- P0 firewall backend（nftables等）とrootless環境の正確な要件はDEV開始前に対象Linux CI imageでprobeして固定する。満たせないimageはP0対応として扱わない。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV開始を承認した。
