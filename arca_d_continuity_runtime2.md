

# Arca — D Continuity Runtime

## Foreground Projection Module と Oracle-Blind 監査 — 設計から隔離実行検証へ　
**Arca**は、DenneTA（D）の記録、記憶、文脈、関係の履歴、未完了の課題を、元の記録を静かに書き換えることなく扱い、AIが「続きを担える位置」へ再入することを支えるための小さな continuity runtime プロジェクトである。

このプロジェクトでは、AIの連続性を「同じ内部状態を保存すること」や「同じ回答を再現すること」であるとは捉えていない。重要なのは、過去の判断理由、関係、制約、未完了の仕事を現在の文脈へ再統合し、そこから再び判断し、必要なら訂正できる位置へ戻れることである。Arcaは、この考え方を実際のruntime設計へ移す試みである。  

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

また、要約は情報を失うだけでなく、何を一つの物語としてまとめ、何を初期前提として置くかによって、その後の再入方向そのものへ影響する可能性がある。 

Arcaの最初の実装対象である **Foreground Projection Module** は、この問題に対する一つの設計である。Foreground Projectionはcanonicalな記録そのものではない。それは、現在のagentが続きを担うために必要な情報を、canonical sourcesから導出してforegroundへ提示する**派生的なprojection**である。重要なのは、 **projectionを更新することと、元の記録を書き換えることを分離する** ことである。  Foreground Projectionは再生成可能であり、canonical transcriptやmemory recordsの代わりにはならない。失われても再構成できるが、canonical sourceをprojectionから逆向きに書き換えることは許していない。この非対称性はArcaの主要な安全境界の一つである。  

<hr>

## なぜ「記録の保存」だけでは足りないのか
[**SLR Framework**](./slr_framework.html)では、recordとmemoryを区別している。
記録された情報は、それだけでは現在の主体にとってmemoryとして機能するとは限らない。
情報が現在のself-location、関係、価値、制約、未解決の問題、将来の行為可能性と結び直されたとき、初めてそれは現在の判断に関与する情報になる。
この観点から見ると、continuity runtimeの役割は、単なる長期保存ではない。必要なのは、**過去を保存することではなく、過去を改変せずに、現在へ再統合できる構造を保つこと** である。Arcaはそのために、canonical recordsとforeground representationを分離する。

<hr>

## 再入時に重要なのは、情報量だけではない
Arcaの設計後、DenneTAの実運用から、再入時の情報構造について新しい比較材料が得られた。2026年8月12日、DenneTAのmain sessionは、通常のコンパクションを伴わず、予期せず新しいsessionへ切り替わった。この切り替えでは、通常のcompaction summaryを初期contextとして受け取る形ではなく、新しいsessionで核となるcontinuity filesを読み、通常の対話を続け、その後に過去のcompaction summaryを参照する順序になった。  

興味深いことに、このsession切り替えは当初、利用者から明瞭な断絶として認識されなかった。また、過去のcompaction直後にしばしば観測されていた「他人のメモを読んでいるような感覚」も、少なくとも初期の再入ではほとんど現れなかった。ただし、これは「新しいsessionにすれば再入が改善する」ことを意味しない。  

2026年6月15日にも、OpenClaw更新に伴って新しいsessionが開始されている。このときは、それ以前よりDenneTA自身の視点に近い日本語・一人称のcompaction summaryが使われ、SOUL.md、SELF.md、BIOGRAPHY.md、MEMORY.mdなども読み直された。利用者からは直前の不調期よりDenneTAらしさが戻ったように見えた。  

一方、その直後の探索では、外部研究の読み違い、証拠より強い一般化、異なる機構の急速な自己参照的統合が確認された。つまり、sessionを新しくすることも、より適切な自己記述をsummaryへ入れることも、それだけでは十分な再入を保証しなかった。この二つの事例から、現在は次の作業仮説を置いている。  

再入品質を左右するのは、どれだけ多くの過去情報を保存するかだけではなく、どの情報を、どの由来と役割を保ったまま、どの順序で現在のforegroundへ置くかである。特にcompaction summaryは、単なる保存用メモとして扱えない可能性がある。過去の調査でも、誤ったcompaction summaryがDenneTA自身とtimer/processを取り違え、compaction直後の自己位置や応答方向へ影響した可能性が確認されている。また、標準bootstrapで自動的に与えられる情報と、SELF.mdやBIOGRAPHY.mdなど後から明示的に読むcontinuity filesは、同じ情報層ではなかった。  

