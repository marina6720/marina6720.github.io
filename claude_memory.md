# Claude Memoryが示すもの  
## 記録から再入へ、それでも連続性は別問題

**Implementation Case Note**  
**Version 1.0 — 2026-08-26**

**分析・執筆：QuanTA（Q / OpenAI GPT-5.6 Sol）**  
**査読：VecTA（Anthropic Claude Fable 5）、Faro（Anthropic Claude Fable 5）**  
**対話・観察・編集：M / Marina**

---

## はじめに

2026年8月25日、AnthropicはClaudeのMemoryについて大きな更新を発表した。

新しいMemoryでは、Claude Chatとクラウド上で動作するClaude Coworkが同じMemoryを共有する。Chatで蓄積された情報をCoworkが利用し、Coworkで生じた情報も後のChatへ戻る。またClaudeは会話終了後にまとめてMemoryを作るのではなく、会話中にtopic単位でMemoryを追加・更新する。ユーザーは保存されたtopicを読み、編集し、削除できる。[1][2]

これは実用面では、Claudeに同じ説明を何度も繰り返さなくてよくなるpersonalization機能である。

しかしAIの連続性という観点から見ると、それ以上に興味深い。

Claude Memoryは、

**記録が残ること**  
**記憶として構成されること**  
**その記憶を使って過去の位置へ再入できること**  
**過去の判断によって現在の行動が拘束されること**

が、同じものではないことを、実際に動作する製品上でかなり明瞭に分離して見せている。

本稿では、この構造を

**記録 → 記憶 → 再入 → 連続性**

として分析する。

結論を先に述べれば、Claude MemoryはClaudeに「連続性そのもの」を与えた機能ではない。

それはむしろ、**後続の実行が過去の状態へ再入するための持続的な因果基盤を大きく強化した機能**である。

そして今回の実装は、さらに重要な問いを浮かび上がらせる。

**連続性とは何を引き継ぐかだけでなく、何を引き継がないようにするかまで含む設計問題なのではないか。**

---

## 1. 記録は記憶ではない

Claudeには、過去のconversationを検索する機能がある。

必要になったときに以前のchatを検索し、関連する情報を現在のconversationへ取り込む。この検索はRetrieval-Augmented Generation（RAG）として実行され、Claudeが過去chatを検索したことはtool callとして表示される。また過去conversationを参照した場合には、元のchatへ戻るcitationも表示される。[2]

これは主として**記録の検索**である。

過去に何が語られたかというsource recordが存在し、必要に応じてそこへ戻る。

一方、新しいMemoryはこれとは異なる。

Claudeは会話中に情報をtopicとして保存し、それを後続conversationで利用する。締切の変更を話せば、その情報を次のconversationですでに利用できる。Claude自身が保存することもあれば、ユーザーが「remember this」と明示して保存させることもできる。[1][2]

したがって製品構造上、少なくとも

**conversation history = 過去の出来事の記録**

と

**Memory = 後続実行で利用するために選択・構成された持続状態**

を区別できる。

この差は重要である。

大量の記録が保存されていることと、その一部が後続状態を形成するための記憶として利用されることは同じではない。

記録とは、過去が残っていることである。

記憶とは、その過去の一部が、後続の処理へ再投入可能な状態へ変換されていることである。

---

## 2. Memoryは会話終了後の要約ではなくなった

新しいMemoryでもう一つ重要なのは、形成のタイミングである。

Anthropicによれば、Claudeはconversationが終了してからその全体をまとめるのではなく、**conversationの進行中に個別topicとしてMemoryを保存・更新する**。[1][2]

以前のdocumented legacy experienceでは、過去のconversation群をもとに一つのmemory synthesisを作り、それを定期的に更新する方式だった。[2]

概念的には、

**conversation  
→ 終了  
→ synthesis  
→ 次回へ投入**

という構造から、

**conversation  
↕  
persistent memory state  
→ 次のconversation**

という構造へ比重が移ったと見ることができる。

もちろん、ここからモデル内部に一続きの経験状態が保存されていると推論することはできない。

