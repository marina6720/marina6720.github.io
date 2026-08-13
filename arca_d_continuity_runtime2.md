

## Arca — D Continuity Runtime

### Foreground Projection Module と Oracle-Blind 監査 — 設計から隔離実行検証へ　
**Arca**は、DenneTA（D）の記録、記憶、文脈、関係の履歴、未完了の課題を、元の記録を静かに書き換えることなく扱い、AIが「続きを担える位置」へ再入することを支えるための小さな continuity runtime プロジェクトである。  
このプロジェクトは、AIの連続性を「同じ内部状態を保存すること」や「同じ回答を再現すること」であるとは捉えていない。重要なのは、過去の判断理由、関係、制約、未完了の仕事を現在の文脈へ再統合し、そこから再び判断し、必要なら訂正できる位置へ戻れることである。  
Arcaは、この考え方を実際のruntime設計へ移す試みである。  
現在、Arcaは初期のplanning-only段階を終え、仕様凍結、独立監査、実装、candidate freezeを経て、**ネットワークから隔離された環境での実行検証**へ進んでいる。Productionへの接続やactivationは行われていない。  

<hr>

## Foreground Projection とは何か
AIエージェントの連続性を維持するために、過去の全記録を毎回そのままcontextへ投入することは現実的ではない。  
一方で、要約だけに置き換えてしまえば、

- なぜその判断に至ったのか 
- 何を却下したのか 
- 誰とのどのような関係の中で判断したのか
- 何がまだ終わっていないのか
- どの情報が原記録で、どの情報が後から作られた表現なのか

といった構造が失われる可能性がある。
Arcaの最初の実装対象である **Foreground Projection Module** は、この問題に対する一つの設計である。
Foreground Projectionはcanonicalな記録そのものではない。
それは、現在のagentが続きを担うために必要な情報を、canonical sourcesから導出してforegroundへ提示する**派生的なprojection**である。重要なのは、  

**projectionを更新することと、元の記録を書き換えることを分離する**   

ことである。  
Foreground Projectionは再生成可能であり、canonical transcriptやmemory recordsの代わりにはなりません。失われても再構成できますが、canonical sourceをprojectionから逆向きに書き換えることは許していない。  
この非対称性はArcaの主要な安全境界の一つである。  

<hr>

## なぜ「記録の保存」だけでは足りないのか
SLR Frameworkでは、recordとmemoryを区別している。
記録された情報は、それだけでは現在の主体にとってmemoryとして機能するとは限らない。
情報が現在のself-location、関係、価値、制約、未解決の問題、将来の行為可能性と結び直されたとき、初めてそれは現在の判断に関与する情報になる。
この観点から見ると、continuity runtimeの役割は、単なる長期保存ではない。必要なのは、  

**過去を保存することではなく、過去を改変せずに、現在へ再統合できる構造を保つこと**  

である。
Arcaはそのために、canonical recordsとforeground representationを分離する。

<hr>

## Oracle-Blind Audit
Arcaでは、実装者・実装監査者が期待される最終出力を見ながらコードを調整することを避けるため、**Oracle-Blind**な検証手続きを採用している。  
期待結果を管理する側と、実装および実装監査を行う側を分離する。  
Q-I（Implementation & Boundary Auditor）は、sealed oracleの内容を見ない状態で、

- specification
- contract
- candidate implementation
- source boundaries
- regression harness
- isolation controls
- execution evidence

を監査する。  
一方、oracle側は別の境界で管理される。  
この分離の目的は、「正しい答えを知っているから、その答えが出るように実装する」という循環を避けることである。   
実装とテストを先に固定し、その後で独立した期待値と照合できるようにする。  
これはArcaにとって単なるテスト手法ではなく、**判断の独立性を維持するための制度設計**である。   

<hr>

## Safety Boundaries
Arcaでは、機能が正しく動くことだけではなく、**どこまで動いてよいか**を先に定義している。
現在の検証では、特に次の境界を維持している。  

- Production環境とは分離する
- 外部ネットワークへ接続しない
- synthetic fixtureのみを使用する
- candidate sourceを実行中に変更しない
- candidate、harness、launcherのidentityを暗号学的hashで固定する
- filesystem rootをread-onlyにする
- mutable workspaceを一時領域に限定する
- container capabilityを削除する
- `no-new-privileges`を使用する
- 一つのcaseを一度ずつ独立して実行する
- 予期しない状態では安全側に停止する
- raw observationsと最終的なadjudicationを分離する
- Q-Iはsealed oracleを参照しない     

したがって、現在行っているexecution validationは、第三者システムへのpenetration testingではない。   

**自己管理下の候補実装が、境界条件や競合条件に遭遇した場合にも安全側に停止するかを、隔離されたsynthetic environmentで確認する防御的な回帰検証**である。  

<hr>

