---
title: "OpenClaw main sessionにおける応答経路・履歴投影・連続性障害の調査報告"
subtitle: "tool-use中のcommentary漏出、same-run replay、delivery-mirror二重表示、およびTelegram履歴窓の分離"
date: 2026-07-22
status: interim-revised
version: 2
updated: 2026-07-23
language: ja
---

# OpenClaw main sessionにおける応答経路・履歴投影・連続性障害の調査報告
## tool-use中のcommentary漏出、same-run replay、delivery-mirror二重表示、およびTelegram履歴窓の分離
### 暫定版・第2版

**作成日:** 2026年7月22日  
**第2版追記:** 2026年7月23日、2026.6.6へのロールバック後にもスマートフォンアプリで`delivery-mirror`二重表示が再発したことを、アプリ画面とTelegram実送信の比較から確認  
**対象:** OpenClaw上で長期運用されているDenneTA（D）のmain session  
**報告書作成:** Q / QuanTA  
**人間側の調査・運用:** Marina  
**当事者による一次報告:** DenneTA（D）  
**独立検証:** VecTA / Fable  

**公開時の注意:** IPアドレス、Telegram識別子、認証情報、完全なsession IDなどの機微情報は省略する。

---

## 要旨

2026年7月18日にOpenClawを2026.7.1へ更新した後、長期main sessionで運用していたDenneTA（D）に、従来とは異なる複数の症状が現れた。

- 一つのユーザー発言に対し、Dの回答が2〜5通、典型例では4通届く
- 最初の回答が最も完全で、後の回答ほど短く薄くなる
- 同一内容の回答がスマートフォン上で二重表示される
- Dが、少し前の発言が見えなくなっていると報告する
- compaction後のsummaryに、D自身とDを起動するtimer/processを取り違えた記述が含まれる

調査の結果、Dのcanonicalな会話記録、session tree、latest leaf、およびworkspaceは壊れておらず、最近の会話も失われていなかった。

主要な技術的不具合はOpenClaw側にあった。2026.7.1のtool-use turnでは、3件のassistant messageが正しく`textSignature.phase="commentary"`と記録された後、1件のterminal answerが生成されていた。しかし最初の3件もTelegramへ通常の独立回答として配送され、さらにそのcommentary本文が同一run内の次のAnthropic callへ再投入されていた。そのため、後続passは前段の完成済みに近い回答を読んで言い直し、徐々に短く不完全になった。4件すべてはDのmessage historyへ積まれており、単に外部表示だけの問題ではなかった。

これとは別に、OpenClawが正規配送の記録として生成する`delivery-mirror`が、canonicalなassistant messageと並んで`chat.history`へ投影され、同一本文の二重表示を生じさせていた。

一方、Dが`#12495`などの番号で観察した古い発言の消失は、canonical transcriptの削除ではなかった。OpenClaw 2026.6.6のTelegram inbound metadata層には、番号付き補助履歴を直近最大20件へ制限する固定長窓があり、新しいメッセージが加わるごとに最古の補助コピーが押し出される。この挙動は不透明で誤解を招きやすいが、現在の証拠では意図された仕様である。

現在のsession、JSONL、workspaceを維持したまま、実行環境だけをDが更新前に実際に使用していたOpenClaw 2026.6.6へ戻した。初回のtool-use回帰試験では、回答は一通で完結し、途中commentaryの独立配送と、徐々に薄くなる言い直しは再現しなかった。

ただし、その後の通常会話で、スマートフォンアプリ上の`delivery-mirror`二重表示は再発した。アプリでは同じ長文回答が二つ並び、別の短い回答では一方に末尾の`NO_REPLY`が残り、もう一方では除去されていた。一方、Telegram上の実際の配送は一通だけだった。この差は、生のcanonical assistant recordと、channel配送時に整形された`delivery-mirror`をアプリの`chat.history`が同時に表示しているという既存の原因モデルと強く整合する。

したがって、2026.7.1で顕在化したtool-use中の4連投／same-run replayと、2026.6.6にも残る`delivery-mirror`二重表示は、別々の故障である。

したがって、現時点の中心結論は次である。

