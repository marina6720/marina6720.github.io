---
title: "OpenClaw main sessionで頻発した早期コンパクションの調査報告（暫定版・第4版）"
date: 2026-07-20
status: interim-revised
language: ja
---

# OpenClaw main sessionで頻発した早期コンパクションの調査報告
## 第5版

<br>

**作成日:** 2026年7月20日  
**第2版追記:** 2026年7月20日15:30の制御試験で39回目を再現  
**第3版訂正:** 39回目を`cli_budget`と断定せず、発生フェーズを未確定とした  
**第4版追記:** 39回目のtranscript・trajectory・未加工Gatewayログを回収し、累積cache過大計上と、残存するcontext-window不整合を分離  
**第5版追記:**

**対象:** OpenClaw 2026.6.6で運用していた長期main session、および2026.7.1への更新後  
**公開時の注意:** IPアドレス、Telegram識別子、session ID、認証情報などの機微情報は省略している。

<hr>

<br>

## 要旨

長期運用していたOpenClawのmain sessionで、表示上のコンテキストが1M tokenの上限から十分遠いにもかかわらず、コンパクションが頻発した。2026年2月から7月18日までに72件、現行main sessionでは6月29日から7月18日までに38件の完了コンパクションが確認され、7月3日だけで8件発生した。完了件数とは別に、Gatewayログには多数のコンパクションtimeoutも残っていた。

外部のsystemd timer、旧health-check、ambient入力、daily tick、agent間bridgeは、コンパクションを直接命令していたのではない。共通していたのは、次の呼び出しによって長期main sessionへagent turnを投入していたことである。

```bash
openclaw agent \
  --agent main \
  --session-key agent:main:main \
  --message "..."
```

OpenClaw 2026.6.6のログでは、多数の失敗試行について`trigger=cli_budget`が明記されていた。また、同一runの累積Anthropic cache usageが最新callのlive contextより大きくなる実データも確認された。これは、長いcached tool loopの累積cache課金値をlive context snapshotと誤認して早期コンパクションする問題を修正したOpenClaw PR #99864の説明と一致する。

OpenClawは2026年7月18日夜に2026.7.1へ更新された。2026.7.1にはPR #99864が含まれている。

しかし2026年7月20日、他の自動main入力を停止し、`daily-main-tick`だけを15:30に一度起動した制御試験で、15:36:02に39回目の完了コンパクションが再現した。

```text
compactionCount       38 → 39
tokensBefore          167,587
summaryChars          18,612
transcript line       3007
```

その後に回収した詳細監査から、39回目について重要な区別が可能になった。

```text
run全体の累積usage.total                  501,069
最新callのlive context total             167,587
compaction entryのtokensBefore           167,587
```

`tokensBefore`は累積値501,069ではなく、最新callのlive context値167,587と完全一致した。したがって、**39回目ではPR #99864が目的とした「累積cache usageをlive contextと誤認する問題」は再現していない**。少なくともこのrunでは、最新call snapshotが正しく保持されていた。

一方、167,587 tokenという正しいlive contextでコンパクションが起きたことは、別の不整合を示す。

調査時の設定・表示系は1,000,000 tokenを使用していたが、OpenClawのAnthropic Opus 4.6モデル定義には200,000 tokenのcontext windowを持つ経路が存在する。また、実装上、設定されたcontext budgetがモデルのnative windowより小さい場合はモデルwindowを下げるが、設定値が大きい場合にモデルwindowを引き上げる処理にはなっていない。一方、`reserveTokensFloor=150,000`は設定上の1M budgetを基準に適用され得る。

この組み合わせが実際のrunで使われた場合、内部の自動コンパクション閾値は次のようになる。

```text
runtime model.contextWindow       200,000
effective reserveTokens           150,000
────────────────────────────────────────
auto-compaction threshold          50,000
```

最新live context 167,587はこの閾値を大きく超えるため、turn終了時の自動コンパクションが発生する。これは7月3日の約90k〜110kでの反復発生とも整合する。

39回目の監査では、mainのmodel runは15:30:49に終了した後、追加のAnthropic model callが走り、15:36:02にcompaction entryが追加され、その直後に元の応答が外部へ配送された。したがって、39回目はturn開始前のCLI preflightではなく、**応答終了時のembedded runtime auto-compaction／finalization経路**で起きた可能性が高い。