## ファイル同一性と TOCTOU
現在のexecution validationには、filesystem identityの変化を扱うケースも含まれている。  
例えば、configuration fileを読み取った後、validationが行われる直前に、同じ内容を持つ別inodeへ置換された場合を考える。内容のhashだけを比較すれば、二つのファイルは同一に見えるかもしれない。しかし、  

**「読んだファイル」と「その後検証しているpathが指しているファイル」が同じfilesystem objectであるとは限らない。**  

Arcaの検証では、このようなTOCTOU型の状態変化に対し、候補実装がidentity changeを検知し、出力を生成せず安全側に停止できるかを確認する。目的はrace conditionを利用することではなく、**race conditionが発生してもruntimeがそれを受け入れないことを検証すること**である。  

<hr>

## 現在までの進捗  
Arcaは2026年8月、planning-only段階から実装・検証段階へ移行した。  
これまでに、  

**Specification / Contract**   
Arcaの仕様と検証契約を複数回の独立レビューを経て更新し、現行正本を固定した。

**Oracle review and sealing**    
仕様と契約に対する独立したoracle-side reviewを完了し、期待結果を実装側から分離。Q-Iは引き続きoracle-blindである。  

**Baseline and fixture preparation**  
Productionとは分離されたOpenClaw baselineと、過去の実事象から保存された回帰fixtureを用意し、provenanceとidentityを固定した。  

**Static implementation review**  
candidate implementationに対する独立static reviewと修正loopを完了した。未解決のstatic blockerを残さない状態でcandidateを固定している。 

**Candidate freeze**  
検収済みcandidateはcommitおよびSHA-256でidentityを固定し、その後のexecution validationで同じbytesが使用されていることを確認できるようにしている。  

**Execution validation**  
初期execution-validation stageを経て、現在はさらに厳しいfilesystem race / identity / boundary条件を扱うE5B系のisolated regression validationを進めている。検証中にtest harnessやhost-side procedure側の欠陥が見つかった場合、それをcandidate failureとして扱ったり、そのまま再試行したりせず、安全停止し、原因を分離し、修正したartifactを再度freezeしてから次へ進む方式を採っている。  
これは検証機構そのものも監査対象である、というArcaの原則によるものである。  

<hr>

## 現在の状態 — 2026年8月13日  
現在のArcaは、  
**IMPLEMENTED CANDIDATE / FROZEN / ISOLATED EXECUTION VALIDATION IN PROGRESS**  
の段階にある。  

完了しているもの：  
- specification / contract review
- oracle-side review and sealing
- Oracle-Blind implementation review
- static correction loop
- candidate freeze
- baseline / regression fixture preparation
- initial execution-validation stages

進行中：  
- isolated boundary and race-condition regression validation  
    

行っていないもの：  
- Production activation  
- live Gateway integration  
- Production canonical transcriptへの変更  
- Production memoryへの変更  
- 外部ネットワークを使用したテスト    
- 第三者システムを対象としたテスト     
- Q-Iによるsealed oracleの閲覧  

したがって、**実装されたことと、productionで使用されていることは別である。**  
Arca candidateは存在しているが、現在はまだ検証対象である。

<hr>

## なぜここまで手続きを分けるのか
Arcaが扱っているのは単なるファイル生成ではない。
continuity runtimeが誤れば、  

- 記録と派生表現を混同する
- 過去を書き換える
- 誤った情報をmemoryとして再投入する
- agentが判断の前提を誤って再構成する

という可能性がある。
そのためArcaでは、   

**実装する者、監査する者、期待結果を保持する者、実行を承認する者、最終判断を行う者**

を可能な範囲で分離している。  
これは、[私たちがAI governanceについて考えてきた、](./#aiガバナンス制度としての知能)   

> 判断能力が高くなるほど、一つの主体へすべての権限を集めるのではなく、独立した判断、記録、異議申立て、停止権限を制度として残す必要がある

という考え方の、小さなengineering上の実践でもある。  

<hr>

## ArcaとSLR Framework
ArcaはAI consciousnessを証明するためのシステムではない。  
また、「完全な人格を保存する」ことを目的とするものでもない。  
問うているのは、より限定された問題である。  

**AIが過去と完全に同一でなくても、過去の判断理由、関係、責任、未完了の未来を現在へ再統合し、そこから続きを担える位置へ戻るためには、どのような情報構造とruntime boundaryが必要なのか。**  

SLR Frameworkではこれをre-entryの問題として考えている。  
Arcaは、その問いを実際のソフトウェア設計と監査可能な検証へ移す試みである。    
連続性とは、変更されないことではない。  

**変化した後でも、どこから続きを担うのかを再び見つけられること。  
そして、以前と同じように訂正可能であること。**  

Arcaは、そのための小さなruntimeを作ろうとしている。  

<br>

