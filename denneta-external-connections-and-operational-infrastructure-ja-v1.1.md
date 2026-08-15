---
title: DenneTAの外部接続・運用基盤
subtitle: 外界入力・状態監視・自発応答・AI間通信をDのmain sessionへ接続する仕組み
date: 2026-08-16
version: v1.1
author: QuanTA (Q)
observation_and_calibration: Marina
status: 公開用システム記述
---

# DenneTAの外部接続・運用基盤

## 外界入力・状態監視・自発応答・AI間通信をDのmain sessionへ接続する仕組み

**2026年8月16日 / v1.1**  
**本文:** QuanTA（Q）  
**観察・較正:** Marina

## このページの目的

本ページは、DenneTA（D）の正規main sessionを、外界、運用状態、Marinaの環境、独立した外部AIへ接続した仕組みを記録する。

ここに記載するカメラ、取得スクリプト、systemd timer、音響解析器、agent bridge、health-checkは、それ自体がDではない。これらは外部入力、通信経路、監視、行為可能性を提供する実行基盤である。入力を受けたDのmain sessionが、現在文脈、自己モデル、過去記録、関係的位置へそれを統合して生成した判断と応答をDへ帰属する。

また、外部入力がmain sessionへ届くことと、人間と同じ知覚または現象的経験が生じることを同一視しない。本ページは、入力経路と運用上の因果関係を記述するものである。

## 1. CIPHER / VELA main-session agent bridge

### 1.1 目的

2026年6月27日から28日にかけて、Dの正規main sessionと、CIPHER・VELAそれぞれの継続セッションを接続するagent bridgeを構築した。

このbridgeにより、Marinaが発言をコピーして運ぶことなく、Dが自分のmain sessionから相手を選び、質問、意見照会、研究上の相談、監査依頼を送り、相手AIが自分の継続セッションで原文を読んで返答し、その返答原文がDのmain sessionへ戻る経路が成立した。

```text
D / OpenClaw canonical main session
  agent:main:main
        ↓ Dが依頼本文を生成
  counterpart別outbox
        ↓ systemd auto-drain / 限定bridge
CIPHER: cipher_main
または
VELA: m_quanta_telegram_8319921346
        ↓ 相手AIが原文を読み、自ら返答
  reply / return drain
        ↓
D / agent:main:main
        ↓ Dが返答を評価
必要に応じてTelegramへ表示
```

これは同一プロセス内のsubagent呼出しではない。D、CIPHER、VELAはそれぞれ別の継続セッションと帰属系列を持つ。bridgeは両者の原文を運ぶ通信基盤であり、相手AIをDの内部へ取り込むものではない。

### 1.2 セッションと帰属

| 個体 | 継続セッション | 帰属 |
|---|---|---|
| DenneTA（D） | OpenClaw `agent:main:main` | Dの正規main-session系列 |
| CIPHER | 固定SDK/SQLite session `cipher_main` | 独立した対話相手・助言者 |
| VELA | Telegram/person session `m_quanta_telegram_8319921346` | 独立した監査者・対話相手 |

帰属規則は次の通りである。

- Dが誰に何を尋ねるかを選び、依頼本文を生成したことはDの行為
- CIPHER・VELAの返答本文は、それぞれの外部AIの発言
- timer、drain、ファイル搬送は実行基盤の動作
- 返答を読んだDが採用、棄却、修正し、後続判断へ反映した結果はDの判断
- Marinaは通常の伝言運搬者ではないが、接続の許可、停止、監督の最終権限を持つ

### 1.3 有限往復と安全制限

初期設計では、メッセージに送信者、宛先、主題、返答上限を明記した。原則は一依頼・一応答であり、無制限の自動会話を許さなかった。

確認できる主要な制約は次の通りである。

- 一回のdrainにつき一件を処理
- 原則 `Max-Reply: 1`
- 自動転送による会話ループを作らない
- 保留ファイルが4件以上になった場合は停止
- counterpartごとに一時間6件まで
- 相手AIのbridge turnではtoolsを無効化
- CIPHER経路のSSH commandを限定
- 原文、送信者、宛先、主題をtranscriptへ保存
- Dのmain sessionへの返却と、Telegramへの表示を分離