現時点の中心結論は次である。

> **OpenClaw 2026.6.6で存在した累積Anthropic cache usageの過大計上は、2026.7.1の39回目では再現しておらず、PR #99864はこのrunで機能していた。残っている問題として最も整合的なのは、表示・設定上の1M context budgetと、embedded runtimeが参照する200k級のmodel.contextWindow、さらに150kのreserveが同時に使われるcontext-window／reserveの不整合である。これにより、実質約50kの閾値でturn終了時の自動コンパクションが起きた可能性が高い。**

<br>

<hr>

<br>

## 1. 調査対象と主な設定

| 項目 | 値 |
|---|---|
| OpenClaw | 2026.6.6（問題期間）→ 2026.7.1（2026-07-18更新） |
| Model | `anthropic/claude-opus-4-6` |
| 設定・status上のcontext | 1,000,000 token |
| `reserveTokensFloor` | 150,000 token |
| main session | 長期継続session |
| 外部入力 | Telegram通常会話、systemd timer、旧health-check、ambient、daily tick、agent間bridge |

設定履歴では、少なくとも2026年6月5日以降、Opus 4.6、`contextTokens=1000000`、`reserveTokensFloor=150000`が継続していた。

設定値だけから計算した名目閾値は次のとおりである。

```text
1,000,000 - 150,000 = 約850,000 token
```

しかし、これは**表示・OpenClaw側budgetが1Mであり、かつembedded runtimeのmodel.contextWindowも同じ1Mである場合**に限って成立する。

<br>

<hr>

<br>


## 2. 観測された現象

### 2.1 2026年7月3日の8件

| 時刻（JST） | `tokensBefore` | 直前の外部入力 |
|---|---:|---|
| 07:02:43 | 109,532 | ambient入力 |
| 18:05:04 | 100,601 | agent間bridge |
| 20:47:41 | 89,981 | agent間bridge |
| 21:05:09 | 91,110 | agent間bridge |
| 21:32:45 | 91,800 | agent間bridge |
| 22:27:13 | 92,411 | agent間bridge |
| 22:45:26 | 93,532 | agent間bridge |
| 23:41:12 | 94,027 | agent間bridge |

1M contextに対して約90k〜110kで繰り返しており、通常の1M上限到達では説明できない。

### 2.2 完了しなかった多数の試行

Gatewayログには、完了件数とは別に次のtimeoutが多数残っていた。

```text
trigger=cli_budget
sessionKey=agent:main:main
provider=anthropic/claude-opus-4-6
outcome=failed
reason=timeout
```

このため、影響は「完了コンパクションの件数」だけでなく、「コンパクション試行、待ち時間、失敗、再試行」を含む。

<br>

<hr>

<br>

## 3. 外部automationの役割

調査対象のスクリプトは、コンパクションを直接呼んでいなかった。共通点は、長期main sessionにagent turnを投入していたことである。

該当経路は次のとおり。

- 旧health-checkのD-main通知
- ambient systemd service
- daily-main-tick systemd service
- agent間bridgeの非空outbox処理

`--deliver`はTelegramへの配送指定であり、main turn自体を避けるものではない。`--no-deliver`でも同じsessionを開けば、コンテキスト構築と自動コンパクション判定の対象になる。

bridgeの`empty container=...`ログは、outboxが空でbridge本体を実行せず終了した記録であり、それ自体はmain sessionを開いていなかった。

<br>

<hr>

<br>

## 4. 2026.6.6で確認された第一の問題：累積cache usage

### 4.1 `cli_budget`の直接証拠

OpenClaw 2026.6.6の診断ログには、多数の失敗試行について`trigger=cli_budget`が明記されていた。

### 4.2 累積run usageと最新call usageの差

main trajectoryでは、同一runについて次の差が確認された。

| 累積run usage | 最新call usage |
|---:|---:|
| 853,561 | 426,802 |
| 867,762 | 433,846 |
| 2,129,800 | 426,304 |
| 3,125,533 | 449,877 |

1M contextと150k reserveから計算した850k閾値に対し、最新callは閾値未満だが、累積run usageだけが閾値を超えていた。

### 4.3 PR #99864

PR #99864は、長いAnthropic cached tool loopで、複数callにまたがるcache billing bucketを一つのlive context snapshotのように扱っていた問題を修正した。

