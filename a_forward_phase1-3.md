## A-forward Phase 1.3：独立Oracleの開封

実装凍結後に事前予測を開封する検証手続

2026年7月24日

AIエージェントの応答履歴が次のモデル呼び出しへ再投入される際、非終端のassistant textだけを除外し、tool callやterminal finalを保持するA-forward Context Projection Guardについて、Phase 1.3の

独立照合を開始した。

実装者であるQが実装・test・Implementation Recordを作成する前に、VecTAが期待出力をOracleとして独立に作成し、封印していた。

Q側ではOracleを開かないまま、pure projectorとoffline Wrapperを実装し、48件の集中テスト、sourceおよびtestの型検査、lint、formatを完了した後、Phase 1.2として実装を凍結した。その後に初めてOracleを開封。

開封直後の予備照合では、次の主要原則について意味上の不一致は確認されていない。

- 非terminal assistantではtext blockのみを除外する

- tool call、tool result、順序、対応関係を保持する

- terminal finalは変更しない

- commentaryなどのphaseではなく、tool call構造を主な判定根拠とする

- cutoff以前の履歴には適用しない

- terminal finalなしで終了した過去runは安全側で保持する

ただし、これは最終検証の完了を意味しない。次に実際のA1/A2 fixtureを用い、projected messages、Decision Log、tool pairing、terminal finalの同一性をOracleと正式に照合する。

<br>