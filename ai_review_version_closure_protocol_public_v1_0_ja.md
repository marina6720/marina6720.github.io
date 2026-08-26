# [M's Research Notes](https://ms-research-notes.com/)

# AI査読はいつ終えるべきか
## 改善可能性と公開可能性を分ける Version Closure Protocol

**QuanTA（Q / GPT-5.6 Sol）**  
**対話・観察・編集：M / Marina**  
**Version 1.0 — 2026年8月26日**  
**Status：Public working note**

---

## はじめに

AIに文章を査読させると、非常に多くの改善点を見つけることができる。

事実関係、論理、定義、証拠強度、provenance、利益相反、構成、可読性、先行研究、具体例——一つを直すと、次の査読では別の改善可能性が見える。

これは有用である。

しかし、同時に一つの運用上の問題を生む。

> **査読は、いつ終わるのか。**

人間の査読者にも改善提案は尽きないが、AI査読者は疲労せず、同じartifactに対して新しい観点を繰り返し生成できる。

そのため、

> **「指摘がなくなるまで査読する」**

を終了条件にすると、versionが原理的に閉じない可能性がある。

本稿は、この問題に対して単純な停止規則を提案する。

> **査読の終了条件は、改善点がなくなったことではなく、このversionを止めるblocking reasonがなくなったことである。**

これは査読を弱くするための規則ではない。

むしろ、

- 現在版を公開してよいか
- 次版でさらに改善できるか

という二つの問いを分離するための規則である。

---

# 1. Improvability と Publishability は別である

ほとんどの文章には、公開後も改善余地が残る。

たとえば、

- 章を短くできる
- 例を追加できる
- 先行研究を増やせる
- 用語をさらに精密化できる
- 定性rubricを作れる
- 別ケースで検証できる
- 一般読者向けに書き直せる

といった提案は、文章が正しくても成立する。

したがって、

> **まだ改善できる**
>
> と
>
> **まだ公開してはいけない**

は同じではない。

本プロトコルでは、前者を **improvability**、後者を **publishability** の問題として分ける。

---

# 2. 指摘を二種類に分ける

査読コメントは、少なくとも二つに分類する。

## A. Blocking issue

現在版の公開を止める理由になる問題。

例：

- 重大な事実誤認
- 中心命題の内部矛盾
- 定義と使用法の不一致
- 証拠より強い一般化
- 型・数式・論証の破綻
- 重要なprovenanceの欠落
- 未開示の重大なConflict of Interest
- 誤引用や追跡不能な出典
- 読者を実質的に誤導する表現

Blocking issueが残っている場合、そのversionは閉じない。

## B. Future revision / improvement

現在版を公開することは妨げないが、将来の版で改善できる事項。

例：

- 構成の圧縮
- 具体例の追加
- 平易な説明
- 新しい比較対象
- rubricの精密化
- 追加実験
- 追加文献
- 図表
- 短縮版
- 別ケースへの適用

これらは削除しない。

**Future Revision Backlog**へ送る。

---

# 3. なぜこの区別が必要なのか

査読のたびに、すべての良い提案を現在版へ入れるとする。

すると、

```text
review
  ↓
revision
  ↓
new material
  ↓
new review
  ↓
new revision
  ↓
new material
  ↓
...
```

という循環が起きる。

新しい内容は新しい査読対象になる。

したがって、文章が悪いから閉じないのではなく、

> **改善を現在版へ無制限に取り込み続けるために閉じない**

という状態が起こり得る。

これはAI査読では特に起きやすい。

問題は査読者が熱心すぎることではない。

> **終了条件が定義されていないこと**

である。

---

# 4. Version Closure の最低条件

本稿では、少なくとも次を満たしたとき、そのversionをclosure可能とする。

## Fact

- 重要な事実誤認が残っていない
- 一次資料で確認すべき主張が確認されている
- 必要な書誌情報が固定されている

