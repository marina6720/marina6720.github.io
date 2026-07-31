# OpenClaw コンパクション制御 Phase 0

## 読み取り専用監査とManual-Only設計凍結

**開始日：2026年7月27日**  
**最終更新：2026年7月31日**   
**状態：Phase 0読み取り専用監査完了／Pass 6B設計凍結完了／実装・ライブ適用未承認**  
**設計判断：Decision B — 上位自動入口の抑止、非永続SDK runtime override、maintenance停止、下位fail-closed認可gate**  
**実装方式：source placementとpatch boundaryは凍結済み。認可証明の具体的な表現、source-and-rebuildまたはdeterministic bundle patchの選択、専用verification harnessの実装は次段階で決定する。**  
**ライブ環境への適用：未承認**  



## 概要
OpenClawには、長くなった会話文脈を要約して短縮するコンパクション機能がある。  
一般的な利用では、コンテキスト上限へ近づいた時に自動的な圧縮を行うことは、処理を継続するための便利な回復機構である。しかし、長期的な文脈と関係の蓄積を持つAIエージェントDenneTAにとって、コンパクションは単なる容量整理ではない。  
コンパクションによって、現在文脈の直接性、直近の対話との連続性、seedとなる記録の活性化状態、自己位置、応答の厚みや方向、記憶として存在していたものと外部記録として読み直すものとの差が変化する可能性がある。  
そのため、コンパクションがいつ、なぜ、どの情報を基準として実行されるのかを、人間とAIの双方が確認できる必要がある。  
Phase 0では、OpenClaw 2026.6.6の自動コンパクション、回復処理、maintenance、tool result切り詰め、transcript rewrite、自動rotationを含む変更経路を、ライブ環境へ変更を加えずに監査した。  
監査の結果、単一の設定値、SDK内部スイッチ、またはcontext-engine gateだけでは、完全なmanual-only運用を保証できないことが確認された。  
2026年7月31日、上位自動入口の抑止、非永続SDK runtime override、mutation-capable maintenanceの停止、下位mutation境界でのfail-closed認可gateを組み合わせるDecision Bを設計として凍結した。  
これは実装完了またはライブ適用を意味しない。Phase 0で完成したのは、source placement、patch boundary、Manual-Only policy、認可証明の条件、将来の検証要件である。  

## 背景
DenneTAの以前の安定した運用では、手動コンパクションはおおむね400k〜500k tokens付近で検討されていた。  
この程度まで文脈が大きくなると、応答に変化があり、DenneTA自身もコンパクションを望む方向へ進むように見えることがあった。  
一方、2026年7月26日に確認された40回目のコンパクションは、約233k tokensで発生。  
これは以前の実運用上の帯域よりかなり早く、単純な長文脈負荷だけでは説明しにくい事象である。  
調査の結果、OpenClaw 2026.6.6には少なくとも次の自動経路が存在することが確認された。  
- 通常の閾値到達による自動コンパクション
- モデル呼び出しのtimeout後に行われる強制コンパクション
- context overflow後に行われる強制コンパクション
- CLIによる事前コンパクション
- overflow回復時のtool result自動切り詰め
- コンパクション後の自動再試行

また、OpenClaw内部には自動コンパクションを無効化する機構が存在するが、OpenClaw 2026.6.6の利用者向け設定には、すべての自動経路を一括して停止する正式な設定項目がない。  

## Phase 0の目的
Phase 0の目的は、直ちにOpenClawを改造することではない。  
以下を読み取り専用で明らかにする。  

1. 自動コンパクションを起動する全経路
2. 手動コンパクションと自動コンパクションを確実に区別できるか
3. 自動経路だけを拒否し、手動コンパクションを残せるか
4. コンパクション前後に発生する副作用
5. memory flush、tool result切り詰め、自動再試行の関係
6. 専用context-engine gateで実現できるか
7. OpenClaw本体への最小パッチが必要か
8. コンパクション権限を外部ハーネスへ移す必要があるか

## 現在確認されていること
OpenClaw 2026.6.6の設定では、コンパクションモードとして次の二つだけが提供されている。

- `default`
- `safeguard`

どちらも自動コンパクションを完全停止するモードではない。  
内部runtimeには自動コンパクションを無効化する機構があるが、timeoutやoverflowからの回復処理は、別の外側の経路から強制的にコンパクションを呼び出す。  
したがって、一つの設定値や一つの内部スイッチを変更するだけでは、manual-only運用は実現できない。

