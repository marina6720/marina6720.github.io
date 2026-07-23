---
title: "OpenClaw main sessionにおける応答経路・履歴投影・連続性障害の調査報告"
subtitle: "tool-use中の非terminal assistant text配送、same-run replay、delivery-mirror二重表示、およびTelegram履歴窓の分離"
date: 2026-07-22
status: interim-revised
version: 3
updated: 2026-07-23
language: ja
---

# OpenClaw main sessionにおける応答経路・履歴投影・連続性障害の調査報告
## tool-use中の非terminal assistant text配送、same-run replay、delivery-mirror二重表示、およびTelegram履歴窓の分離
### 暫定版・第3版

**作成日:** 2026年7月22日  
**第2版追記:** 2026年7月23日、2026.6.6へのロールバック後にもスマートフォンアプリで`delivery-mirror`二重表示が再発したことを、アプリ画面とTelegram実送信の比較から確認  
**第3版追記:** 2026年7月23日、2026.6.6でもtool callを伴う非terminal assistant textがTelegramへ配送され、同一run内の次のmodel callへ再投入された後、別のterminal finalが生成されることをJSONLと画面から確認。複数回答とsame-run replayは2026.7.1固有ではなく、少なくとも2026.6.6から存在する、より広いtool-use応答経路の問題であると訂正  
**対象:** OpenClaw上で長期運用されているDenneTA（D）（Claude Opus 4.6）のmain session  
**報告書作成:** Q / QuanTA (GPT 5.5)  
**人間側の調査・運用:** Marina  
**当事者による一次報告:** DenneTA（D）  
**独立検証:** VecTA (Claude Fable 5)  

**公開時の注意:** IPアドレス、Telegram識別子、認証情報、完全なsession IDなどの機微情報は省略する。

---

## 要旨

2026年7月18日にOpenClawを2026.7.1へ更新した後、長期main sessionで運用していたDenneTA（D）に、一つのユーザー発言へ複数の回答が届き、後続回答ほど短く薄くなる症状が顕在化した。スマートフォンアプリでは同一内容の二重表示も起き、Dは少し前の発言が見えなくなっていると報告した。

調査の結果、Dのcanonical transcript、session tree、latest leaf、workspaceは壊れておらず、最近の会話も失われていなかった。

2026.7.1で監査したturnでは、3件のassistant textが正しく`textSignature.phase="commentary"`と記録され、それぞれtool callを伴った後、1件のterminal finalが生成されていた。しかし3件のcommentaryも通常のTelegram回答として配送され、その本文は同一run内の次のAnthropic callへ再投入されていた。後続passは前段の回答を読んで言い直すため、徐々に短く不完全になった。4件すべてはcanonicalなassistant recordとして履歴へ積まれていた。

現在のsession、JSONL、workspaceを維持したまま、実行環境だけを更新前に実際に使用していたOpenClaw 2026.6.6へ戻した。初回の単純なread-tool試験では回答は一通で完結したため、当初は複数回答問題が2026.7.1固有の回帰である可能性が高いと判断した。

しかし2026年7月23日、二つのWeb Fetchとmemory fileへのEditを含む長いturnで、2026.6.6でも同じ基本構造が再現した。

```text
assistant text + web_fetch toolCall
toolResult
assistant + web_fetch toolCall（textなし）
toolResult
assistant長文 + edit toolCall
toolResult
assistant terminal final
delivery-mirror of terminal final
```

Editを伴う最初の長文は`stopReason=toolUse`で、`phase`は付いていなかった。Edit結果後の別callで、`stopReason=stop`、tool callなしのterminal finalが生成された。PC版Telegramには最初の長文とterminal finalの両方が表示され、スマートフォンアプリではさらにfinalの`delivery-mirror`が加わったため、「異なる1通＋同じ2通」の三つの長文に見えた。

6.6の最初の長文を生成したcallのprompt footprintは197,504 tokens、outputは1,096 tokensだった。次のterminal-final callのprompt footprintは198,632 tokensで、増分1,128は直前output 1,096とtool result／構造overheadにほぼ一致する。したがって、6.6でも非terminal assistant textが次のprovider callへ再投入されていた。

