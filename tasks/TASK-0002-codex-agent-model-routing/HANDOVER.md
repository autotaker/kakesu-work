---
task_id: "TASK-0002"
status: draft
completed_at: ""
---

# TASK-0002 HANDOVER

## 成果

- 製品Taskブランチのコミット`cfbf299`で、main / PLAN / DEV / QA / REVIEW / explorerのCodex model routingとreasoning effortをproject-scoped設定として実装した。
- 承認済み`sol-high`で、DEV profile証跡、限定read-only explorer、両repository rootの設定整合、親所有commit、簡潔な起動証跡を実装・検証した。
- 独立レビュー、製品`main`へのmerge、work adapterのgovernance commit、merge後QAは後続gateであり、本HANDOVERはDEV完了時点の`draft`である。

## 主要な変更

- 製品`.codex/config.toml`と7個のrole TOMLを正規契約とし、main=`gpt-5.6-sol/high`、PLAN/QA/REVIEW=`gpt-5.6-terra/medium`、DEV=`gpt-5.6-luna/xhigh`または`gpt-5.6-sol/high`、explorer=`gpt-5.6-luna/medium/read-only`を明示した。
- `agent-routing.mjs`に固定role override、DEVリスク選定とpromotion、最大depth/thread、bounded question、adapter digest/drift、起動証跡、子Agent結果のfail-closed検査を集約した。
- work / Wiki launcherは子stdinをclosedにし、子Agentのstage・commit・scope外変更を拒否する。子成功後だけ、共通lockを持つ親がscope検査、stage、共有hook、検証、commit後検査を行う。
- `make work-config-sync`で製品側の正規契約からwork側adapterを決定的に生成・検査できるようにした。adapterの更新とcommitは製品merge後にmain Agentが別のgovernance操作として行う。
- role、DEV選定、explorer、adapter drift、redaction、親所有commitのテストを追加し、Agent責務と開発プロセス文書、用語集を同期した。

## 検証結果

- `node --test scripts/task/agent-routing.test.mjs scripts/task/development-process.test.mjs`: 16件pass、0件fail。
- 製品Task worktreeの`make check`: pass。Go、Python、Rust、tabletop、用語、process test、lint、format、Clippy、文書lint、`git diff --check`を含む全検査が成功した。
- 製品Task worktreeからの`make work-check WORK_ROOT=/Users/autotaker/git/agent-harness-work`: pass。1 Epic、2 Task、7 Wiki pageを検証した。
- 製品`main`からの`make -C ../agent-harness work-check`は、merge前の旧validatorがwork側の割当symlinkとGit登録先の実体pathを文字列比較するため、実在するTASK-0002 worktreeを未登録と誤判定してFAILした。本Task実装は`realpath`比較へ変更しており、同じwork repositoryの検査がpassすることを確認した。
- 検査後の製品Task worktree: clean。検証対象コミットは`cfbf299`。
- `REVIEW_RESULT.md`と`QA_RESULT.md`はpendingであり、独立レビューおよびQA-001〜QA-013のmerge後受け入れ判定は未実施である。

## 判断

- DEV profileは承認済み`sol-high`を使用した。launcher、sandbox/commit所有権、兄弟2 repository root、設定契約、起動証跡を横断する高リスク変更であり、Luna条件を満たさない。
- role契約の正本は製品repositoryに一元化し、work repositoryにはmain-ownedの生成adapterだけを置く。手動複写やglobal設定、未文書include、model aliasには依存しない。
- fixed roleの不一致overrideは起動前FAILとし、同一値の冗長指定だけを許可する。既存override互換は固定roleを経由しないWiki legacy経路に限定する。
- 子AgentへGit権限を渡さず、lockとcommit権限をlauncher親へ集約する。role Agent、explorerへ承認・merge・commit権限を移さない。

## 既知の制約と未解決事項

- 独立Reviewerの判定、製品`main`への`--no-ff` merge、main-owned work adapterの生成・governance commit、merge済みmainでのQA-001〜QA-013は未完了である。
- work adapterは製品側の正規設定をmergeした後でなければ同期しない。drift検査とwork側governance commitが完了するまで、merge後QAへ進めない。
- merge前の製品`main`にある旧`work-check`は、worktree symlinkのpath差を吸収できない。本TaskをmergeするまではTASK-0002の現行Task worktreeから検査する必要がある。
- 本DEV検証は決定的な設定・fixtureを対象とし、live modelの応答品質やサービス可用性を受け入れ判定に含めていない。

## 運用上の注意

- role Agentは役割別launcherから起動し、固定契約と異なる`PROFILE` / `MODEL` / `EFFORT`を指定しない。
- explorerへの委譲は一回につき一件の限定されたrepository調査だけとし、短い根拠要約とファイル参照だけを受け取る。編集、Git操作、scope拡大、再委譲を許可しない。
- 製品merge後、main Agentは共通lock下で`make work-config-sync`を実行し、生成された`.codex/config.toml`だけを別のgovernance commitとして記録する。その後に`make work-config-sync CHECK=1`とmerge後QAを行う。
- launcher childは編集と検証だけを行う。stage、commit、merge、`.git`書き込みは行わず、親のscope・hook・検証が失敗した場合は`commit:null`として終了する。

## Wikiへ引き渡す知識

### 再利用可能な知識

- role別model routingは、正規TOML、launcherの入力検査、決定的テストを組み合わせると、利用者のglobal設定や自己申告に依存せず監査できる。
- 複数repository rootで同じproject設定を使う場合、正本を一方へ固定し、他方をdigest付き生成adapterにしてdriftをfail closedにすると所有境界を維持できる。
- DEVの低コストprofileは全低リスク条件を満たす場合だけ許可し、高リスクsignal、不明、横断性が一つでもあれば高能力profileへ倒す。promotionは履歴とmain承認を必要とし、降格は許可しない。
- 書き込み子Agentとlock所有launcher親を分離し、親だけがscope検査、stage、hook、commit、再検証を行うと、Agent責務とcommit authorityを一致させられる。

### 反例・失敗・注意点

- role値を複数launcherや両repositoryへ手動複写するとdriftする。正規契約から生成し、完全一致を検査する。
- read-only sandboxだけではexplorerのscopeと再委譲を説明し切れない。設定、固定指示、launcher検査、負例テストを重ねる。
- 子Agentのexit 0だけでcommitしてはいけない。HEAD、stage、変更scope、hook、変更前後の検証を親が確認する。
- `luna-xhigh`を単なる既定値にすると、契約、Schema、security、concurrency、migration、不明リスクを過小評価する。

### 更新候補ページ

- `wiki/semantic/scripts/task-delivery.md`
- `wiki/semantic/schemas/work-repository-boundary.md`
- `wiki/semantic/schemas/codex-agent-model-routing.md`（新規候補）
- `wiki/decisions/DECISION-0003-codex-agent-model-routing.md`（新規候補）

## ブートストラップ例外

- routing実装自身を起動するTASK-0002に限り、承認済み`sol-high`を一時的な明示profileとしてlauncher親から指定した。これは自己ホスト開始時の起動方法だけの例外であり、role分離、Task worktree、親所有commit、独立レビュー、`--no-ff` merge、merge後QAのgateは緩和しない。