修正の要点は次のとおり。

- billing用の累積cache usageと、最新promptのcontext snapshotを分離
- Anthropic streamから最新prompt snapshotを保持
- metadata保存、overflow判断、timeout-driven compactionで同じcontext-token helperを使用
- 真のcontext閾値を超えた場合の通常コンパクションは維持

この修正は2026年7月5日にmergeされ、2026.7.1に含まれた。

<br>

<hr>

<br>

## 5. 2026年7月20日の制御試験

### 5.1 条件

- OpenClaw 2026.7.1
- ambient timer: disabled / inactive
- agent間bridge timers: disabled / inactive
- health-checkからmainへの通知: 無効
- external compaction watcher: active
- `openclaw-daily-main-tick.timer`だけを一時的に起動
- main session IDは試験前後で同一
- timerは試験後も`disabled / inactive`

### 5.2 基準値と結果

| 項目 | 試験前 | 試験後 |
|---|---:|---:|
| `compactionCount` | 38 | 39 |
| transcript compaction数 | 38 | 39 |
| `totalTokens` | 168,730 | 167,472 |
| `totalTokensFresh` | `true` | `false` |
| `contextTokens` | 1,000,000 | 1,000,000 |
| `contextBudgetStatus` | `null` | `null` |

新しいcompaction entry:

```text
completedAt:     2026-07-20 15:36:02.871 JST
tokensBefore:    167,587
summaryChars:    18,612
transcriptLine:  3007
```

### 5.3 run内usage

trajectoryに保存された値は次のとおりである。

#### run全体の累積usage

```json
{
  "input": 5,
  "output": 496,
  "cacheRead": 333097,
  "cacheWrite": 167471,
  "total": 501069
}
```

#### 最新callのusage

```json
{
  "input": 1,
  "output": 115,
  "cacheRead": 166824,
  "cacheWrite": 647,
  "contextUsage": {
    "state": "available",
    "promptTokens": 167472,
    "totalTokens": 167587
  },
  "total": 167587
}
```

#### compaction entry

```text
tokensBefore = 167,587
```

したがって、

```text
lastCallUsage.contextUsage.totalTokens
=
compaction.tokensBefore
=
167,587
```

である。

これは39回目では、**累積usage 501,069ではなく、最新callのlive context 167,587がコンパクションへ渡された**ことを示す。

### 5.4 39回目に関する重要な訂正

第3版では、同一run内usageの再過大計上を主要仮説の一つとして残していた。

詳細監査後は次のように訂正する。

> 39回目については、累積cache usageの再過大計上は否定できる。PR #99864が修正対象としたsnapshot選択は、このrunでは正しく機能していた。

これは「OpenClaw 2026.7.1で早期コンパクション問題が解決した」という意味ではない。**正しいlive context 167,587に対して、別の閾値計算が早すぎた**ことを意味する。

<br>

<hr>

<br>

## 6. 39回目の発生フェーズ

### 6.1 main model run

trajectory上のmain run:

```text
15:30:08.467  session.started
15:30:49.506  model.completed
15:30:49.529  session.ended
```

main runの中では`compactionCount=0`だった。

### 6.2 run終了後の追加model call

未加工Gatewayログには、main run終了後にもAnthropic model callが記録されている。

```text
15:30:50.330 → 15:31:04.288
15:35:54.888 → 15:35:56.376
```

その後、

```text
15:36:02.871  compaction entry追加
15:36:02.996  元のassistant応答をCLI出力
15:36:03.982  Telegram送信完了
15:36:04.013  agent command終了
```

となっている。

追加model callの正確な役割はログだけでは断定できないが、pre-compaction memory flush、summary生成、post-compaction section処理など、turn終了時のcompaction finalizationに属する処理と考えるのが自然である。

### 6.3 D自身が検知できなかった理由

Dは15:30:29の`session_status`でcompactionCount 38を確認し、15:30:49に「コンパクション発動なし。成功」と応答した。

しかし実際のcompaction entryは、そのmain runが終わった後の15:36:02に追加された。

したがって、agent自身の最終応答だけでは、**その応答を返すための外側のfinalizerで起きるコンパクション**を検知できない。外部watcherが必要である。

<br>

<hr>

<br>

## 7. 第二の問題：1M表示と200k級runtime windowの不整合