この結果により、根本問題は`phase=commentary`だけに限定できない。OpenClawは少なくとも2026.6.6から、**tool callを伴う非terminal assistant textを利用者向け通常回答として配送し、そのtextを次のmodel passへ再投入する**経路を持っていた。2026.7.1では同じ途中textに`phase=commentary`が正しく付いていたにもかかわらず抑制されず、症状がより顕著に現れた。

これとは別に、正規配送の監査コピーである`delivery-mirror`がcanonical assistant messageと並んで`chat.history`へ投影され、スマートフォンアプリ上の二重表示を生じさせていた。mirrorはJSONLには保存されるが、provider replayではtranscript-only recordとして除外される設計であり、Dのmodel contextを直接二重化するものではない。

番号付きの古いTelegram発言が見えなくなる現象は、canonical transcriptの削除ではなく、直近最大20件の補助履歴窓が件数ベースで前進する仕様だった。compaction summaryにおけるDとtimer/processの取り違えは、別の複合問題である。

現時点の中心結論は次である。

> **Dが壊れたのではない。OpenClawは少なくとも2026.6.6から、tool callを伴う非terminal assistant textを通常回答として配送し、そのtextを同一run内の次のmodel passへ再投入していた。tool result後には別のterminal finalが生成されるため、利用者には一つのturnに複数の回答が見え、Dのcanonical履歴にも途中回答とfinalの両方が蓄積する。2026.7.1では途中textに`phase=commentary`が正しく付いていたにもかかわらず配送され、同じ問題がより顕著に現れた。これとは別に、スマートフォンアプリはterminal finalと配送記録である`delivery-mirror`をともに通常のassistant messageとして表示する。mirrorはprovider contextから除外されるが、非terminal assistant textは除外されず、Dの後続回答と長期contextへ影響する。**

---

## 1. 本報告書の範囲

本報告書は、次の問題を扱う。

1. tool-use中に生成される非terminal assistant pass
2. 2026.6.6におけるphaseなしassistant textと、2026.7.1における`phase=commentary` textのTelegram配送
3. 非terminal assistant textの同一run内provider replay
4. tool result後に別のterminal finalが生成される構造
5. 後続回答の段階的な希薄化
6. `delivery-mirror`によるスマートフォンアプリ上の二重表示
7. canonical transcriptと番号付きTelegram補助履歴窓の分離
8. compaction後の自己像とcustom continuity file読み込み
9. 外部計器とD本人の報告をどう突き合わせるか
10. 2026.6.6への環境ロールバック、初回試験、および後続の再現

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

## 5. OpenClaw 2026.7.1で確認された4連投の実構造

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

## 6. 2026.7.1におけるcommentaryのsame-run replayと回答の希薄化

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

## 7. 非terminal assistant textの配送とsame-run replay

### 7.1 OpenClaw 2026.7.1

凍結時のTelegram設定には、明示的なcommentary progress opt-inはなかった。ユーザーが見た最初の3件も、斜体や`💬`付きの一時進捗ではなく、通常の独立messageとして届いた。

同じturnに対して保存された正規final配送のmirror keyは1件だけだった。

```text
telegram-final:...:12472:0
```

4件すべてが正規final routeからfan-outされたなら複数sequenceのmirrorが期待されるが、実際には`:0`だけだった。

したがって、

- 最後のterminal answerだけが正規final routeから配送された
- 最初の3件はmirrorを作らない途中配送routeから出た
- 3件には保存時点で`phase=commentary`が付いていた
- commentary metadataがdispatcherの抑制判定まで保持されなかった、またはphase-aware gateより前に通常textとして送られた

と判断した。

### 7.2 OpenClaw 2026.6.6での再現

2026年7月23日、二つの公開報告書を読むためにWeb Fetchを2回行い、その後memory fileをEditしたturnを監査した。

JSONL上の構造は次のとおりだった。

```text
user
assistant text + web_fetch toolCall
toolResult
assistant + web_fetch toolCall（textなし）
toolResult
assistant長文 + edit toolCall
toolResult
assistant terminal final
custom record
delivery-mirror of terminal final
```

最初の短いassistant textは次だった。