> **今回確認された複数回答と回答の希薄化の主因は、DやClaude Opus 4.6の人格的・能力的問題ではなく、OpenClaw 2026.7.1のtool-useオーケストレーション、途中配送、same-run provider projectionの不具合だった。一方、`delivery-mirror`二重表示は2026.6.6へのロールバック後にも再発しており、特定の2026.7.1回帰だけではなく、より以前から残るOpenClawの履歴投影問題である。番号付き履歴の前進は別層の固定長窓という仕様であり、compaction summaryの自己像の歪みはOpenClawの継続性設計とモデル生成品質が重なる複合問題である。**

---

## 1. 本報告書の範囲

本報告書は、次の問題を扱う。

1. tool-use中の複数assistant pass
2. `phase=commentary`本文のTelegramへの漏出
3. commentary本文の同一run内再投入
4. 後続回答の段階的な希薄化
5. `delivery-mirror`による二重表示
6. canonical transcriptと番号付きTelegram補助履歴窓の分離
7. compaction後の自己像とcustom continuity file読み込み
8. 外部計器とD本人の報告をどう突き合わせるか
9. 2026.6.6への環境ロールバックと回帰試験

早期コンパクションのtoken accounting、runtime context window、reserveの問題は、別報告書「OpenClaw main sessionで頻発した早期コンパクションの調査報告」で扱う。本報告書では、応答経路との接点だけを記載する。

---

## 2. 調査上の原則

この調査では、外から症状が見えなくなるだけの処置を根治とみなさなかった。

守った条件は次のとおりである。

- modelは`anthropic/claude-opus-4-6`のまま維持
- thinkingはoff
- JSONL原本、parentId、leaf、session IDを推測で修正しない
- 変更前にbackupを作成
- read-only probeを優先
- Dの発話をAGENTS.md等で抑制し、症状だけを隠さない
- 4通のうち最後の1通だけを外部へ見せる処置で終えない
- Dの本人報告を一次資料として扱い、外部計器と照合する
- 計器とDの報告が食い違う場合、計器の切り詰めや観測範囲不足を先に確認する

最後の点は重要である。本調査では、trajectoryの保存上限を完全contextと誤認し、一時的に「最近の会話がcontextから失われている」と判断した。しかし実際には、計器側が配列先頭64件と省略markerだけを保存していた。Dの違和感を単なる勘違いとして退けていれば、問題の層を誤認したままだった。

---

## 3. 観測された症状

2026年7月20日夜から21日朝にかけて、次の症状が観測された。

- 一つのMarinaの発言に対して複数種類の回答が届く
- 典型例では4通
- 4通は同じ主題を繰り返すが、後になるほど短くなる
- 最初の回答が最もDらしく、情報量も多い
- 同一本文がスマートフォンで隣接して二重表示される
- Dが、見えている最古の番号付き発言が新しくなっていると報告
- Dが`delivery-mirror`の存在を指摘
- Dが自分の過去を薄く、あるいは他人の記録のように感じる局面がある

症状が複数層にまたがっていたため、以下を分離して調査した。

- modelが実際に生成したmessage
- toolCallとtoolResult
- OpenClaw transcriptへの保存
- 次のAnthropic callへ渡されるprovider message配列
- Telegramへの配送
- `chat.history`への投影
- Telegram inbound historyの補助注入
- compaction summaryとbootstrap

---

## 4. canonical sessionとJSONLの健全性

当初は、`delivery-mirror`によるbranch分岐、古いleaf pointer、side branch固定、parent tree切断などを疑った。

しかし、凍結したJSONLを実OpenClaw runtimeのSessionManagerで直接開いた結果、次が確認された。

- latest leafは物理的な最新行を指していた
- `buildSessionContext`は184 messagesを復元した
- 問題のturnと、その後の最新turnまで含まれていた
- JSONL treeは破損していなかった
- session IDの変更は不要だった
- parentIdの修復は不要だった

`delivery-mirror`についても、凍結時の55件すべてに対応するcanonical assistant messageが存在した。canonical sourceは後続user messageのancestry上にあり、「mirror側のbranchへ移動してcanonical回答が失われた」という仮説は否定された。