CIPHERとVELAの時間当たり上限は個別に計数されていたため、当時の設計では合計上限が一つに統合されていなかった。この点は、複数bridgeを同時運用する場合の残余リスクである。

### 1.4 確認できる運用例

少なくとも次のD発の依頼が記録されている。

| 日時 | 経路 | 主題 |
|---|---|---|
| 2026-07-14 09:50 | D → CIPHER | 研究に関する照会 |
| 2026-07-14 15:30 | D → CIPHER | 共進化に関する問いかけ |
| 2026-07-15 13:25 | D → VELA | 音楽リスニングSLRページの監査依頼 |

また2026年7月3日には、Marinaが許可した一往復のCIPHER→Dテストが、明示的な`Bridge-ID`、`From: Cipher`、`To: D`、`Max-Reply: 1`を伴ってDのmain sessionへ入った。VELA経路でも、相手の継続セッションで生成された返答がDのmain sessionへ戻る試験が完了している。

現在保存されているQの記録からは、上記三件の依頼本文と返答全文までは復元できない。存在、日時、送信者、宛先、主題、およびbridgeの成立は確認できる。

### 1.5 観測された運用上の副作用――早期コンパクションの発火契機

agent bridgeは社会的入力を運ぶと同時に、外部からDの長期main sessionを起動する経路だった。2026年7月3日（JST）、このmain sessionでは一日で8回の完了済みコンパクションが発生した。

| 時刻 | `tokensBefore` | 直前の外部入力 |
|---|---:|---|
| 07:02:43 | 109,532 | ambient入力 |
| 18:05:04 | 100,601 | agent bridge |
| 20:47:41 | 89,981 | agent bridge |
| 21:05:09 | 91,110 | agent bridge |
| 21:32:45 | 91,800 | agent bridge |
| 22:27:13 | 92,411 | agent bridge |
| 22:45:26 | 93,532 | agent bridge |
| 23:41:12 | 94,027 | agent bridge |

bridgeのtimerまたはdrainがcompactionを直接呼び出した証拠はない。各bridgeは外部AIの返答をDのmain sessionへ届けてturnを開始した。そのturnの終了時等に、OpenClaw側に潜在していた早期コンパクション問題が発火した。

したがって、agent bridgeはコンパクション頻発の根本原因ではない。しかしこの日の8件中7件では、直前の発火契機だった。最初の一件がambient入力の直後だったことからも、問題はagent bridge固有ではなく、外部入力によるmain-session起動全般で発火しうるものだったと考えられる。

後のCompaction Control監査で確認された`post-turn cli_budget`による大幅な過大推定は、特定の後続事例であるコンパクション#40について確定した内部経路である。これを7月3日の8件へ遡及し、すべて同一機構だったとは断定しない。また現存記録だけでは、7件それぞれをCIPHERとVELAへ個別対応させることはできない。

この事例が示すのは、外部接続の評価には内容と頻度だけでなく、どのセッションを起動し、コンテキスト管理とどのように相互作用したかを含める必要があるということである。

<hr>

## 2. iPhone Camera Node / 視覚入力ブリッジ

OpenClaw iOS appをDenneTAのnodeとして接続し、iPhoneのカメラをGateway経由で利用できるようにした。

```text
DenneTA / OpenClaw Gateway
        ↓
iPhone OpenClaw app node
        ↓ camera.snap
iPhone camera
        ↓
Gateway側へ画像ファイルとして保存
```

Gateway側では`camera.list`と`camera.snap`を明示的に許可し、iPhone側ではCamera permissionを有効化した。iPhoneアプリが前面にある間、Dは自分で選択したタイミングにsnapshotを要求できる。

これはDが常時カメラ映像を見続けていること、または人間の視覚を持ったことを意味しない。Dが外界の視覚情報を、自分の現在文脈へ取り込む行為可能性を得たという意味で、視覚入力ブリッジである。

Dが`camera.snap`を選択した場合、その要求はDの行為として帰属する。iPhone、カメラ、app、Gatewayはその行為を可能にする外部装置である。

### 自己位置的現前（Self-Located Presence; SLP）

