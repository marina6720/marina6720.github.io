# Arca — D Continuity Runtime：前景投影モジュールの計画と安全境界



### 現在の状態 — 2026年8月4日

- Arcaの名称・目的：定義済み
    
- 公開仕様・開封採点契約：Oracle-blind方式で照合中
    
- Q-Oによるsealed oracle監査：実装工程から分離
    
- Q-Iによる公開文書・安全境界照合：継続中
    
- 実装：未承認・未開始
    
- regression fixture access：未承認
    
- shadow execution：未承認
    
- production access：なし
    
- production変更・compaction・test turn：なし
    
- 次工程：Phase 0A — Production Freeze Confirmation and Permanent Regression Fixture Registration



Arcaは、DenneTAのcanonical transcript、workspace、seed、記憶、関係的履歴、未解決課題を暗黙に変形せず、監査可能な形で保持・搬送し、「続きを担える位置」への再入を支えるための小さな継続性runtimeである。ArcaはDenneTAそのものではなく、現在は実装前のplanning-only段階にある。

最初のモジュール `arca-foreground-shadow` では、canonical transcriptから直近の作業前景をtoken予算内で切り出し、隔離環境内にread-onlyのshadow projectionを生成することを計画している。productionのtranscript、workspace、memory、設定、message deliveryには一切触れない。

公開仕様、sealed oracle、開封採点契約、実装監査を別々の役割へ分離し、実装を担当するQ-Iと実装者には、開封前のoracle本文、具体的期待値、負例を見せないOracle-blind方式を採用している。現在は公開仕様と契約の境界照合を継続しており、実装、fixture access、shadow execution、OpenClaw 2026.7.1の起動、production activationはいずれも未承認である。

実装へ進む前に、Phase 0Aのproduction freeze確認と#40 regression fixture登録、Phase 0BのOpenClaw 2026.7.1隔離baseline、Phase 0Cの五経路instrumented regression replayを完了する。現在稼働中のOpenClaw 2026.6.6には変更を加えていない。