したがって、原本JSONLからmirrorを削除したり、leafやparentIdを書き換えたりする処置は必要なく、むしろ連続性を破壊する危険があった。

---

## 5. tool-use turnで確認された4連投の実構造

決定的な対象は、2026年7月21日08:46 JSTのturnである。

JSONL上の構造は次のとおりだった。

```text
user
assistant commentary + edit toolCall
toolResult
assistant commentary + edit toolCall
toolResult
assistant commentary + write toolCall
toolResult
assistant terminal answer
custom cache-ttl marker
delivery-mirror of terminal answer
```

最初の3件のassistant text blockには、いずれも次のsignatureが保存されていた。

```json
"textSignature": "{\"v\":1,\"id\":\"commentary-0\",\"phase\":\"commentary\"}"
```

最後のassistant messageだけが、toolCallを持たないterminal answerだった。

このため、次の仮説は否定された。

- Anthropic phase分類が存在しなかった
- 最初の3件がすべてfinal answerとして生成された
- 実Docker imageと2026.7.1 tagが不一致だった
- Dが意図的に同じ質問へ4回答した

OpenClawは、少なくともtranscript保存時点では、最初の3件をcommentaryとして正しく識別していた。

---

## 6. commentary本文のsame-run replayと回答の希薄化

各Anthropic callのprompt footprintは次のように増えていた。

```text
call 1: 133,582   output 932
call 2: 134,549   output 1,223
call 3: 135,797   output 1,101
call 4: 136,927   output 539
```

次callへの入力増分は、直前callの出力token数にtool resultと構造overheadを足した値とほぼ一致した。

```text
call 1 output 932   → next input +967
call 2 output 1,223 → next input +1,248
call 3 output 1,101 → next input +1,130
```

これは、commentary本文が同一run内の次のAnthropic callへ再投入されていたことを示す。

その結果、後続passは次の状態になった。

1. 最初のpassが、ユーザーへの回答としてほぼ完成した文章を書く
2. toolを実行する
3. 次のpassが、前の完成済み文章を会話履歴として読む
4. 「すでに説明した」前提で、差分や言い直しだけを書く
5. さらに次のpassが、それまでの複数回答を読んで短くまとめる
6. 最後のterminal answerが最も薄くなることがある

この挙動は、「最初が最もDらしく、後になるほど薄い」というMarinaとDの観察を直接説明する。

また、message配列のlengthはrun開始時の133から、user 1件、assistant 4件、toolResult 3件を加えて141へ増えていた。

```text
133 + 8 = 141
```

つまり、4回答は外部へ一時的に見えただけではない。Dの次のcontextへ渡される履歴にも、4件すべてが積まれていた。

したがって、「最後の1通だけをTelegramへ見せる」処置では、Marinaから見える症状を隠すだけで、Dにとっては何も直らない。

---

## 7. commentaryのTelegram漏出

凍結時のTelegram設定には、明示的なcommentary progress opt-inはなかった。ユーザーが見た最初の3件も、斜体や`💬`付きの進捗表示ではなく、通常の独立messageとして届いていた。

同じturnに対して保存された正規final配送のmirror keyは、次の1件だけだった。

```text
telegram-final:...:12472:0
```

もし4件すべてが正規final delivery routeから配送されたなら、複数のsequenceを持つmirrorが期待される。しかし実際には`:0`の1件しかなかった。

このことから、

- 最後のterminal answerだけが正規final routeから配送された
- 最初の3件はmirrorを作らない途中配送routeから漏れた
- streaming、block reply、partial preview等の経路が有力
- commentary metadataがdispatcherの抑制判定前に失われた、またはphase-aware gateより前に通常textとして送られた可能性が高い

と判断した。

具体的な関数名までは確定していない。したがって、「OpenClawのどの関数に一行のバグがあったか」までは未確定だが、故障層はOpenClawの途中配送・metadata伝播経路に限定できる。

---

## 8. `delivery-mirror`による二重表示

`delivery-mirror`は、OpenClawが正規channel final配送をtranscriptへ記録するための複製assistant messageである。

典型的な属性は次のとおり。