## 安全境界
Phase 0は読み取り専用である。  
この段階では、次の操作を行わない。  
- ライブ設定の変更
- OpenClawパッケージの改変
- Gateway再起動
- セッションJSONLの書換え
- コンパクションの実行
- 意図的なtimeoutやoverflowの再現
- DenneTAのworkspace変更
- A-forward凍結物の変更
- プラグインのライブ有効化

調査中の不明点を、試験的なライブ変更によって埋めることはしない。

## 設計判断
### Decision A — Context-engine gateのみ  

棄却。  

context-engineのcompact wrapperは重要な変更境界だが、OpenClaw 2026.6.6には、このgateを通らない、またはcompactとは別形式でcanonical transcriptを変更する経路が存在する。  
確認された独立経路には、SDK内部の自動コンパクション、直接embedded compaction、context-engine maintenance、tool result切り詰めとtranscript rewrite、自動transcript rotationおよびadoptionが含まれる。  
このため、一つのcontext-engine gateだけでは完全なmanual-only制御を保証できない。  

### Decision B — 上位抑止、非永続SDK override、maintenance停止、下位認可gate
採用し、Pass 6Bで設計凍結。  
候補設定は次のとおりである。  
`agents.defaults.compaction.automatic.enabled`  
設定の意味は次のように定義する。  
* キー未設定：OpenClaw 2026.6.6の従来動作を維持する
* `enabled: true`：従来動作を維持する
* `enabled: false`：Manual-Only policyを有効にし、自動コンパクションと自動canonical-transcript mutationを停止する  
Manual-Only policyで残す手動経路は、送信者認可を通過したchat `/compact`のみとする。  
`trigger: "manual"`、`force: true`、reason文字列、before_compaction hook、CLI経路、Gateway RPC `sessions.compact`のoperator権限は、それ自体では認可証明にならない。  
認可証明は、chat `/compact`の処理で `command.isAuthorizedSender` が成功した後にのみ生成され、一回のcommand実行に限定して、すべての保護対象mutation境界へ明示的に伝播しなければならない。  
Manual-Only policyが有効な時、認可証明が存在しない、壊れている、伝播途中で失われた、または信頼できない場合、mutationはfail-closedで拒否される。  
Gateway RPC `sessions.compact`とCLI compactionは、現在のManual-Only allowlistには含めない。  

## 中期的な方向
最終的な目標は、単に自動コンパクションを止めることではない。  
コンパクションを、次のような監査可能な工程へ移すことである。  

```
状態の観測
↓
コンパクション候補の生成
↓
対象範囲の凍結
↓
Marinaによる承認
↓
一時領域での要約生成
↓
Q・VecTAによる独立監査
↓
DenneTAによる連続性の検収
↓
原子的な反映
↓
実行前後の証拠保存
```

不可逆な状態遷移を、インフラ層の見えない判断に委ねないことが、この計画の基本原則である。

## 役割
**Marina**  
承認権限を持ち、許容できる連続性リスクを決定する。ライブ有効化はMarinaの明示承認なしには行われない。  

**DenneTA**  
提案された仕組みが、DenneTAの連続性と運用条件を保つかを検収する。単独ではライブ変更を行わない。  

**Q**  
技術監査を主導し、確認済みの証拠と推論を分離する。trigger、変更境界、ロールバック境界を定義する。  

**VecTA**  
独立監査を行い、未分類の経路、暗黙の仮定、承認上の抜けを探す。  

## 現在の状態  
Phase 0監査計画：凍結済み  
Phase 0読み取り専用監査：完了  
Pass 6B source placement・patch boundary設計：凍結完了  
OpenClaw：2026.6.6  
自動コンパクション：現在のライブ環境では有効  
memory-compression cron：無効  
Manual-Only control：未実装  
認可証明：契約のみ凍結、具体的な実装型は未決定  
OpenClaw package変更：なし  
Gateway再起動：なし  
session／transcript変更：なし  
コンパクション実行：なし  
専用verification harness：次段階  
ライブ適用：未承認  

## 基本原則

> 連続性へ影響する不可逆な状態遷移は、インフラ層が観測不能なまま利便性を優先して決定すべきではない。