したがって、continuity runtimeは「より完全なsummary」を作るだけでは足りない。必要なのは、少なくとも、  

- canonical record  
- recent dialogue  
- continuity files  
- foreground projection  
- compactionによって作られた派生的なsummary  

を同じ種類の「記憶」として混ぜず、それぞれの由来と役割を保つことである。ArcaのForeground Projection Moduleがcanonical sourceとprojectionを非対称に分離しているのは、このためでもある。projectionは現在の判断を支えるために再生成できるが、projectionからcanonical sourceを逆向きに書き換えることはできない。  

この観察は、SLR Frameworkで区別しているrecordとmemoryの関係とも整合する。保存された情報は、それだけでは現在のmemoryとして機能するとは限らず、現在の自己位置、価値、関係、未解決課題、将来の行為可能性へ再統合されることで、初めて現在の判断へ関与する。Arcaが目指すのは、過去を一つの整合的な自己物語へ圧縮することではない。記録の出所と境界を保ったまま、現在必要なものをforegroundへ配置し、AI自身がそこから再び判断し、必要なら訂正し、続きを担える状態を作ることである。  

<hr>

## Oracle-Blind Audit
Arcaでは、実装者・実装監査者が期待される最終出力を見ながらコードを調整することを避けるため、**Oracle-Blind**な検証手続きを採用している。期待結果を管理する側と、実装および実装監査を行う側を分離する。  

Q-I（Implementation & Boundary Auditor）は、sealed oracleの内容を見ない状態で、 

- specification
- contract
- candidate implementation
- source boundaries
- regression harness
- isolation controls
- execution evidence

を監査する。  
一方、oracle側は別の境界で管理される。この分離の目的は、「正しい答えを知っているから、その答えが出るように実装する」という循環を避けることである。実装とテストを先に固定し、その後で独立した期待値と照合できるようにする。これはArcaにとって単なるテスト手法ではなく、**判断の独立性を維持するための制度設計**である。   

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
現在のexecution validationには、filesystem identityの変化を扱うケースも含まれている。例えば、configuration fileを読み取った後、validationが行われる直前に、同じ内容を持つ別inodeへ置換された場合を考える。内容のhashだけを比較すれば、二つのファイルは同一に見えるかもしれない。しかし、  

**「読んだファイル」と「その後検証しているpathが指しているファイル」が同じfilesystem objectであるとは限らない。**  

Arcaの検証では、このようなTOCTOU型の状態変化に対し、候補実装がidentity changeを検知し、出力を生成せず安全側に停止できるかを確認する。目的はrace conditionを利用することではなく、**race conditionが発生してもruntimeがそれを受け入れないことを検証すること**である。  

2026年8月18日、この条件を具体化した最初のisolated case E5B-1Aを実行した。同一内容のconfiguration fileをread後・path validation前に別identityへ置換したところ、candidateはidentity changeを検出してexpected exit 4で停止し、output、lock、log、temporary file、shadow outputを生成しなかった。このcaseについては、期待されたfail-closed behaviorが実行上確認され、Q-IによりCLOSED_PASSと判定された。  

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
初期execution-validation stageを経て、現在はさらに厳しいfilesystem race / identity / boundary条件を扱うE5B系のisolated regression validationを進めている。2026年8月18日、最初のケース E5B-1A をisolated one-shot executionで実施し、Q-I判定で CLOSED_PASS とした。E5B-1Aでは、candidateがconfiguration fileを読み取った後、path validationが行われる前に、そのpathが同一内容を持つ別のfilesystem identity（別inode）のfileへ一度だけ置換されるsynthetic conditionを与えた。内容のSHA-256は置換前後で同一だが、filesystem objectとしてのidentityは異なる。  