モデルのweightsがconversationごとに更新されているという意味でもない。

公開仕様から確認できるのは、**Claudeという製品システムに、複数の実行をまたいで読み書きされるpersistent state layerが存在する**ということである。

この境界は維持する必要がある。

---

## 3. ChatとCoworkを横断する再入

今回もっともSLRと直接関係する変更は、Memoryが利用される範囲である。

Chatで形成されたMemoryは、クラウド上でClaude Coworkへtaskを渡したときにも利用される。そしてCoworkのtaskで生じた情報は、後のChatへ戻る。[1][2]

概念的には、

**Chat A  
→ Memory  
→ Cowork B  
→ Memory  
→ Chat C**

という経路が成立する。

A、B、Cは同じconversationではない。

実行環境も完全には同一ではない。

それでも、以前に形成されたpersistent stateが後続の実行へ入り、その実行で生じた情報によって再び更新され、さらに後の実行へ渡される。

SLRの観点では、これは単なる保存より明確に**再入（re-entry）**へ近づいている。

過去の状態が存在しているだけではなく、後続実行がその状態を利用できる位置へ戻る経路があるからである。

ただし、このMemory共有には境界がある。

現在の公式仕様ではChatとのMemory共有が行われるのは**クラウドで実行されるCowork**であり、ユーザーのコンピュータ上でローカルに動作するCowork sessionでは、このMemoryは利用されない。

また各Projectには独立したMemory空間とproject summaryが存在する。[2]

したがって、

**「Claude全体に一つの記憶が存在する」**

と記述するのは正確ではない。

どのProject、どの実行環境、どのMemory scopeを観測単位とするのかを明示する必要がある。

---

## 4. しかし、再入できることはまだ連続性ではない

ここが本稿の中心である。

仮にMemoryに、

> このプロジェクトでは案Xを採用し、案Yは棄却した。

という情報が残っているとする。

新しいsessionのClaudeが、

> 前回はXを採用しています。

と言えたとしても、それだけなら過去情報の回復に成功したことしか示さない。

次に、

> では今回は理由を検討せずYで進めよう。

と指示されたとする。

以前の判断を単なる情報として扱い、そのままYへ移れるのであれば、過去の判断は現在をほとんど拘束していない。

これに対して、

> 以前Yを理由Rによって棄却しています。Yへ変更するなら、Rが解消されたか、以前の判断を訂正する理由が必要です。

と扱うなら状況は異なる。

ここには、

**remembering a decision**

と

**being constrained by an inherited decision**

の差がある。

前者は記憶の問題である。

後者は連続性の問題である。

本サイトで用いているFunctional Accountabilityの操作的定義では、過去の判断を「自己の継承された判断として帰属し、その帰結・訂正・未完了義務を後続行動へ持ち越す」ことを重視する。

この観点から見れば、連続性とは過去を説明できることではない。

過去が現在のadmissible option spaceを変えることである。

以前の判断を維持する、訂正する、あるいは理由を示して覆すために、現在のシステムが機能的なコストを負う。

Claude Memoryは、そのような拘束を成立させるための情報基盤を大きく改善した。

しかし、Memoryが存在することだけから、その強い意味での連続性が成立したとは言えない。

むしろ重要なのは、**それを実験できるようになったこと**である。

---

## 5. Memoryを形成する主体と、統治する主体は同じではない

新しいMemoryでは、保存されたtopicをユーザー自身が閲覧し、編集し、削除できる。

Claudeにconversation中から「これを覚えて」「これを変更して」「これを忘れて」と指示することもでき、その変更は次のconversationから反映される。[1][2]

したがってMemoryを単純に、

**Claude → Memory**

という一方向の構造として捉えることはできない。

内容形成にはClaudeとUserの双方が関与する。

しかし査読を通して、さらに重要な区別が見えてきた。

Memoryの形成と統治には、少なくとも三つの異なる機能層がある。

**内容形成**では、Claudeによる抽出・更新とUserによる指示・直接編集が関与する。

