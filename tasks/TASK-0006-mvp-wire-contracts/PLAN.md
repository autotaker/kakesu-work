---
task_id: "TASK-0006"
status: draft
planner_agent: "planner-agent-terra-medium"
approved_by: ""
approved_at: ""
approved_dev_profile: "luna-xhigh"
planned_implementation_files: 12
planned_implementation_lines: 760
estimate_points: 5
---

# TASK-0006 PLAN

## 受け入れ条件の具体化

| 条件 | 観測方法 | 根拠 |
|---|---|---|
| 3言語相互fixture | Go `go test`、Python `pytest`、Rust `cargo test`が同一JSON fixtureを読んでaccept/rejectを一致させる | `docs/13` §7、common envelope schema |
| Wire metadata | fixtureの必須metadata欠落・同一route・非object payloadを各adapterが拒否する | `schemas/AGENTS.md`、`message-envelope.schema.json` |
| ローカル設定 | config testがUDS endpointとschema rootを解決し、秘密値なし・Plane別endpointを確認する | `docs/00` §3.1 |

## 関連Wikiと判断

- `docs/00-kakesu.md` §3.1、`docs/13-technology-stack.md` §7。
- `schemas/README.md`のdraft-v0 revision規則、`schemas/AGENTS.md`のstrict/negative規則。
- 現行`core/internal/message/envelope.go`、`memory/.../message.py`、`governance/src/message.rs`。

## 設計

### 選択案

canonical JSON Schemaをwire正本とし、`fixtures/wire/envelope/*.json`に有効1件・不正4件を置く。Go/Python/Rust adapterは各fixtureをdecode後に、自言語の境界検査と同じaccept/reject結果を返す。ローカル設定は明示的な各Plane UDS pathとschema rootを持つ非秘密設定にし、後続Transportが使用する。

### 代替案と不採用理由

- Protobuf/gRPC: schema正本を重複させるため採用しない。
- 言語別fixture: driftを検出できないため採用しない。
- UDS実装まで含める: delivery/ACKの責務はTask-0008であり、契約Taskを膨張させない。

### 責務と境界

- Schema: wire構造・revision・digestの正本。config: endpoint命名とschema root。adapter: decode/最小境界検査。fixture: 言語非依存の互換証跡。
- payload意味検証はSchema validator導入後に各受信Planeが行う。本Taskのadapterはenvelope境界に限定する。

### 不変条件

- `message_id`は空でない、routeは異なるcomponent、correlation/timestamp/schema referenceは必須、payloadはobject。
- `causation_message_id`はexplicit nullを許し省略と混同しない。時刻はUTC RFC3339、schema revisionは1以上。
- fixtureのschema ID/revision/digestとcatalog実体が不一致なら受理しない。

### 失敗時・移行・互換性

- config不備・fixture parse失敗は起動/テストをfail-fastにする。
- draft-v0なのでbreaking変更は許すが、schema revisionとfixture参照を同時更新する。旧revisionの互換shimは作らない。
- listener/queueが未実装の段階ではendpointへ接続せず、設定解決までを検証境界にする。

## 変更予定

見積もり対象は実装コード、Schema、設定ファイルだけとする。

| ファイル | 種別 | 概算変更行数 | 変更内容 |
|---|---|---:|---|
| `schemas/draft-v0/common/message-envelope.schema.json` | schema | 35 | cross-language明確化に必要なmetadata/format調整 |
| `schemas/draft-v0/common/schema-reference.schema.json` | schema | 25 | schema参照のstrict検査 |
| `configs/local/wire.json` | config | 45 | Plane別UDS endpointとschema root |
| `fixtures/wire/envelope/valid.json` | fixture | 25 | 正常envelope |
| `fixtures/wire/envelope/*.invalid.json` | fixture | 100 | 4つの単一違反負例 |
| `core/internal/message/fixture_test.go` | implementation | 110 | shared fixture reader/assertion |
| `core/internal/config/wire.go` | implementation | 80 | config decode/validate |
| `core/internal/config/wire_test.go` | implementation | 70 | config tests |
| `memory/src/kakesu_memory/wire_fixtures.py` | implementation | 65 | fixture adapter |
| `memory/tests/test_wire_fixtures.py` | implementation | 70 | Python assertions |
| `governance/src/wire_fixtures.rs` | implementation | 65 | fixture adapter |
| `governance/tests/wire_fixtures.rs` | implementation | 70 | Rust assertions |

## 見積もり

```text
file_score = ceil(planned_implementation_files / 3)
line_score = ceil(planned_implementation_lines / 200)
estimate_points = 1, 2, 3, 5, 8, 13のうちmax(1, file_score, line_score)以上の最小値
```

## 実装手順

1. canonical schemaと3言語envelopeの差異を表にし、fixtureの正負ケースを固定する。
2. non-secret local configと各言語のconfig readerを追加する。
3. shared fixture readerを3言語へ追加し、同一結果をテストする。
4. canonical JSON serializationとSHA-256によるschema digest生成を決定的に実装し、schema/fixture digestの不一致とconfig不正を負例として追加する。

## 検証計画

- Go `go test ./...`、Python `uv run pytest`、Rust `cargo test`。
- fixtureの全ケースを言語横断で比較し、`git diff --check`とschema validatorを実行する。
- Task-0008着手前に、endpoint命名とschema rootをTransport計画と照合する。

## 未解決事項

- なし。schema digestのcanonical serializationと生成器は本Taskが所有し、後続Taskへ持ち越さない。

## main Agentレビュー

- [ ] 受け入れ条件が検証可能である。
- [ ] 設計観点と代替案を検討している。
- [ ] QA計画を作成できる。
- [ ] 見積もりが規則どおりである。
- [ ] DEV開始を承認した。