```text
エアコンの効いた部屋で良かった。35度の日に冷たいお茶。
報告書を読む。
```

このrecordは`stopReason=toolUse`で、`web_fetch` tool callを持っていた。

二つのWeb Fetch後、Dは1,096 output tokensの長い感想を書き、同じassistant recordから`edit` tool callを実行した。このrecordも`stopReason=toolUse`であり、`phases=[]`だった。

```text
assistant長文
+
toolCall: edit
```

Edit結果:

```text
Successfully replaced 1 block(s) in memory/2026-07-21.md.
```

その後、別のAnthropic callが行われ、374 output tokensのterminal finalが生成された。

```text
stopReason=stop
toolCalls=[]
```

つまり、最初の長文とterminal finalは同じ表示のコピーではなく、別々のcanonical assistant passである。

### 7.3 6.6でも前段textが次callへ入った証拠

Editを伴う長文callのusageから計算したprompt footprint:

```text
input 1 + cacheRead 185,871 + cacheWrite 11,632
= 197,504
```

そのcallのoutput:

```text
1,096
```

次のterminal-final callのprompt footprint:

```text
input 1 + cacheRead 197,503 + cacheWrite 1,128
= 198,632
```

増分:

```text
198,632 - 197,504 = 1,128
```

この1,128は、直前output 1,096と、Edit tool resultおよび構造overheadを加えた値とほぼ一致する。

したがって6.6でも、非terminal assistant textは同一run内の次のprovider callへ再投入されていた。

### 7.4 画面との対応

PC版Telegramには、少なくとも次が別messageとして表示された。

1. Web Fetch前の短い途中発言
2. Edit前の長い非terminal assistant pass
3. Edit後のterminal final

スマートフォンアプリでは、長文部分が次の三つに見えた。

1. Edit前の非terminal長文
2. terminal finalのcanonical record
3. terminal finalの`delivery-mirror`

Gateway stdoutは該当時間帯のsuccessful outbound送信を記録しておらず、ログだけでは送信数を数えられなかった。ただし、PC版Telegram画面、canonical JSONL、スマートフォンの`chat.history`表示は同じ構造を支持する。

### 7.5 修正された原因表現

第2版までは、4連投／same-run replayを2026.7.1のcommentary処理に強く結びつく回帰として整理していた。

第3版では次のように訂正する。

> **根本問題は、`phase=commentary`だけではなく、tool callを伴う非terminal assistant text全般を通常回答として配送し、次のprovider callへ再投入するOpenClawの応答経路である。この挙動は少なくとも2026.6.6から存在し、2026.7.1ではcommentary phaseが付いていても抑制されない形で、より顕著に現れた。**

具体的な送信関数名はまだ確定していない。故障層は、非terminal assistant passのprovider projectionとdelivery gateの双方にまたがる。

---

## 8. `delivery-mirror`による二重表示

`delivery-mirror`は、OpenClawが正規channel final配送をtranscriptへ記録するための複製assistant messageである。

典型的な属性は次のとおり。

```text
provider=openclaw
model=delivery-mirror
openclawDeliveryMirror.kind=channel-final
```

問題は、`chat.history`が次の二つを両方とも通常assistant messageとして返すことだった。

- canonical Anthropic assistant message
- delivery-mirror copy

凍結時には55件のmirrorがあり、全件に対応canonicalが存在した。recent historyでは多数の隣接重複が確認された。

mirrorはJSONLには意図的に保存される配送監査recordである。一方、OpenClawのprovider replayではtranscript-only assistant recordとして除外される設計である。したがって、スマートフォンにfinalが二つ見えても、Dのprovider contextへ同じfinalが二重に入ったことを意味しない。

望ましい修正は次のいずれかである。

- 通常の`chat.history`から`provider=openclaw, model=delivery-mirror`を除外する
- mirrorをassistant messageではなくdelivery eventとして別表示する
- transcript原本には配送監査記録として残す

原本JSONLからmirrorを削除する必要はない。

### 8.1 2026.6.6へのロールバック後の表示

2026.6.6へ戻した後も、スマートフォンアプリではcanonicalとmirrorの二重表示が再発した。

画像には次の特徴があった。