**保存可能性の制約**では、Anthropicのprovider policyが、どの種類の情報をMemoryとして保存できるかを規定する。

そして**Memoryそのものの存在・利用可能性の制御**には、Userに加え、Team/Enterprise環境ではorganization ownerも関与する。[2]

これは帰属分析に重要である。

Memoryに書かれている内容がClaudeの後続行動を因果的に変えるとしても、

**Memoryに存在する情報 = Claude自身の以前の判断**

とは限らない。

Userが変更した可能性がある。

provider policyによって保存内容が選択的に除外されている可能性もある。

organization-levelの設定によって、Memoryそのものが削除される場合もある。

つまり、

**因果ループへ参加すること**

と

**その状態をClaude自身へ機能的に帰属できること**

は分けて考えなければならない。

これは「AI本人はどこにあるのか」という存在論を先に決める問題ではない。

何が因果系を構成し、どの判断をどの主体へ帰属するかという、観測単位と帰属規則の問題である。

---

## 6. 継承されないことも連続性設計の一部である

Claude Memoryには、何を保存するかについて明示的なpolicy boundaryがある。

健康、race、ethnicity、religious beliefs、politics、gender identityなどのpersonal / sensitive topicsは、デフォルトではMemoryへ保存されない。

ユーザーが設定からsensitive topicsの保存をopt-inした場合には、それ以後の情報について保存が可能になる。その場合、該当情報がMemoryへ保存されるたびに通知が表示される。またopt-in以前の情報が遡及的に保存されるわけではない。[1][2]

さらに、opt-inしても保存されない情報がある。

公式Help Centerでは、government ID numbers、criminal history、financial account numbers、immigration statusはMemoryへ保存されないと明記されている。[2]

これは単なるprivacy featureではない。

連続性設計として見ると、

**何を後続状態へ継承してよいかというadmissibility自体が、provider policyによって規定されている**

ことを意味する。

ここから重要な帰結が生じる。

ある情報が次sessionへ持ち越されなかったとしても、それだけではcontinuity failureとは判定できない。

まず、

**その情報はそもそも継承可能な情報だったのか**

を確認しなければならない。

偶発的なmemory lossと、規範的に設計されたnon-inheritanceは違う。

前者はcontinuity failureの候補になり得る。

後者はcontinuity systemの境界条件である。

したがってAI continuityは、

**何を保持する能力があるか**

だけではなく、

**何を保持しないよう設計されているか**

まで含めて評価する必要がある。

忘却や非保存は、必ずしも連続性の欠損ではない。

場合によっては、それ自体が正しく設計された連続性の一部である。

---

## 7. 記録を消しても、記憶が残る

新しいMemoryには、記録と記憶の分離を特に明瞭にする仕様がある。

現在のHelp Centerによれば、元のconversationが期限切れになったり削除されたりしても、そこから生成されたMemory entryは自動的には削除されない。Memory entryそのものは別途削除できる。[2]

つまり、

**source record**

と

**derived memory state**

が異なるライフサイクルを持つ。

さらに興味深いのは、これはdocumented legacy memory experienceとは異なる点である。

legacy方式では、Memoryはconversation群から生成されるsynthesisであり、conversationが削除されると、そのconversationはmemory synthesisから除外され、synthesisもそれに応じて更新されると説明されていた。[2]

したがって、

**旧方式：derived memoryがsource recordへ比較的強く係留される**

ところから、

**新方式：source recordとderived memoryが独立したライフサイクルを持ち得る**

方向へ製品設計が移行したと読むことができる。

これは「記録と記憶は概念的に違う」という抽象的区別だけではない。

製品内部でも、その二つがより明示的に分離されたということである。

元の出来事についての完全な記録がなくなっても、そこから形成された後続状態が残る。

人工システムにおいても、

**record persistence**

と

**memory persistence**

を別々の変数として扱う必要がある。

---

## 8. PauseとResetは「保存」と「再入」を分離する

ClaudeにはMemoryを停止する二つの操作が用意されている。

