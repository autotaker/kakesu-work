---
task_id: "TASK-0004"
status: draft
completed_at: ""
---

# TASK-0004 HANDOVER

## 成果

- 製品Task branch `task/TASK-0004-remove-redundant-agent-depth-limits`のcommit `32a1c38ace6b1395b82c5b001777355b459a9558`で、通常6 roleの重複したagent-local `max_depth = 1`を削除した。
- project-scopedな`agents.max_depth = 2` / `agents.max_threads = 2`を全体トポロジー上限として維持し、Explorer固有の`max_depth = 0` / `max_threads = 1`と既存の明示launcher contractを維持した。
- canonical parser、digest、adapter render/checkが、通常roleへのlocal depth再導入とExplorer no-childの欠落・緩和を別の理由で同期前にfail closedするようにした。

## 主要な変更

- `.codex/agents/{main,planner,qa,reviewer,dev-luna,dev-sol}.toml`から`max_depth`だけを削除し、各roleの`max_threads = 2`、Explorer registry、model、effort、sandboxは変更していない。
- `scripts/task/agent-routing.mjs`に通常roleの`[agents]`構造検査を追加し、local `max_depth`を`ROUTING_ROLE_LOCAL_DEPTH_FORBIDDEN`、thread policy driftを`ROUTING_ROLE_THREAD_POLICY_MISMATCH`で拒否する。
- `scripts/task/agent-routing.test.mjs`に、通常6 roleのdepth key不在、全roleへの値を問わない再導入、Explorer depth/thread緩和、project depth/thread drift、adapter同期前fail-closedの決定的fixtureを追加した。
- `docs/development/agent-roles.md`を更新し、通常roleがproject-scoped上限に従うことと、Explorerだけがrole-local no-child上限を持つことを区別した。

## 検証結果

- `git diff --check main...HEAD`: pass。
- `node --test scripts/task/agent-routing.test.mjs`: 11件pass、0件fail。
- `node --test scripts/task/development-process.test.mjs`: 21件pass、0件fail。
- 製品`make check`: pass。Go build/test/vet、Python build/pytest/ruff、Rust build/test/fmt/clippy、tabletop・用語・文書検査、process tests 32件がすべて成功した。
- 検証後の製品Task worktreeはcleanである。

## 判断

- 通常roleのchild到達可能性はproject `.codex/config.toml`だけを正本とし、role-local depth keyの不在自体をcanonical contractとして検査する。
- Explorerのno-childは通常roleと一括処理せず、Explorer TOML、固定prompt、read-only sandbox、明示launcher、closed stdin、write scopeなし、`commit:null`、child-spawn負例の多層contractとして維持する。
- canonical driftはdigest計算・adapter render/checkより前に拒否し、driftした設定からadapterを生成または更新しない。

## 既知の制約と未解決事項

- `REVIEW_RESULT.md`の先行`fail`は、TASK-0004実装前のproduct `main` commit `a0661057b56461c1a0e0f01a326d487b094e5ea1`を対象にした結果であり、実装commit `32a1c38ace6b1395b82c5b001777355b459a9558`は未レビューである。独立Reviewerによる正しいcommitの再レビューが必要である。
- product `main`への`--no-ff` merge、main Agent所有のwork adapter同期と完全一致検査、`make -C ../agent-harness work-check`、マージ後QAは未実施である。

## 運用上の注意

- DEVはproduct Task worktreeだけを変更した。work repositoryの`.codex/config.toml`はmain Agentがlock下の専用sync launcherで同期・commitする。
- product `main`にはTask外の未commit `.codex/config.toml`変更が観測されたが、DEVは変更・stage・commitしていない。main Agentはmerge前にこの既存差分を分類する必要がある。
- 再レビューではbranch名ではなく実装commit `32a1c38ace6b1395b82c5b001777355b459a9558`を明示して対象を固定する。

## Wikiへ引き渡す知識

### 再利用可能な知識

- 同じ深度keyでも、通常roleでは「keyが存在しないこと」、Explorerでは「`max_depth = 0`が存在すること」が別の不変条件になる。構造検査を分けることで、重複設定の再導入と権限境界の緩和を区別できる。
- canonical parserをdigest、adapter render、adapter checkの共通入口にすると、構造driftを生成・同期前に拒否し、adapterを変更前の状態に保てる。
- Explorer no-childは深度値だけに依存せず、固定prompt、明示launcher、read-only、closed stdin、write scopeなし、`commit:null`、child-spawn負例を重ねて検証する。

### 反例・失敗・注意点

- 通常roleとExplorerの`max_depth`を一括削除するとExplorer固有のno-child防御を失う。通常roleだけを列挙し、Explorerのrequired keyを別検査にする必要がある。
- projectの値だけを照合するテストでは通常roleへの重複key再導入を検出できない。各通常roleへの再導入をtemporary canonical fixtureで注入し、parser、digest、render、checkの全入口を確認する。
- Reviewerがproduct `main`を暗黙に対象にすると、未mergeのTask branch実装を見落として誤ったFAILになる。reviewed commitをlauncher入力と証跡で固定する。

### 更新候補ページ

- `wiki/semantic/schemas/codex-agent-model-routing.md`
- `wiki/semantic/scripts/task-delivery.md`

## ブートストラップ例外

- 該当なし