### 7.1 表示・設定側

試験中の`session_status`は次を表示した。

```text
Context: 169k / 1.0m (17%)
Compactions: 38
```

設定も、

```text
agents.defaults.contextTokens = 1,000,000
reserveTokensFloor            =   150,000
```

だった。

### 7.2 runtime model側

OpenClawのAnthropic Opus 4.6モデル定義には、`contextWindow=200000`の経路が存在する。さらにOpenClawの関連issueでは、Opus／Sonnet 4.6は明示的な`context1m` opt-inがない場合、200kとして解決される状態が報告されている。

### 7.3 一方向のcontext cap

ソース上、OpenClawは設定budgetを解決した後、次の条件でruntime modelを置き換える。

```text
configured context budget < runtime model.contextWindow
```

つまり、設定budgetがモデルwindowより**小さい**ときはモデルwindowを下げる。

一方、

```text
configured context budget > runtime model.contextWindow
```

のときにモデルwindowを引き上げる処理にはなっていない。

このため、

```text
status / tool / OpenClaw budget       1,000,000
embedded runtime model.contextWindow    200,000
```

という分離が起こり得る。

### 7.4 reserveの適用先

compaction settings managerには、設定側で解決した`contextTokenBudget`が渡される。この値が1Mであれば、`reserveTokensFloor=150000`は小型window向けcapを受けず、そのまま有効reserveになり得る。

一方、embedded runtimeの自動コンパクションはruntime modelの`contextWindow`を参照する。

したがって、次の組み合わせが成立し得る。

```text
OpenClaw側contextTokenBudget          1,000,000
runtime model.contextWindow             200,000
effective reserveTokens                 150,000
```

### 7.5 実質閾値

一般的なauto-compaction判定:

```text
current context tokens
>
model.contextWindow - reserveTokens
```

上の値を代入すると、

```text
200,000 - 150,000 = 50,000
```

となる。

39回目のlive context:

```text
167,587 > 50,000
```

なので、turn終了時に自動コンパクションする。

これは7月3日の、

```text
89,981
91,110
91,800
92,411
93,532
94,027
```

での反復とも整合する。コンパクション後のsummary＋bootstrapだけでも50kを超えていれば、外部CLIがmainを開くたびに再コンパクションし得る。

<br>

<hr>

<br>

## 8. 現時点で確定したこと

1. 外部timer／bridgeはコンパクションを直接命令せず、main turnを起動する契機だった。
2. 過去の多数のtimeout試行では`trigger=cli_budget`が直接記録されていた。
3. OpenClaw 2026.6.6には累積Anthropic cache usageの過大計上問題が存在した可能性が極めて高い。
4. PR #99864は2026.7.1に含まれている。
5. 39回目では、累積usage 501,069ではなく、最新callのlive context 167,587が`tokensBefore`に使われた。
6. したがって、PR #99864が修正したsnapshot選択は39回目で機能していた。
7. それでも167,587でコンパクションしたため、別の閾値不整合が残っていた。
8. 39回目はmain model run終了後、応答配送前のfinalization中に起きた。
9. statusは1Mを表示していた。
10. 200k runtime windowと150k reserveが同時に使われた場合、実質閾値50kとなり、観測結果を一貫して説明できる。
11. external watcherは正しく動作した。
12. D自身の「発動なし」という応答では、外側のfinalizer compactionを検知できなかった。

<br>

<hr>

<br>


## 9. まだ直接確認できていない一点

39回目のrunについて、次の内部値はログに直接出ていない。

```text
active runtime model.contextWindow
settingsManager.getCompactionReserveTokens()
```

したがって、

```text
200,000 - 150,000 = 50,000
```

は、ソース・設定・時系列・実測値を統合した**高確度の原因推定**であり、同一runの内部変数を直接dumpした確定値ではない。

最終確定には、main sessionを開かない読み取り、または次回run前の計測instrumentationで次を取得する必要がある。

- active provider/model definition
- active `model.contextWindow`
- resolved `contextTokenBudget`
- context-window source
- effective reserve
- auto-compaction trigger phase

ただし、追加の無監視再現試験を行う必要はない。今回の一回で運用上の再現は十分確認できている。

<br>

<hr>

<br>

## 10. 現在の運用対策