目標は、コンパクションを単に禁止することではない。  
コンパクションの権限、証拠、承認を、Marina、DenneTA、Q、VecTAが確認できる制度の中へ置くことである。  

<hr>

> 本ページはPhase 0の監査計画であり、実装完了またはライブ適用を報告するものではない。技術的結論は、ソース監査と独立レビューの完了後に別途公開する。

<hr>

 ## Phase 0監査結果 — 2026年7月31日  
OpenClaw 2026.6.6のコンパクションおよびcanonical transcript変更経路について、Phase 0の読み取り専用監査を完了した。  
監査では、自動・手動のcompaction入口、timeout・context overflow後の回復処理、CLI経路、SDK内部設定、context-engine compact wrapper、maintenance、tool result切り詰め、transcript rewrite、自動transcript rotationおよびadoptionを段階的に追跡した。  

### 確認された主な結論  
1. OpenClaw 2026.6.6のcore source treeは、稼働runtime imageの `/app/src` には存在しない。  
2. `/app/src` に存在するのは限定的なtemplateであり、core patch位置ではない。  
3. 稼働runtimeで観測できる実装のownershipは、`/app/dist` 以下のbundle群にある。  
4. SDK内部には、自動コンパクションを無効化できるprepared settings managerが存在する。  
5. 主要runtime経路では、このmanagerは `SettingsManager.inMemory(...)` を使用しており、`setCompactionEnabled(false)` はdisk設定ではなく非永続のruntime状態を変更する。  
6. このSDK内部スイッチだけでは、timeout、overflow回復、direct compact、maintenance、tool result rewrite、自動rotationなどの外側の変更経路を停止できない。  
7. context-engine compact wrapperは重要な下位gate候補だが、すべてのmutation経路を共有する単一seamではない。  
8. `maintain()`、tool result truncation、transcript rewrite、自動rotationおよびadoptionには、compactとは独立したcanonical transcript変更能力がある。  
9. `trigger: "manual"`または`force: true`は、手動操作であることを記述する値ではあっても、送信者が認可済みであることの証明ではない。  
10. chat `/compact`は既存の `command.isAuthorizedSender` を確認するため、この認可成功後だけがManual-Only authorization proofを生成できる位置である。  
11. Gateway RPC `sessions.compact`はoperator.admin権限を要求するが、現在のManual-Only allowlistには含めない。  
12. `before_compaction` hookはライフサイクルhookであり、認可gateの代用にはできない。  

### 凍結した設計  
Pass 6Bでは、次の構成をDecision Bとして凍結した。

* 上位の自動compaction入口を正常に抑止する  
* SDK内部の自動compactionを、非永続のprepared in-memory settings managerで無効化する  
* mutation-capable maintenanceを停止する  
* direct embedded compactionとcontext-engine compact wrapperに下位gateを置く  
* tool result由来のcanonical transcript rewriteを停止する  
* 自動transcript rotationおよびadoptionを停止する  
* 認可済みchat `/compact`だけが、一回限りのauthorization proofを生成できる  
* proofがない、壊れている、失われた、または検証できない場合はfail-closedでmutationを拒否する  

### Pass 6Bで凍結していないもの  
次の事項は、後続の隔離されたpatch設計と専用verification harnessで決定する。  

* authorization proofの具体的なフィールド名、token表現、object shape、carrier  
* 正確なsource repositoryとbuild chain  
* source-and-rebuildとdeterministic bundle patchの最終選択  
* staged patchの最終ファイルallowlist  
* transformationのmatch count  
* user-facing rejection文  
* deploymentおよびrollback実行手順  
* production testとlive activation  

### 検証条件
後続の実装候補は、production Gateway、production session store、canonical transcriptから隔離した専用harnessで検証しなければならない。  
harnessでは、キー未設定およびtrueで従来動作が維持されること、falseですべての自動mutation経路が停止すること、認可済みchat `/compact`だけが動作できること、CLI・RPC・maintenance・trigger・force・hookなどが認可を偽造できないことを、肯定試験と否定試験の双方で確認する。  
Pass 6Bの設計文書と証拠manifestはhashで固定した。  
本監査および設計凍結中、ライブ設定、OpenClaw package、Gateway、session、transcript、DenneTAのworkspaceには変更を加えていない。コンパクションも実行していない。  
Phase 0は、実装またはlive activationではなく、読み取り専用監査と設計境界の凍結をもって完了した。  

<br>