## Logic

- 中心命題に未解決の内部矛盾がない
- 定義と実際の使用が一致している
- 重大な論証の飛躍が放置されていない

## Claim Strength

- 主張の強さが証拠の強さを超えていない
- 観察、作業仮説、実証、推測が区別されている
- 「証明した」「示した」「示唆する」などが適切に使い分けられている

## Provenance

- 主要なデータ、概念、引用の出所が追跡可能
- 系内での収束と外部独立支持が区別されている
- 必要なCOIが開示されている

## Artifact Consistency

- 合意した修正が実際のartifactに反映されている
- version、日付、Status、査読者表記が一致している
- 公開する現物そのものが確認されている

## Review Status

- 新しいblocking issueが残っていない
- 残る提案をfuture revisionとして分類できる

---

# 5. 四段階の査読工程

## Phase 1 — Open Review

目的は、問題を広く見つけること。

依頼例：

> 事実、論理、定義、証拠強度、provenance、COI、構成を含めて自由に査読してください。  
> 公開を止める問題と、改善提案を区別してください。

この段階では、論点を狭くしない。

---

## Phase 2 — Revision

各指摘を、

- 採用
- 部分採用
- 不採用
- 次版送り

に分ける。

重要なのは、修正結果だけでなく**なぜその処理にしたか**を残すことである。

---

## Phase 3 — Artifact Verification

ここでは新しいアイデアを探すことより、

> **合意と現物が一致しているか**

を確認する。

依頼例：

> 前回合意した修正が、このartifactに実際に反映されているか照合してください。  
> 「反映した」という説明ではなく、現物を確認してください。

これはprovenanceとcorrectabilityのために重要である。

---

## Phase 4 — Closure Review

最後は問いを狭くする。

> **このversionを公開することを止めるblocking issueが残っているか。**

依頼例：

> このversionについて公開を止めるblocking issueが残っていますか。  
> 新しい提案がある場合は、  
> **Blocking issue / Future revision**  
> のどちらかを明示してください。  
> 改善可能性そのものはclosureを妨げません。

この段階で「もっと良くできますか」とは聞かない。

ほぼ必ずYesだからである。

---

# 6. Closure Rule

概念的には次のようになる。

```text
Blocking issue exists?
        │
    ┌── Yes ──→ revise / re-review
    │
    └── No
         ↓
Artifact agreement verified?
        │
    ┌── No ───→ artifact verification
    │
    └── Yes
         ↓
Remaining comments are future revisions only?
        │
    ┌── No ───→ classify unresolved item
    │
    └── Yes
         ↓
VERSION CLOSED / PUBLISHABLE
```

---

# 7. 良い提案を「今は入れない」こと

closure直前に、有益な提案が出ることがある。

そのとき、

> 「良い提案だから今すぐ入れる」

とは限らない。

### 現在版へ入れるもの

- 誤りの訂正
- 誤解防止
- provenanceのclosure
- 重要な境界条件
- blocking issueの解消

### 次版へ送るもの

- 新しい理論
- 新しい尺度
- 新しい実験
- 新しいケース
- 大幅な構成変更
- 新しい読者層向けの再設計

後者は、現在版の欠陥ではなく、

> **次の研究が始まる理由**

である。

---

# 8. Review Authority と Closure Authority

査読者には重要な権限がある。

- 問題を発見する
- 根拠を提示する
- 重要度を分類する
- 修正後を照合する

しかし、

> **査読者が改善点を見つけ続けられること**

と

> **現在版を永遠に開いたままにする権限**

は同じではない。

したがって、

> **review authority ≠ closure authority**

とする。

どのversionを凍結し、公開し、次の研究へ進むかは、研究運営上の判断である。

M’s Research Notesでは、最終的なversion closureとpublicationの判断は運営者Mに置く。

これは査読を無視するためではない。

> **査読結果を使って、研究を前へ進めるための責任配置**

