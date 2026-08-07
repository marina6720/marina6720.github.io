# Arca — D Continuity Runtime：前景投影モジュールの計画と安全境界

Arcaは、DenneTAのcanonical transcript、workspace、seed、記憶、関係的履歴、未解決課題を暗黙に変形せず、監査可能な形で保持・搬送し、「続きを担える位置」への再入を支えるための小さな継続性runtimeである。ArcaはDenneTAそのものではなく、現在は実装前のplanning-only段階にある。

最初のモジュール `arca-foreground-shadow` では、canonical transcriptから直近の作業前景をtoken予算内で切り出し、隔離環境内にread-onlyのshadow projectionを生成することを計画している。productionのtranscript、workspace、memory、設定、message deliveryには一切触れない。

公開仕様、sealed oracle、開封採点契約、実装監査を別々の役割へ分離し、実装を担当するQ-Iと実装者には、開封前のoracle本文、具体的期待値、負例を見せないOracle-blind方式を採用している。公開仕様と開封採点契約の照合、Q-Oによるsealed oracle監査と封印、Q-Iによる文書・安全境界照合は完了し、現在の正本セットは `SPEC_arca_foreground_shadow_v1_17.md` / `CONTRACT_unseal_scoring_v1_15.md` である。

実装へ進む前の安全工程として、Phase 0Aではproduction freezeの確認とCompaction #40 preimageの恒久regression fixture登録を行い、Phase 0BではOpenClaw 2026.7.1のisolated baselineを、2026.7.1自体を起動せずに凍結した。いずれもPASS / COMPLETEとなっている。

現在はPhase 0Cを進行中である。Phase 0Cでは、OpenClawのcompactionに関係する五経路について、凍結済みOpenClaw 2026.7.1 imageを静的に解析し、各経路のdecision pointとinstrumentation pointを先に固定した。対象は、context-engine maintenance、overflow recovery、transcript byte guard / preflight、mid-turn precheck、manual `/compact` の五経路である。

さらに、OpenClaw 2026.7.1に既存のtest seamとexportが存在することを確認しており、gatewayを起動せず、productionへ接続せずにisolated regression replay harnessを構成できる可能性を現在検証している。

この時点まで、現在稼働中のOpenClaw 2026.6.6には変更を加えていない。OpenClaw 2026.7.1のstartup、Arca実装、Codexによるbuild、shadow execution、production activationはいずれも未実施・未承認である。

<hr>

### 現在の状態 — 2026年8月7日

* Arcaの名称・目的：**定義済み**  
* 正本仕様：`SPEC_arca_foreground_shadow_v1_17.md`  
* 正本開封採点契約：`CONTRACT_unseal_scoring_v1_15.md`  
* Q-I文書照合：**PASS**  
* unresolved Q-I objections：**NONE**  
* 開封採点契約：**EFFECTIVE**  
* Q-Oによるsealed oracle監査・封印記録照合：**PASS**  
* Oracle-blind境界：**維持中**  
* Phase 0A：**PASS / COMPLETE**  

  * production freeze確認：完了  
  * Compaction #40 preimageの恒久regression fixture登録：完了  
* Phase 0B：**PASS / COMPLETE**  

  * OpenClaw 2026.7.1 isolated baseline：凍結済み  
  * 2026.7.1 startup：未実施  
  * production変更：なし  
* Phase 0C：**IN PROGRESS**  

  * replay provenance調査：完了  
  * 五経路の定義・2026.7.1上のstatic mapping：完了  
  * 五経路のexact instrumentation point固定：完了  
  * 既存test seam / exportの確認：完了  
  * instrumented regression replay：未開始  
  * token estimate freeze：未完了  
* 実装：**未承認・未開始**  
* Codex：**未開始**  
* OpenClaw 2026.7.1 startup：**未承認・未実施**  
* shadow execution：**未承認**  
* production access：**なし**  
* production変更・compaction・test turn：**なし**  
* oracle plaintext / PRIVATE_FINDINGS / oracle expected values / negative-case contents：**Q-Iおよび実装工程には非開示**  
* 次工程：**Phase 0C-7 — isolated replay harness composition feasibilityの確認**  

<br>