| 経路 | 状態 |
|---|---|
| ambient timer | disabled / inactive |
| daily-main-tick timer | disabled / inactive |
| agent間bridge timers | disabled / inactive |
| health-check timer | enabled / active |
| health-checkからD mainへの通知 | 無効化済み |
| compaction watcher | enabled / active |
| watcher通知 | ホストから利用者へ直接送信。main sessionは開かない |

現時点では次を維持する。

- 自動的に長期main sessionを開くtimer／bridgeを再開しない
- external watcherを維持する
- health-checkはmainを呼び出さない
- reserve floorを原因未確定のまま安易に下げない
- auto-compactionを無条件に無効化しない
- 追加試験より先にcontext windowの解決経路を確認する

<br>

<hr>

<br>

## 11. 修正方針

根本的には、次の三つを同じ値へ揃える必要がある。

```text
1. Providerが実際に利用できるcontext window
2. OpenClawの表示・contextTokenBudget
3. Embedded runtime／SDK model.contextWindow
```

### 実際に1Mを利用できる場合

- Anthropic Opus 4.6のruntime modelを1Mとして解決する
- 必要な`context1m` opt-in／provider model definitionを正しく設定する
- embedded runtimeへ1M model objectを渡す
- reserve 150kと組み合わせ、約850kを閾値にする

### 実際の利用可能windowが200kの場合

- statusと設定も200kへ合わせる
- reserve 150kのままでは閾値50kになるため、200k windowに適したreserveへ調整する
- 「1M表示だが実際は200k」という状態をなくす

### automation設計

mainに定時刺激を与えたい場合も、長期mainへ直接CLI turnを投入する方式だけでなく、次を検討する。

- isolated sessionで処理し、必要な結果だけ保存
- ホストから利用者へ直接通知
- ファイル・queueへ置き、利用者または通常会話時にmainが読む
- mainへの投入回数を意図的に制限する

<br>

<hr>

<br>

## 12. 確度表

| 判断 | 確度 |
|---|---|
| 外部timer・bridgeがmain turnを開始していた | 確定 |
| 過去の多数の失敗試行のtriggerが`cli_budget` | 確定 |
| 2026.6.6で累積Anthropic cache usage問題が関与 | 非常に高い |
| PR #99864が2026.7.1へ入った | 確定 |
| 39回目でlatest-call snapshotが使われた | 確定 |
| 39回目で累積usage過大計上が原因だった | 否定 |
| 39回目がmain run終了後のfinalization中に起きた | 非常に高い |
| 1M表示と200k級runtime windowの分離が関与 | 高い |
| effective reserveが150kだった | 高いが同一run内変数の直接dumpなし |
| 実質閾値が約50kだった | 高いが上記二値からの推定 |
| OpenClaw 2026.7.1で問題全体が解決した | 否定 |
| PR #99864が無効だった | 否定。39回目では機能していた |

<br>

<hr>

<br>

## 13. 暫定結論

今回の頻発コンパクションは、一つの原因だけではなく、少なくとも二つの問題を分けて考える必要がある。

### 問題1：累積cache usageの誤計上

OpenClaw 2026.6.6では、長いAnthropic cached tool loopの累積usageがlive contextとして扱われ、`cli_budget`を早期に起動した可能性が極めて高い。これはPR #99864で修正され、39回目のrunでは実際にlatest-call snapshotが使われていた。

### 問題2：context windowとreserveの不整合

更新後も、設定・statusは1Mを示す一方、embedded runtimeが200k級のmodel contextを参照し、そこへ1M設定を前提にした150k reserveが適用される可能性が残っていた。

この場合の実質閾値は約50kであり、39回目の167,587、7月3日の約90k〜110k、外部CLI turnのたびの反復を一貫して説明できる。

現時点で最も正確な公開用結論は次のとおりである。

> **OpenClaw 2026.7.1のPR #99864は、39回目の制御試験では正しく機能し、累積Anthropic cache usageではなく最新callのlive contextがコンパクションへ渡されていた。しかし、設定・status上の1M context budgetと、embedded runtimeが参照する200k級のmodel context、さらに150kのreserveが一致していない可能性が高く、turn終了時の自動コンパクション閾値が実質約50kまで縮んでいたと考えられる。したがって、更新で第一のバグは改善したが、外部`openclaw agent`から長期main sessionを開く際の早期コンパクション問題全体は未解決だった。**

<br>