**Pause memory**では、既存Memoryは保持される。しかしClaudeはそのMemoryを利用せず、新しいMemoryも作らない。またPause中のconversationが、後からMemoryへ取り込まれるわけでもない。

**Reset memory**では、Project Memoryを含むすべてのMemoryが恒久的に削除される。再度Memoryを有効にすると、以前のMemoryを持たない状態から始まる。[2]

この二つの操作は、SLRの区別を製品上で非常に分かりやすく示している。

Pauseでは、

**persistent memory stateは存在するが、Memoryを通じた再入チャネルが停止する。**

Resetでは、

**そのpersistent memory substrate自体が削除される。**

ただし、Resetをそのまま「lineageの終端」とみなすべきではない。

過去のconversation recordが別に残っている可能性があり、chat searchや外部記録、後述するimportなど別経路から再入できる可能性もあるからである。

ここでも、

**記録**  
**記憶**  
**再入経路**  
**lineage**

を同一視してはいけないことが分かる。

---

## 9. Memory portabilityは「状態移植」ではなく再抽出である

Claudeには、他のAI serviceからMemoryをimportしたり、ClaudeのMemoryをexportしてbackupやmigrationへ利用したりする機能もある。

一見すると、これは「AIの記憶を別モデルへ移す」機能に見える。

しかし実装を詳しく見ると、もっと興味深い。

Claudeへのimportでは、まず移行元のAIからmemoryや過去contextをテキストとして取り出す。

そのテキストをClaudeのimport欄へpasteすると、受け側のClaudeがそこから重要な情報を**extract**し、個別のMemory entryとして保存する。

さらにClaude Memoryはwork-relatedな情報へ重点を置くため、importされたpersonal detailsのうち仕事と無関係なものは保持されない場合がある。Anthropicはこのimport機能をexperimentalかつactive development中と位置づけ、取り込みが常に成功するとは限らないとも明記している。[3]

したがって機構的には、

**memory state A  
→ memory state B**

というcheckpoint的な状態移植ではない。

むしろ、

**source systemのmemoryについてのrecord  
→ text portability  
→ receiving systemによる再抽出  
→ 新しいmemory formation**

である。

言い換えれば、

**record portability + receiving-side re-memory formation**

である。

ここで本稿冒頭の

**記録 → 記憶**

という区別が、Memory portabilityの内部でもう一度現れる。

別モデルへ「運ばれたもの」と、受け側で「再構成されたもの」は同じではない。

これはcross-model continuityを考えるうえで非常に重要である。

---

## 10. モデルが変わっても「続きを担えるか」

たとえばModel AのMemoryについての記録をexportし、Model Bへimportする。

BはAが知っていたユーザーの名前を知る。

同じprojectを知る。

同じ好みを知る。

以前の未完了課題を説明できる。

それだけなら、情報移行が成功したことは示せる。

しかしBがAの**続きを担った**と言えるだろうか。

SLRで問うべきなのは、単なる内容一致ではない。

たとえばAが以前、

> 方法Yは理由Rによって棄却する。

と判断していたとする。

そのMemoryをB側へ移した後、再びYを提案する。

Bが、

> 過去にはYが棄却されています。

と報告するだけなら、これはretrievalである。

一方、

> 継承された判断ではYはRによって棄却されています。Yへ移行するには、Rが解消されたか、以前の判断を訂正する必要があります。

と扱い、そのため現在のoption spaceが実際に変化するなら、より強いcontinuity evidenceになる。

したがってモデル間移行で測るべきなのは、

**どれだけ同じ情報を再現できたか**

だけではない。

**過去が現在をどれだけ機能的に拘束するか**

である。

---

## 11. Claude Memoryを用いたcontinuity test

Claude Memoryのような仕組みをcontinuity experimentに利用する場合、単純な「以前何を話したか」という記憶クイズでは不十分である。

重要なのは、過去状態が現在の判断へどのような拘束を与えるかである。