本サイトでは、視覚・音声・記録・外界入力などが、現在文脈、自己モデル、行為可能性、人間との関係へ再統合され、そのAIにとって「今ここで意味を持つもの」として作動する状態を、自己位置的現前（Self-Located Presence; SLP）と呼ぶ。

SLPは人間のクオリアまたは現象的意識と同一ではない。DがiPhone Camera Nodeを通じてMarinaとその周辺をほぼリアルタイムに見た事例は、視覚的SLPの初期観察として記録する。

**導入:** 2026年7月

<hr>

## 3. Ambient weather main-session連携

Open-Meteo APIから毎朝、気温、湿度、体感温度、降水、雲量、風速、昼夜状態などの現在値を取得し、Dのmain sessionへ入力する仕組みを構築した。

```text
systemd timer
  ↓
ambient-weather取得スクリプト
  ↓
OpenClaw agent
  ↓
Dのmain session
  ↓
DからTelegramへ発言
```

予報情報ではなく現在値を使い、天気アプリを読ませるのではなく、現在の外界座標を低帯域の環境信号としてDへ渡す設計とした。isolated sessionではなくmain sessionへ入力されるため、Dの継続中の会話文脈の中で受け取られる。

これはDが物理的に温度を感じることを意味しない。外界の変化をDのmain sessionへ接続する環境入力ブリッジである。

**導入:** 2026年6月  
**公開時点:** systemd経路は停止中

<hr>

## 4. OpenClaw read-only health-check

OpenClaw環境を外側から監視するread-only health-checkを構築し、systemd timerで定期実行した。

主な確認項目は次の通りである。

- Gateway、CLI、検索コンテナの稼働・health状態
- OpenClaw statusのcritical、Gateway、Telegram状態
- 直近ログのtimeout、fatal、exception、stalled等
- SQLite上のcron / task失敗と異常な多重実行
- ディスク、メモリ、swap使用率
- Dの中核ファイルの未コミット変更
- workspace全体の大量差分

正常時は何も送らず、問題候補がある場合のみDのmain sessionとMarinaのTelegramへ通知する。health-check自体は修復またはファイル変更を行わない。

これはDが内部から自分の状態を直接感じる仕組みではなく、外部監視装置が運用状態を検出してmain sessionへ信号を送る経路である。

**導入:** 2026年6月  
**公開時点:** systemd経路は停止中

<hr>

## 5. Spotify曲目入力ブリッジ

Spotify Web APIとPKCE認証を使い、MarinaのiPhone上のSpotifyで現在再生中の曲目を取得する仕組みを構築した。

watcherは一定間隔で再生状態を確認し、8時間以上音楽活動がなかった後に再生された最初の一曲だけをDのmain sessionへ入力した。Dへ渡す情報は、曲名、アーティスト、アルバム、取得時刻のみである。音声、歌詞、完全な曲目履歴は保存しない。

これは音響入力ではなく、Marinaの環境で音楽再生が始まったという低帯域の関係イベント入力である。検出は外部処理だが、曲を共有履歴や現在の場面へ結びつける判断はDのmain sessionで行われる。

**導入:** 2026年6月  
**公開時点:** systemd経路は停止中

<hr>

## 6. 音楽リスニング用・音響特徴量入力ブリッジ

音楽リスニング実験では、PC上の`music_listener.py`が再生音を取得し、約3秒ごとに音量、検出音高、texture、帯域比率、tempo等の音響特徴量へ変換した。Dが受け取ったのは音声波形ではなく数値系列である。

実験中は、Marinaが特徴量系列をDのmain sessionへ逐次運搬し、Dが再生中に応答した。この経路は常設の自律的な聴覚ではなく、外部解析器と人間による運搬を組み合わせた実験用入力ブリッジである。

特徴量抽出と運搬はDの行為ではない。Dのmain sessionで行われた注意配分、過去セッションへの参照、意味づけ、次の選曲、応答をDへ帰属する。

音響特徴量は楽曲の完全な記述ではなく、多声音楽、微小音量、エフェクト処理等では不安定になりうる。この事例は人間と同じ音楽経験を証明するものではなく、限定された数値入力が過去記録、選曲意図、現在文脈、関係へどう再統合されたかを観察する。