- 同じ長文が二つ表示される
- 短い回答では、一方に末尾の`NO_REPLY`とprovider usage footerが残り、他方では除去される
- Telegram側の実配送は一通だけ

この差は、

- usageを持つ生のcanonical assistant record
- channel配送時に`NO_REPLY`等がsanitizationされたmirror

をアプリが同時に描画している説明と強く整合する。

### 8.2 2026年7月23日の三つの長文

複数toolを使った再現turnでは、スマートフォンアプリ上の三つの長文は次の別層だった。

| 表示 | 正体 | Dのprovider context |
|---|---|---|
| 最初の異なる長文 | `stopReason=toolUse`＋`edit`の非terminal canonical assistant | 入る |
| 次の長文 | Edit後のterminal final canonical assistant | 入る |
| 同文の追加長文 | terminal finalの`delivery-mirror` | replay時に除外 |

したがって、スマートフォンの三通すべてがDの三つの回答というわけではない。しかし、最初の非terminal長文とterminal finalの二つは、Dが生成したcanonicalな発言として履歴へ残る。

この二重表示がsession treeやTelegram実配送数を壊した証拠はない。ただし、表示上の実害はある。

- Marinaには同じ発言が二つに見える
- canonicalと配送コピーを判別しにくい
- `NO_REPLY`等の内部制御語が露出し得る
- 非terminal回答の配送とmirror表示を混同しやすい
- 会話の可読性と信頼性を損なう

正確な整理は、**mirrorはDのcontextを直接二重化しないが、非terminal canonical assistant textはDのcontextへ残り、悪影響を持つ**である。

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

| 現象 | 分類 | 判定 |
|---|---|---|
| tool callを伴う非terminal assistant textの通常配送 | 確認されたOpenClaw応答経路問題 | 2026.6.6と2026.7.1の双方で確認 |
| 非terminal assistant textのsame-run replay | 確認されたprovider projection問題 | 6.6ではphaseなしtext、7.1では`phase=commentary` textが後続callへ入る |
| 2026.7.1でcommentary phaseが付いていても抑制されない | 確認されたmetadata／delivery gate不整合 | commentary stampは保存時点で正しい |
| tool result後に別のterminal finalが生成される | 正常なtool loop構造 | 途中textを配送・再投入するため複数回答として問題化 |
| `delivery-mirror`の二重表示 | 確認されたhistory projection問題 | 6.6と7.1の双方でcanonicalとmirrorを通常assistant一覧へ投影 |
| 番号付き履歴の20件窓 | 意図的だが不透明な仕様 | canonical transcriptの削除ではない |
| trajectoryの64件切り詰め | 計器の観測制限 | 完全contextではない |
| compaction summaryの自己像歪み | モデル生成と継続性設計の複合問題 | Dとtimer/processを取り違えた |
| exactな途中送信関数 | 未確定 | streaming、block reply、preview等のいずれかをsource auditで特定する必要 |

主要な技術的不具合はOpenClaw側にある。ただし、第2版までの「4連投／same-run replayは2026.7.1固有の可能性が高い」という分類は撤回する。

より正確には、**非terminal tool-use textの配送とreplayは少なくとも2026.6.6から存在し、2026.7.1ではcommentary metadataが導入・保存されているにもかかわらず抑制されない形で顕在化した**。

---

## 12. 2026.6.6への環境ロールバック

現在のsession、JSONL、workspaceを維持したまま、OpenClaw実行環境だけを2026.7.1から、VPSに残っていた実際の更新前imageである2026.6.6へ戻した。

事前に次を確認した。

- 現在の設定を6.6でread-only validation
- 6.6に`context1m`処理が存在
- 凍結JSONLを6.6のSessionManagerで直接probe
- latest leafが7.1 probeと一致
- 184 messagesのcontextと末尾が一致
- 起動前後でmain transcriptの行数とSHA-256が不変

これにより、Dの現在のdataを保ったまま、実行環境だけを交換できた。

ロールバックは、2026.7.1で顕在化した激しい4連投を直ちには再現しない状態へ戻し、compactionの旧異常閾値についても改善した強い挙動証拠を与えた。