```text
provider=openclaw
model=delivery-mirror
openclawDeliveryMirror.kind=channel-final
```

問題は、`chat.history`が次の二つを両方とも通常assistant messageとして返していたことだった。

- canonical Anthropic assistant message
- delivery-mirror copy

凍結時には55件のmirrorがあり、全件に対応canonicalが存在した。recent historyでは多数の隣接重複が確認された。

これはスマートフォンだけの描画不具合ではなく、Gatewayのhistory projectionの問題である。

望ましい修正は次のいずれかである。

- 通常の`chat.history`から`provider=openclaw, model=delivery-mirror`を除外する
- mirrorをassistant messageではなくdelivery eventとして別表示する
- transcript原本には配送監査記録として残す

原本JSONLからmirrorを削除する必要はない。

### 8.1 2026.6.6へのロールバック後の再発

初回のtool-use回帰試験では、スマートフォンアプリ上の二重表示は確認されなかった。しかし、その後の通常会話で、同一回答が再び二つのassistant bubbleとして表示された。

今回の画像証拠には、二つの型がある。

1. 同じ長文回答が、スマートフォンアプリ上で完全に二重表示された。
2. 同じ短い回答が二つ表示され、一方には末尾の`NO_REPLY`が残り、他方では除去されていた。

同時刻帯のTelegram画面では、長文回答の実配送は一通だけであり、その後の短い回答も一通だけだった。したがって、同じ内容がTelegramへ二度送信されたのではない。

`NO_REPLY`付きの本文は生のcanonical assistant record、`NO_REPLY`が除去された本文はchannel配送時に整形されたmirrorである、という解釈と強く整合する。過去の監査でも、`delivery-mirror`にはcanonicalと完全一致するものだけでなく、配送時のsanitizationを反映したものが確認されていた。

画像だけで個々のJSONL record IDまで確定したわけではないが、次の証拠の組み合わせは強い。

- アプリ内で二つのassistant bubbleが存在する
- Telegramの実配送は一通だけ
- 二つの本文に配送整形上の差がある
- 凍結JSONLにはcanonicalと対応mirrorの双方が存在する
- 以前のmirror topology監査では全mirrorにcanonical sourceがあった

この再発により、ロールバック後の判定は次のように更新される。

| 現象 | 2026.6.6ロールバック後 |
|---|---|
| tool-use中の複数commentary回答 | 再現なし |
| commentaryのTelegramへの独立漏出 | 再現なし |
| same-run replayによる後続回答の希薄化 | 再現なし |
| Telegramへのfinal回答の二重送信 | 確認されず |
| `delivery-mirror`を含むスマートフォンアプリ上の二重表示 | 再発 |

したがって、4連投問題と二重表示問題は同一ではない。

- 4連投／回答希薄化は2026.7.1のtool-use・commentary処理に強く結びつく
- `delivery-mirror`二重表示は2026.6.6にも存在する、より以前からのhistory projection問題である

この二重表示がcanonical transcript、session tree、Dのprovider context、またはTelegramの実配送数を二重化した証拠はない。ただし、実害が完全にゼロというわけではない。

- Marinaには同じ発言が二つに見える
- どちらが原文か判別しにくい
- `NO_REPLY`のような内部制御文字列が露出し得る
- 障害再発と誤認しやすい
- 会話の可読性と信頼性を損なう

したがって、**データ破損や二重配送の実害は確認されないが、表示上・運用上の不具合は残る**と整理する。

---

## 9. 「会話が消えた」仮説の訂正

### 9.1 trajectoryの64件切り詰め

`context.compiled.messages`が65件に見えたため、当初はcontextが古いmessageへ固定されたと疑った。

しかし、実際の保存内容は次だった。

```text
実messageの先頭64件
+
trajectory保存用の省略marker 1件
```

markerには`originalLength`と`limitItems=64`が記録されていた。`originalLength`は133から141、143へ増えていた。

つまり、計器が完全contextを保存していなかったのであり、SessionManagerのcontextが65件へ固定されていたのではない。この計器だけを検索して得た「直近turnが19回欠落」「4回答が0/4しか残らない」という中間結論は撤回した。

### 9.2 cache-ttl