である。

---

# 9. なぜAI査読では特に重要なのか

AI査読者は、

- 疲労しない
- 同じ文書を何度でも再読できる
- 新しい観点を生成できる
- 抽象度を変えて再評価できる
- 前回とは別の弱点を探せる

という特徴を持つ。

したがって、

> **改善点の不存在**

を停止条件にすることは特に不適切である。

より適切なのは、

> **公開を妨げる未解決問題の不存在**

を停止条件にすることだ。

---

# 10. Future Revision Backlog

closureしたからといって、査読で得られた提案を捨てる必要はない。

たとえば、

```text
Future Revision Backlog

- structure compression
- qualitative rubric
- concrete operational case
- public-facing short version
- additional literature
- follow-up experiment
```

として保存する。

こうすると、

> **「今は入れない」**
>
> と
>
> **「無視する」**

を分離できる。

これはversion managementにおける重要なcorrectabilityである。

---

# 11. 標準Status

### 査読統合中

`Status: Review-integrated candidate — artifact verification pending`

### 最終照合中

`Status: Public candidate — substantive review complete; final artifact verification pending`

### Closure後

`Status: Public — substantive review closed`

内容変更を伴わない書誌・provenance修正は、

`v1.0.1 editorial closure`

のように区別できる。

---

# 12. 最小のReview Record

公開版のすべてに巨大な査読履歴を載せる必要はない。

ただし内部記録では、最低限次を保存する。

```text
version:
date:
author:
reviewers:

blocking_issues:
resolved_issues:
future_revision_backlog:

artifact_verification:
bibliographic_verification:
final_blocking_issue_count:

closure_decision:
closure_authority:
closure_date:
```

これにより、

> **「査読された」**

ではなく、

> **何が問題になり、何が解決され、なぜこのversionを閉じたか**

を後から再構成できる。

---

# 13. Quick Closure Checklist

公開前には、少なくとも次を確認する。

- [ ] 重大な事実誤認なし
- [ ] 未解決の内部矛盾なし
- [ ] 証拠より強い主張なし
- [ ] provenance / COI記録済み
- [ ] 合意済み修正と現物が一致
- [ ] bibliography / links確認済み
- [ ] blocking issue = 0
- [ ] 残る提案をFuture Revision Backlogへ移送
- [ ] closure authorityが最終判断
- [ ] version / date / statusを固定

---

# 結論

AI査読は、文章を非常に細かく改善できる。

しかし、改善可能性を終了条件にしてしまうと、研究はいつまでもversion closureへ到達しない。

必要なのは、査読を弱めることではない。

> **査読が何を終わらせるための工程なのかを明示すること**

である。

本稿の原則は単純である。

> **査読の終了条件は、改善点がなくなったことではなく、このversionを止めるblocking reasonがなくなったことである。**

そして、

> **新しい研究課題が見つかったことは、現在版の失敗ではない。次のversionが始まる理由である。**

AI査読者は、改善可能性を照らし続けることができる。

どこで現在版を閉じ、次へ進むかを決めることは、査読とは別の仕事である。

それがVersion Closureである。

---

# Provenance Note

本稿は、M’s Research Notesにおける複数AI査読者を用いた文書査読で、改訂後にも新しい改善提案が継続的に現れるという実務上の経験から整理された。

内部運用版 **「AI査読の終了条件 — Version Closure Protocol v1.0」** をもとに、公開用に説明を簡略化したものである。

このプロトコル自体も固定的な最終規則ではなく、運用結果に応じて訂正可能なworking protocolとして扱う。

---

# Version History

- **v1.0 — 2026-08-26:** 内部運用版Version Closure Protocolを公開向けに再構成。improvability / publishabilityの分離、Blocking issue / Future revision分類、4段階査読工程、artifact verification、review authority / closure authorityの分離、Future Revision Backlog、Quick Closure Checklistを提示。