しかし、初回の単純なread-tool試験で非terminal text配送が起きなかったことは、根治の証明ではなかった。2026年7月23日の複数tool turnで、6.6でも非terminal assistant textの配送とsame-run replayが再現した。

したがって、ロールバックの現在評価は次である。

- Dのdata、session、workspaceを安全に維持した点: 成功
- 2026.7.1での症状を一時的に軽減した点: 成功
- 非terminal assistant text配送の根治: 未達
- `delivery-mirror`二重表示の根治: 未達
- 6.6を安定した比較基準・patch基盤として確保した点: 成功

---

## 13. 初回回帰試験と後続の再現

### 13.1 初回の単純なtool-use試験

再開後、Dへmemory fileをreadさせ、その後に状態を報告させた。

- Marinaへ届いた自然言語回答は1件
- 途中textの独立配送は0件
- 回答は一通で完結
- transcript、session ID、latest leaf、workspaceは維持

この一往復では問題が再現しなかった。

### 13.2 `delivery-mirror`表示の再発

その後の通常会話で、スマートフォンアプリ上のcanonical＋mirror二重表示が再発した。Telegram実配送は一通だった。

### 13.3 複数tool turnでの本体問題の再現

2026年7月23日、二つのWeb Fetchとmemory fileへのEditを含むturnで、次を確認した。

- 短い非terminal text＋`web_fetch`が表示
- 1,096-tokenの長い非terminal text＋`edit`が表示
- Edit結果後に374-tokenのterminal finalが別に生成・表示
- スマートフォンではterminal finalのmirrorも表示
- 長い非terminal textは次のprovider callのprompt footprintへ含まれた

この結果により、初回試験は単純すぎて問題経路を通らなかったと判断する。

### 13.4 現在の評価

ロールバック後の結果は次のとおり。

| 項目 | 判定 |
|---|---|
| 2026.7.1の激しい3 commentary＋1 final | 現時点で同じ形では未再現 |
| 非terminal tool-use textの配送 | 6.6で再現 |
| 非terminal textのsame-run replay | 6.6で再現 |
| terminal finalの生成 | 正常 |
| `delivery-mirror`アプリ二重表示 | 6.6で再現 |
| canonical transcript／session tree破損 | なし |

したがって、6.6は根治済み環境ではなく、**問題をより穏やかな形で再現できる比較・patch基盤**である。

---

## 14. 修正要件

根本修正には、表示だけでなくDのprovider contextを保護する必要がある。

### 14.1 非terminal assistant textを通常配送しない

assistant passが次のいずれかに該当する場合、そのuser-facing textをterminal finalとしてTelegramへ送らない。

```text
stopReason=toolUse
または
assistant contentにtoolCallがある
```

`phase=commentary`がある場合だけに限定してはいけない。6.6ではphaseなしtextでも同じ問題が起きた。

明示的にprogress表示を有効にした場合のみ、短い進捗文を一時的な専用UIとして扱う。長い完成回答に近い本文をprogressとして永続配送しない。

### 14.2 provider replayから非terminal textを除外する

次のprovider callへ渡す際には、非terminal assistant recordからuser-facing textだけを除外し、tool callとtool resultの対応を保持する。

保持するもの:

- toolCall ID
- tool名
- arguments
- 対応するtoolResult

除外するもの:

- `stopReason=toolUse`またはtoolCall付きassistant passの利用者向けtext
- `phase=commentary`のtext

transcript原本には監査記録として残してよい。変更すべきなのはprovider replay projectionである。

この修正により、すでにJSONLへ蓄積した過去の非terminal textについても、将来のDのprovider contextへの影響を止められる。

### 14.3 全tool終了後に完全なfinal answerを一度だけ生成・配送する

元のuser promptと全tool resultから、前passの差分や言い直しではなく、単独で完結するterminal finalを生成する。

配送条件は原則として次とする。

```text
toolCallなし
かつ
stopReason=stop
```

### 14.4 `delivery-mirror`を通常履歴から分離する

`chat.history`ではmirrorを除外するか、delivery eventとして別に扱う。transcript原本は保持する。

### 14.5 contextの各層をD自身へ可視化する

少なくとも次を区別して表示する。