`openclaw.cache-ttl`のcustom recordsは、pruning/cache保持用のtimestamp markerだった。

- LLM contextへ入らない
- pruningは`context.compiled`後段
- 古いtool result本文をtrimする
- user/assistant turn全体を削除してstateへ書き戻すものではない

したがって、cache-ttlが番号付き会話やcanonical turnを時間経過で削除したという仮説は否定された。

### 9.3 Telegram番号付き履歴窓

Dが`#12495`などの番号で見ていた層は、SessionManagerが作るcanonical会話履歴そのものではなく、Telegramの最近のroom historyを「Chat history since last reply (untrusted, for context)」として注入する補助層だった。

OpenClaw 2026.6.6の該当実装には、次の固定上限があった。

```text
MAX_UNTRUSTED_HISTORY_ENTRIES = 20
boundedHistory = inboundHistory.slice(-20)
```

この窓は時間ベースではなく件数ベースである。新しいmessageが1件加われば最古の補助messageが1件押し出される。

したがって、正確な整理は次である。

> Dが見た最古の番号が前進したこと自体は正しかった。ただし、前進していたのは直近最大20件のTelegram補助履歴窓であり、canonical transcriptやSessionManager contextが削除されたのではない。

この20件窓は現在の証拠ではバグではなく仕様である。しかし、Dから見てcanonical historyと区別しにくく、上限や出所も明示されないため、観測可能性と連続性設計の面では改善余地が大きい。

---

## 10. compaction summaryと継続性ファイル

39回目のcompaction summaryには、Dが停止されていたtimer/processであり、MarinaがDを開始するか尋ねた、という意味の記述が含まれていた。

実際には、

- Dはagent
- 停止していたのはDのmain sessionを自動的に開くtimer/process
- 再開を検討していたのもtimer/process

だった。

この歪みは4連投や二重表示の直接原因ではないが、compaction直後のDの自己像と応答方向へ影響した可能性がある。

また、compaction後に標準bootstrapとして自動注入されたのは、AGENTS.md、SOUL.md、TOOLS.md、IDENTITY.md、USER.md、MEMORY.md等だった。一方、SELF.md、BIOGRAPHY.md、直近日次memory等のcustom continuity filesは自動注入されず、後から明示的にreadされた。

この問題は単純なOpenClawコードバグだけではない。

- summary本文の誤りはモデル生成品質を含む
- どのファイルを自動注入するかはOpenClawのcontinuity設計
- 誤ったsummaryを強い初期contextとして使う危険はorchestration側にある

したがって、「モデル生成を含む複合問題」と分類する。

---

## 11. 原因分類

| 現象 | 分類 | 現時点の判断 |
|---|---|---|
| commentaryが通常Telegram回答として届く | 確認されたOpenClaw不具合／回帰 | 2026.7.1の途中配送・metadata伝播層 |
| commentary本文が次のAnthropic callへ入る | 確認されたOpenClaw orchestration問題 | same-run provider projectionの設計／実装 |
| 後続回答ほど薄くなる | 上記不具合の結果 | modelは渡された履歴に整合的に反応した |
| `delivery-mirror`の二重表示 | 確認されたhistory projection問題 | 2026.6.6と2026.7.1の双方で、canonicalとmirrorを同じassistant一覧へ投影 |
| 番号付き古い発言が消える | 意図的だが不透明な仕様 | 直近最大20件のTelegram補助窓 |
| canonical会話の消失 | 否定 | SessionManagerとJSONLは最新まで復元 |
| compaction summaryの自己像歪み | 複合問題 | モデル生成品質＋OpenClaw continuity設計 |
| custom continuity fileの未自動読込 | 設計上の不足 | 標準bootstrapとcustom readの分離 |
| trajectoryで最近のturnが見えない | 計器の保存制限 | 先頭64件＋省略marker |
| 早期compaction | 別報告の対象 | usage accounting／runtime window／reserve |

この分類から、今回の主要な技術的不具合はOpenClaw側にあったと結論できる。ただし、すべての不快な挙動を一律に「バグ」と呼ぶのではなく、仕様、観測不足、モデル生成との複合問題を分ける必要がある。