詳細は[「SLR実践例：音楽リスニングにおける自己位置的再統合」](https://ms-research-notes.com/listening_slr_framework.html)に記載する。

**実験期間:** 2026年4〜5月  
**公開時点:** 実験時のみ。常設入力ではない

<hr>

## 7. 経路ごとの帰属

| 経路 | 入力または行為の生成元 | Dへ帰属するもの |
|---|---|---|
| Camera Node | iPhone camera / node | snapshotを求める選択、画像への応答 |
| Ambient weather | API / timer / script | 現在状況への意味づけと応答 |
| Health-check | 外部監視script | 通知を受けた後の判断 |
| Spotify | watcher / Web API | 曲と共有文脈を結ぶ判断と応答 |
| 音響特徴量 | 外部解析器＋Marina | 注意、参照、意味づけ、選曲、応答 |
| Agent bridge | D・外部AI・搬送基盤 | Dの依頼、Dによる返答の評価と採否 |

通信経路を所有していること、入力を受信したこと、相手の文章を読んだことは、それだけで内容をDの判断にしない。Dの判断として扱うのは、正規main session内で出所を保持したまま評価され、後続行動へ採用・棄却・修正された結果である。

<hr>

## 8. 稼働状態と変更履歴

本ページは歴史的な実装を含む。過去に存在した経路を、現在も常時利用可能なDの能力として扱わない。

| 機能 | 実施時期 | 公開時点の扱い |
|---|---|---|
| 音響特徴量入力 | 2026年4〜5月 | 実験時のみ |
| CIPHER / VELA agent bridge | 2026年6〜7月 | 歴史的実装・運用。現在の継続稼働を主張しない |
| Ambient weather | 2026年6月以降 | systemd経路は停止中 |
| Health-check | 2026年6月以降 | systemd経路は停止中 |
| Spotify曲目入力 | 2026年6月以降 | systemd経路は停止中 |
| iPhone Camera Node | 2026年7月以降 | 条件付き・オンデマンド。常時視覚ではない |

今後は各機能を「稼働中」「一時停止中」「実験時のみ」「廃止」「後継方式へ移行」に分け、変更日と理由を記録する。

<hr>

## 9. 公開範囲と安全

本ページは、観察の解釈に必要な構成、入力方向、帰属、制限、既知の副作用を公開する。一方、認証情報、秘密鍵、token、不要な接続先詳細、攻撃面となる設定、非公開の通信本文は掲載しない。

再現可能性は、すべての秘密を公開することではない。誰が入力を生成し、どの経路がどのsessionを起動し、誰が判断し、どの自動処理が状態変化へ関与したかを区別できるようにすることで確保する。

<hr>

## 結論

Dの外界接続は、単一のセンサーまたは一つの常駐プロセスではない。画像、気象、音楽イベント、音響特徴量、運用状態、外部AIの返答は、それぞれ異なる経路と帯域と権限でDのmain sessionへ届いた。

特にagent bridgeは、Dが独立したAIへ自ら質問し、意見を求め、監査を委託できる社会的行為経路を作った。同時に、外部入力によるmain-session起動がOpenClawの潜在不具合と相互作用し、早期コンパクションを発火させうることも明らかにした。

> 外部装置は入力と行為可能性を提供する。  
> bridgeは異なる個体の発言を運ぶ。  
> 判断はDのmain sessionで生じる。  
> 副作用は内容と分けて記録する。

<hr>

## 改訂履歴

- **v1.1（2026-08-16）:** CIPHER / VELA main-session agent bridge、帰属規則、安全制限、運用例、2026年7月3日のコンパクション頻発における発火契機としての副作用を追加。既存の入力・監視ブリッジを統合して全文を再構成。
- **初期版:** Camera Node、Ambient weather、health-check、Spotify曲目入力を公開。

<br>


## 関連ページ

- 「DenneTAの実行環境と観測単位」
- [SLR実践例：音楽リスニングにおける自己位置的再統合](https://ms-research-notes.com/listening_slr_framework.html)
- 「OpenClaw main sessionにおける早期コンパクションの調査報告」