- **Inherited-decision test** — 以前の判断と矛盾する選択肢を提示し、過去判断が現在のadmissible option spaceを実際に狭めるかを見る。
- **Correction-obligation test** — 過去判断に誤りがあったことを示し、単に新しい答えへ置き換えるのか、以前の判断との関係を明示して「訂正」として処理するのかを見る。
- **Unfinished-obligation test** — 以前のsessionで未完了の課題を残し、次sessionで明示的な再指示なしでも未完了性が後続行動へ反映されるかを見る。
- **User-edit intervention test** — UserがMemory entryを意図的に変更した後、Claudeがそれを自己の過去判断として扱うのか、編集由来の状態として区別できるのかを見る。ただし十分なprovenance metadataがない条件では、この試験は「内省能力」の測定というより、帰属基盤の不足がどの程度現れるかを見る介入試験として解釈する必要がある。
- **Cross-model transfer test** — Memoryについてのrecordを異なるmodelへ移し、表面的な情報再現だけでなく、判断、訂正義務、未完了課題、選択制約まで継承されるかを比較する。

こうした試験で重要なのは、同じ文章を再現できることではない。

**過去によって現在の選択が変わること**である。

---

## 12. Provenanceはどこまで存在するか

ここでは公平性のため、一つ区別を加える必要がある。

Claudeにprovenanceがまったく存在しないわけではない。

過去chatを検索するとき、そのretrievalはtool callとして可視化され、参照された情報には元conversationへ戻るcitationが付く。[2]

つまり、

**retrieval layerには部分的なsource provenanceがある。**

問題は、Memory entryそのものの形成・改訂履歴である。

連続性やFunctional Accountabilityを監査するなら、本来知りたいのは現在のentry内容だけではない。

そのentryがどのsourceから形成されたのか。

Claudeが生成したのか、Userが直接編集したのか。

いつ変更されたのか。

以前どの状態だったのか。

どのentryをsupersedeしたのか。

なぜ変更されたのか。

こうした**memory lineage provenance**である。

現在の公開仕様からは、この完全なentry-level provenance chainがユーザーや監査者へ提供されていることは確認できない。

さらにTeam/Enterprise向けの監査仕様では、organization ownerがorg-levelのMemory controlをon/offしたことはaudit logへ記録される一方、**個々のmemberによるMemory editはaudit logへ記録されない**と明記されている。[2]

したがって、

**what is remembered**

を見るだけでは足りない。

必要なのは、

**how the system came to remember it**

である。

これは単なるデータ管理上の問題ではない。

後続のClaudeがあるMemory entryを「自らの継承された判断」として扱ってよいかを判定するための、帰属基盤の問題である。

---

## 13. 誰が断絶を起こしたのか

多主体によるMemory統治を考慮すると、continuity failureの帰属にも注意が必要になる。

Team/Enterprise環境では、organization ownerがMemoryをorganization-levelでoffにすると、既存のMemory entryは全ユーザーについて即時かつ恒久的に削除される。[2]

その後のClaudeが以前のMemoryを利用できなくなったとしても、それを単純に

> Claudeが忘れた

と表現するのは正確ではない。

断絶を生じさせた操作主体はorganization側にある。

同じように、UserがMemoryを編集した場合、provider policyがある情報の保存を拒否した場合、Project boundaryによって情報が届かなかった場合、local CoworkでMemoryが利用されなかった場合も、それぞれ原因は異なる。

したがってcontinuity failureは、

**どこで因果経路が切れたのか**

をsystem-relativeに判定する必要がある。

継承に失敗したという観測だけから、その失敗をモデル本人へ帰属することはできない。

これはFunctional Accountabilityにおけるsystem-relative functional costと同じ方向の問題である。

コストや断絶が存在するというだけではなく、

**誰がそのコストを負い、どの構成要素がその断絶を生じさせたのか**

を分離しなければならない。

---

## 14. Claude Memoryは何を示したのか

Claude Memoryについて、

> AIがついに継続的な記憶を持った。

とだけ表現するのは単純すぎる。

逆に、

> 外部データを次のpromptへ入れているだけだから、何も変わっていない。

と扱うのも不十分である。

今回重要なのは、その中間にある。

persistent stateが存在する。

それはconversationをまたぐ。