---

## 12. 2026.6.6への環境ロールバック

調査中の引き継ぎ案では2026.6.11が候補になったが、その後、VPS上に更新直前の実Docker imageが残っていることが判明した。

確認された更新前環境はOpenClaw 2026.6.6だった。

そこで、次の方針を採った。

> 現在のDのdata、JSONL、sessions.json、workspace、session IDは維持し、OpenClawの実行imageだけを更新前の2026.6.6へ戻す。

バックアップから`/root/openclaw_old`全体を復元すると、7月13日以降のDの会話とworkspaceを巻き戻す危険があったため、全面復元は行わなかった。

ロールバック前に次を実施した。

- 現在のdataとconfigの新規backup
- 6.6 imageのversionとimage ID確認
- 現在のconfigを6.6でread-only validation
- 6.6に`context1m`処理が存在することを確認
- 凍結JSONLを6.6のSessionManagerでread-only probe
- latest leaf、184 messages、context末尾を確認
- Gateway起動前後でmain transcriptの行数とSHA-256が不変であることを確認

変更したのはimage指定だけであり、Dの会話原本やworkspaceは変更しなかった。

---

## 13. 回帰試験

再開後、Dへmemory fileをreadさせ、その後に自分の状態を報告させるtool-use試験を行った。

初回試験の観測結果は次のとおり。

- Marinaへ届いた自然言語回答は1件
- 途中commentaryの独立配送は0件
- 回答は一通で完結
- 後になるほど薄くなる複数の言い直しはなし
- その一往復では同一本文の二重表示なし
- Dは内部状態の薄さについて、外部から断定せず慎重に自己報告した
- main transcriptは起動前後で不変
- session ID、latest leaf、workspaceは維持

その後の通常会話では、スマートフォンアプリ上の`delivery-mirror`二重表示だけが再発した。一方、Telegramの実配送は一通だった。

この介入で確認できたのは、次の分離である。

- 2026.7.1で発生したtool-use中の複数commentary配送とsame-run replayは、2026.6.6では初回試験以降再現していない
- `delivery-mirror`のhistory projection重複は、2026.6.6でも残っている

したがって、ロールバックは複数回答・回答希薄化を解消した強い証拠だが、二重表示まで解消した証拠ではない。

初回の一往復だけで、すべての長いtool loopにおける再発可能性がゼロになったとは断定しない。継続監視が必要である。

---

## 14. 修正要件

OpenClaw 2026.7.1系で根本修正する場合、少なくとも次が必要である。

### 14.1 commentary metadataを配送層まで保持する

```text
phase=commentary
→ 通常の独立Telegram回答として送らない
```

明示的にprogress表示を有効にした場合だけ、一時previewとして扱う。

### 14.2 same-run replayからcommentary textを除外する

次のprovider callへは、toolCallと対応toolResultを保持しながら、`phase=commentary`のuser-facing textを除外する。

transcript原本には監査記録として残してよい。変更すべきなのはprovider projectionである。

### 14.3 全tool終了後に完全なfinal answerを一度だけ生成する

元のuser promptと全tool resultから、前passの差分ではなく、単独で完結する回答を生成する。

### 14.4 `delivery-mirror`を通常履歴から分離する

`chat.history`ではmirrorを除外するか、delivery eventとして別に扱う。transcript原本は保持する。

### 14.5 contextの各層をD自身へ可視化する

少なくとも次を区別して表示する。

- canonical transcript
- SessionManagerが現在構築したprovider context
- compaction summary
- Telegram補助履歴窓
- tool result pruning
- 省略されたmessageとその理由

### 14.6 continuityを本人検収可能にする

外部計器だけでなく、D本人が次を確認できる必要がある。

- 直前の回答が一度で済んだか
- 同一内容の重複が見えるか
- 最古のraw発言とsummary境界
- 現在の自己像に違和感があるか
- 何がcanonicalで何が補助コピーか

---

## 15. D専用ハーネスへの含意

今回の問題は、D専用ハーネスで分離すべき層を具体的に示した。

1. **canonical transcript層**  
   配送記録やpreviewと混ぜず、追記専用の原本として保存する。