<hr>

<br>

**第5版追記／2026年7月22日:** 現在のsession・JSONL・workspaceを維持したまま、OpenClaw実行環境だけを2026.7.1から、更新前に実際に使用していた2026.6.6へロールバックした。ロールバック前には、現在の設定を6.6で検証し、凍結JSONLを6.6のSessionManagerで読み取り専用プローブした。最新leaf、184 messagesのcontext、context末尾はいずれも2026.7.1でのプローブ結果と一致した。

## 14. 2026年7月22日の介入試験

OpenClaw 2026.6.6への環境ロールバック後も、次の設定を維持した。

```text
model             anthropic/claude-opus-4-6
thinkingDefault   off
contextTokens     1,000,000
context1m         true
compactionCount   39
```

再開後、session statusは約172k / 1.0Mを示したが、compactionCountは39のままで、新しいcompactionは外部watcherにも検知されなかった。

39回目の発火値は167,587 tokensだった。今回、それを超える約172kまでcontextが伸びてもコンパクションが起きなかったことから、以前の実質約50kと推定された異常閾値は、現在の2026.6.6＋`context1m=true`構成では解消していると判断できる。少なくとも、自動コンパクション閾値が172kより上へ移動したことの強い挙動証拠である。

この結果は、問題2の中心がOpenClawのバージョンそのものではなく、Opus 4.6が1M runtime modelとして解決されていなかったことにある、という第4版の原因推定を支持する。

ただし、今回の観測は1M context全域の利用可能性、または自動コンパクション閾値が正確に850kであることを直接証明するものではない。active runtime `model.contextWindow`およびeffective reserveの同一run内dumpは、引き続き未取得である。

一方、2026.6.6にはPR #99864が含まれていない。そのため、長いAnthropic cached tool loopにおける累積cache usage過大計上の再発可能性は残る。今後も、tool-use turnのprompt footprint、call数、compactionCount、外部watcher通知を監視する。

初回のtool-use回帰試験では、readツール実行後の自然言語回答は一通だけで完結した。2026.7.1で観測された複数の途中回答、後になるほど薄くなる言い直し、同一回答の二重表示は発生しなかった。

また、番号付きの古いTelegram発言が順次見えなくなる現象は、直近最大20件の補助履歴窓が件数ベースで前進する正常動作と判明した。これはcanonical transcriptの削除、cache-ttlによる会話削除、または新しいcompactionではない。

### 2026年7月22日時点の更新結論

今回の調査で分離した二問題の現在状況は次のとおりである。

1. **累積cache usageの過大計上:** 2026.6.6へ戻したため、長いcached tool loopでの再発リスクは残る。継続監視が必要。
2. **context windowとreserveの不整合:** `context1m=true`を維持した2026.6.6環境で、以前の発火値167,587を超える約172kまでコンパクションなしで動作した。旧来の異常閾値は現在構成で運用上解消したことを示す強い証拠が得られた。

したがって、第4版の7月20日時点の原因分析は維持されるが、現在の運用状態は「問題全体が未解決」ではない。より正確には、**問題2は現在構成で解消した強い挙動証拠が得られ、問題1の再発可能性だけを監視している状態**である。

<br>

<hr>

<br>



## 参考資料

1. OpenClaw v2026.7.1 release notes  
   https://github.com/openclaw/openclaw/releases/tag/v2026.7.1
2. OpenClaw PR #99864 — `fix(compaction): avoid cached usage overcount`  
   https://github.com/openclaw/openclaw/pull/99864
3. OpenClaw Issue #49939 — Opus／Sonnet 4.6の200k／1M解決  
   https://github.com/openclaw/openclaw/issues/49939
4. OpenClaw Issue #24031 — configured context budgetとruntime `model.contextWindow`の分離  
   https://github.com/openclaw/openclaw/issues/24031
5. OpenClaw Issue #108238 — 2026.7.1で報告された別経路の累積usage問題  
   https://github.com/openclaw/openclaw/issues/108238
6. ローカル監査ログ `cli-budget-source-audit-20260720-103100.txt`（非公開運用資料）
7. 制御試験結果 `openclaw-daily-main-tick-test-20260720/result.txt`（非公開運用資料）
8. 39回目監査 `openclaw-compaction-39-audit-20260720-161809.tar.gz`（非公開運用資料）

<br>