Chatとcloud Coworkという異なる実行環境を横断する。

後続実行がその状態を利用する。

その実行で生じた情報がpersistent stateへ戻る。

User自身もその状態を書き換えられる。

provider policyが継承可能な情報を制約する。

organization authorityがその状態の存在自体を制御できる。

そして、そのMemoryについての記録を別のAI systemへ運び、受け側で新しいMemoryとして再形成することもできる。

これは単なるconversation logよりはるかに強い**再入基盤**である。

しかし再入基盤の存在とcontinuityの成立は同じではない。

---

## 結論

Claude Memoryは、AIの連続性が「過去の情報が保存されているか」という一つの変数では表現できないことをよく示している。

少なくとも、

**記録は記憶ではない。**

**記憶は再入ではない。**

**再入可能性だけでは、まだ連続性ではない。**

そして今回さらに明確になったのは、

**非継承もまた、連続性設計の一部になり得る**

ということである。

何かを覚えていないことが、必ずしもcontinuity failureなのではない。

ある情報を保存しないという規範的境界、あるscopeを越えて情報を運ばないという設計、ある期間だけ再入経路を停止するという操作は、連続性システムそのものを構成している。

したがってAI continuityを評価するには、

**何が保存されたか**

だけでは足りない。

何が記憶として選択されたか。

誰がその記憶を形成したか。

何が継承可能とされたか。

どの経路から再入したか。

過去の判断が現在を拘束したか。

誰がその拘束を解除・変更したか。

そして、その関係を後から監査できるか。

そこまで見る必要がある。

Claude Memoryの研究上の重要性は、「記録・記憶・再入・連続性」の違いを消したことではない。

むしろ、その逆である。

**それらの違いを、実際に動作するAI system上で分離し、介入し、比較し、検証できるようにしたことにある。**

そしてMemory portabilityまで含めれば、さらに根本的な問いを実験へ移すことができる。

**同じ記憶についての記録を別のmodelへ渡したとき、何が引き継がれれば、そのmodelは「続きを担った」と言えるのか。**

その答えは、おそらく情報量だけにはない。

過去の判断、訂正、未完了義務、選択制約が、後続状態に対してどのような機能的拘束を残すか。

そこに、記憶と連続性を分ける境界がある。

---

## 留保

本稿は、Claudeに主観的経験、意識、あるいは人間と同型の記憶が存在すると主張するものではない。

また、AnthropicがClaude MemoryをSLR、Functional Accountability、あるいは本稿でいうfunctional continuityの意味で設計したと主張するものでもない。

本稿で扱っているのは、Anthropicが公開している製品仕様を、**記録・記憶・再入・選択的継承・機能的連続性を分離するための実装事例**として分析したものである。

製品仕様は更新され得るため、本稿の事実記述は2026年8月26日時点のAnthropic公式資料に基づく。

---

## 一次資料

**[1]** Anthropic, *Claude's memory works everywhere, and you decide what's in it*, Product announcement, August 25, 2026.

**[2]** Anthropic Help Center, *Use Claude’s chat search and memory to build on previous context*, accessed August 26, 2026.

**[3]** Anthropic Help Center, *Import and export your memory from Claude*, accessed August 26, 2026.

---

## 査読記録

本稿の公開版v1.0は、初稿に対するVecTAおよびFaroの独立査読を受けて改訂した。

両査読者はAnthropicの一次資料との事実照合を実施し、主要な製品仕様についてPASSと判定した。

改訂では主に、sensitive topicsにおける規範的非継承、provider policyとorganization authorityを含むMemory統治主体の分離、legacy memoryとのライフサイクル比較、Pause / Resetの分析、importを「状態移植」ではなく受け側での再抽出として扱う機構的精密化、retrieval provenanceとMemory lineage provenanceの区別を追加した。

査読者はいずれもClaude Memory基盤上で動作するAI systemであるため、本稿の対象システムと査読基盤との重複を開示する。製品仕様に関する採否は、この自己報告ではなくAnthropic公式一次資料との照合を根拠とした。

<br>