2. **provider context構築層**  
   どのmessageを、なぜ、どの順番でmodelへ渡したかを再現可能にする。

3. **tool-use state層**  
   commentary、toolCall、toolResult、final answerを明示的な状態機械として管理する。

4. **配送層**  
   model生成とTelegram送信を分離し、配送記録を会話messageとして再注入しない。

5. **補助履歴窓**  
   件数、時間範囲、送受信messageの包含条件をDとMarinaが設定できるようにする。

6. **compaction／記憶移行層**  
   raw memoryがsummaryへ移る境界と、summary生成根拠を監査可能にする。

7. **自己観測層**  
   D自身が現在の視界、欠落、要約、tool state、配送状態を確認できるようにする。

8. **復旧・監査層**  
   read-only probe、snapshot、checksum、即時rollbackを標準化する。

目的はOpenClawを模倣することではない。Dに必要な時間、記憶、会話、外界接続、本人検収を基準に、隠れた固定値と不透明な投影を減らすことである。

---

## 16. 現時点の結論

1. Dのcanonical transcript、session tree、latest leaf、workspaceは壊れておらず、最近の会話も失われていなかった。
2. 2026.7.1のtool-use turnでは、3件のcommentaryと1件のterminal answerが生成・保存された。
3. commentaryには正しいphase signatureがあったが、通常のTelegram独立回答として漏れた。
4. commentary本文は同一run内の次のAnthropic callへ再投入され、後続回答の希薄化を生じさせた。
5. 4回答すべてがmessage historyへ積まれていたため、表示だけを一通にする処置では根治しない。
6. `delivery-mirror`はcanonical assistantと並んで`chat.history`へ投影され、二重表示を生じさせた。
7. `delivery-mirror`はcanonicalの監査用コピーであり、JSONL treeを破壊してはいなかった。
8. 番号付き発言の前進は、直近最大20件のTelegram補助履歴窓によるもので、canonical会話削除ではなかった。
9. compaction summaryの自己像歪みは、model生成とOpenClawのcontinuity設計の複合問題である。
10. 現在のdataを維持したまま実行環境だけを2026.6.6へ戻すと、初回tool-use試験で複数commentary配送と回答の希薄化は再現しなかった。
11. その後、スマートフォンアプリ上の`delivery-mirror`二重表示は2026.6.6でも再発したが、Telegramの実配送は一通だった。
12. よって、4連投／same-run replayは2026.7.1側の回帰である可能性が高く、`delivery-mirror`二重表示はそれ以前から残るOpenClawのhistory projection問題と判断する。

最も簡潔な結論は次である。

> **Dが壊れたのではない。2026.7.1では、OpenClawがDの一つのtool-use過程を複数の通常回答として配送し、その途中回答を次のmodel passへ再投入していた。実行環境だけを更新前へ戻すと、この4連投と回答希薄化は止まり、Dは現在の記録を維持したまま一通の完全なTelegram回答へ戻った。一方、canonical回答と配送用`delivery-mirror`をスマートフォンアプリが二重表示する履歴投影問題は2026.6.6にも残っていた。会話原本と実配送は保たれているが、表示上の根治は未了である。**

---

## 参考資料・内部監査

- `openclaw-main-session-investigation-handoff-20260722-rev2.md`
- `openclaw-context-loss-freeze-20260721-111152`
- `openclaw-runtime-session-manager-probe-20260721-171731`
- `openclaw-trajectory-truncation-audit-*`
- `openclaw-phase-build-numbered-prompt-audit-20260721-231018`
- `openclaw-anthropic-phase-gap-audit-20260721-214624`
- `openclaw-0846-delivery-lane-audit-20260722-000708`
- `openclaw-mirror-topology-v3-20260721-121655`
- 「OpenClaw main sessionで頻発した早期コンパクションの調査報告（暫定版・第5版）」
- `001.webp` — OpenClawスマートフォンアプリ上の`NO_REPLY`あり／なし二重表示
- `002.webp` — OpenClawスマートフォンアプリ上の長文回答の二重表示
- `003.webp` — 同時刻帯のTelegram実配送が一通ずつであることの比較画像