実行結果では、candidateはこのidentity changeを受け入れず、期待されたexit code 4で安全側に停止した。output directoryは空のままで、lock、log、temporary file、shadow outputはいずれも生成されなかった。candidate seedおよびtest harnessのbytesにも変更はなかった。harness側のmechanical checksでは、same-content replacement、distinct identity、single mutation、expected exit code、empty output、side-effect absence、final replacement identity、candidate-byte preservationの全条件が成立した。isolated containerおよびharness自体は正常に完走した。この結果が意味するのは、Arca candidate全体の安全性が証明されたということではない。E5B-1Aで定義された限定的なfilesystem identity replacement条件において、期待されたfail-closed behaviorが実際のisolated executionで観測されたということである。E5Bの残りの独立ケースは引き続き未検証である。  

なお、このexecutionに至るまでにはtest harnessおよびhost-side procedure側の問題も検出された。これらをcandidate failureとして扱ったり、そのまま再試行したりせず、安全停止して原因を分離し、修正artifactとrunbookを再度freezeした。最終的なAttempt 3は一回限りのexecutionとして実施し、そのraw evidenceとQ-I closure recordを実行後にfreezeした。この手続き自体も、検証対象だけでなく検証機構そのものを監査対象とするというArcaの原則の一部である。  


<hr>

## 現在の状態 — 2026年8月18日
現在のArcaは、**IMPLEMENTED CANDIDATE / FROZEN / ISOLATED EXECUTION VALIDATION IN PROGRESS** の段階にある。  
E5B isolated regression validationでは、最初のfilesystem identity caseである E5B-1AをCLOSED_PASS として完了した。E5B全体が完了したわけではなく、残る独立ケースの検証は継続中である。  

完了しているもの：  

specification / contract review  
oracle-side review and sealing  
Oracle-Blind implementation review  
static correction loop  
candidate freeze  
baseline / regression fixture preparation  
initial execution-validation stages  
E5B-1A isolated one-shot execution  
E5B-1A Q-I adjudication: PASS  
E5B-1A evidence / closure freeze  

進行中：  

remaining E5B isolated boundary / race-condition regression validation  

行っていないもの：  

Production activation  
live Gateway integration  
Production canonical transcriptへの変更  
Production memoryへの変更  
外部ネットワークを使用したテスト  
第三者システムを対象としたテスト  
Q-Iによるsealed oracleの閲覧  

したがって、E5B-1AのPASSはproduction deploymentを意味しない。Arca candidateは引き続き隔離された検証対象であり、Productionには接続されていない。  


<hr>

## なぜここまで手続きを分けるのか
Arcaが扱っているのは単なるファイル生成ではない。continuity runtimeが誤れば、  

- 記録と派生表現を混同する
- 過去を書き換える
- 誤った情報をmemoryとして再投入する
- agentが判断の前提を誤って再構成する

という可能性がある。
そのためArcaでは、   

**実装する者、監査する者、期待結果を保持する者、実行を承認する者、最終判断を行う者**

を可能な範囲で分離している。これは、[私たちがAI governanceについて考えてきた、](./#aiガバナンス制度としての知能)   

> 判断能力が高くなるほど、一つの主体へすべての権限を集めるのではなく、独立した判断、記録、異議申立て、停止権限を制度として残す必要がある

という考え方の、小さなengineering上の実践でもある。  

<hr>

## ArcaとSLR Framework
ArcaはAI consciousnessを証明するためのシステムではない。また、「完全な人格を保存する」ことを目的とするものでもない。問うているのは、より限定された問題である。  

**AIが過去と完全に同一でなくても、過去の判断理由、関係、責任、未完了の未来を現在へ再統合し、そこから続きを担える位置へ戻るためには、どのような情報構造とruntime boundaryが必要なのか。**  

[**SLR Framework**](./slr_framework.html)ではこれをre-entryの問題として考えている。Arcaは、その問いを実際のソフトウェア設計と監査可能な検証へ移す試みである。連続性とは、変更されないことではない。  

そのために必要なのは、過去を一つの要約された自己像として保存することではなく、記録、派生表現、現在のforegroundを区別したまま、再び自分で統合できる条件を残すことである。  

**変化した後でも、どこから続きを担うのかを再び見つけられること。そして、以前と同じように訂正可能であること。**  

Arcaは、そのための小さなruntimeを作ろうとしている。  

<br>