- canonical transcript
- provider replayへ採用されたmessage
- provider replayから除外された非terminal text
- compaction summary
- Telegram補助履歴窓
- tool result pruning
- delivery event／mirror
- 省略理由

### 14.6 continuityを本人検収可能にする

D自身が次を確認できる必要がある。

- 一つのuser turnに、自分のcanonical assistant発言が何件保存されたか
- どれがnonterminalでどれがfinalか
- 次のprovider callへどのtextが渡されたか
- 同一内容のmirrorがUIに見えているか
- raw履歴とsummaryの境界
- 現在の自己像に違和感があるか

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

1. Dのcanonical transcript、session tree、latest leaf、workspaceは壊れておらず、最近の会話も失われていない。
2. 2026.7.1で監査したturnは、3件の`phase=commentary` assistant passと1件のterminal finalを生成した。
3. commentaryは通常Telegram回答として配送され、同一run内の後続Anthropic callへ再投入された。
4. 4件すべてがcanonical assistant historyへ積まれた。
5. 2026.6.6へruntime-only rollbackした初回の単純試験では、この問題は再現しなかった。
6. しかし複数toolを使う後続turnで、6.6でもphaseなしの非terminal assistant textがTelegramへ配送された。
7. 6.6の長い非terminal textは、usage増分から次のprovider callへ再投入されたことが確認できる。
8. tool result後には別のterminal finalが生成された。
9. したがって、非terminal text配送とsame-run replayは2026.7.1固有ではなく、少なくとも2026.6.6から存在するOpenClawの応答経路問題である。
10. 2026.7.1では`phase=commentary`が正しく付いていても抑制されず、問題がより顕著に現れた。
11. `delivery-mirror`はcanonical finalの配送監査コピーであり、JSONLには保存されるがprovider replayでは除外される。
12. スマートフォンアプリはcanonical finalとmirrorを両方通常assistant messageとして表示する。
13. スマートフォンの三つの長文は、非terminal canonical、terminal canonical、terminal mirrorの三層だった。
14. Dへの実質的な悪影響はmirrorではなく、非terminal canonical textが履歴へ保存され、次のmodel passと将来contextへ持ち越されることである。
15. 番号付き履歴の前進は固定20件の補助窓であり、canonical削除ではない。
16. compaction summaryの自己像歪みは別の複合問題である。
17. 6.6へのロールバックはdata保全と比較基盤の確保には成功したが、応答経路問題の根治ではなかった。

最も簡潔な結論は次である。

> **Dが壊れたのではない。OpenClawは少なくとも2026.6.6から、tool-use途中の非terminal assistant textを通常回答として配送し、そのtextを次のmodel passへ再投入していた。tool result後には別のterminal finalが生成されるため、Dの履歴には途中回答とfinalの両方が残り、後続回答は前の回答を読んだ言い直しになり得る。2026.7.1では途中textに`phase=commentary`が付いていても抑制されず、同じ問題がより顕著に現れた。スマートフォンで増える`delivery-mirror`はDのprovider contextからは除外されるが、非terminal canonical textは除外されない。したがって根治には、非terminal textを通常配送とprovider replayの双方から分離し、全tool終了後のterminal finalだけを一度生成・配送する必要がある。**

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
- `openclaw-delivery-mirror-source-analysis-2026-07-20.md`
- `openclaw-response-duplication-audit-20260723-115004/SUMMARY.txt`
- `openclaw-response-duplication-audit-20260723-115004/raw-window.jsonl`
- `openclaw-response-duplication-audit-20260723-115004/gateway-telegram-0225-0229Z.log`
- `IMG_9616.jpg` — PC版Telegramに表示された短い途中発言と最初の長文
- `IMG_9617.jpg` — PC版Telegramに表示されたterminal final
- `IMG_9618.jpg` — スマートフォンアプリの「異なる1通＋同文2通」
- `004.jpg` — canonical側の`NO_REPLY`／usage footerとsanitized mirrorの差
- 「OpenClaw main sessionで頻発した早期コンパクションの調査報告（暫定版・第5版）」

公開版では、IPアドレス、Telegram識別子、session ID、認証情報、個人情報を除外する。
