# PCE 準備金のグローバルプール化

- **対象読者**: ガバナンス投票者（非エンジニア）
- **目的**: PCE 準備金の設計原則を明示し、Community ごとに分離された準備金をグローバルプールに統合する提案

---

## 0. この文書の要点

- 現状、PCE 準備金は **Community ごとに分離**して管理されている
- ARIGATO ミントや Community 間 swap によって「個別 Community の準備金」と「実際の claim」がズレていく構造がある
- 提案: 全 Community 共通の **グローバル PCE プール**に統合し、swap-back のチェックはプール全体で行う
- 設計原則を明示: 100% 裏付けは保証しない「流動性プール型（fractional reserve）」として運用する

---

## 1. 前提: PCE と Community Token

PCE プロトコルには 3 種類のトークン概念がある。

- **PCE Token**: プロトコル全体の基軸トークン
- **Community Token**: Community ごとに発行される地域通貨的トークン
- **PCE BASE Token**: Community Token の裏付け価値（= 最終的に何 PCE と交換できるか）

Community Token は PCE を**担保にして発行**される。PCE を預けると Community Token が返り（swap）、Community Token を戻すと PCE が返ってくる。このロックされた PCE が本提案で扱う **PCE 準備金**。

---

## 2. 現状: Community ごとに分離された準備金

Community Token が発行される仕組み:

- ユーザーが PCE を預けると、その Community の準備金として **PCE Token 実体がコントラクト内にロック**される
- ロックされた金額は `depositedPCEToken[community]` という変数で Community ごとに追跡される
- swap-back 時は、戻そうとする PCE 量がこの準備金以下でなければ失敗する

つまり各 Community は **自分の準備金に紐づいた Community Token しか戻せない**、という分離構造になっている。

---

## 3. 現状の問題点

Community ごとに分離した準備金モデルには、時間経過で 3 つのズレが発生する。

### 3.1 ARIGATO ミントによる claim の膨張

ARIGATO CREATION は送金のたびに Community Token を追加ミントする。このとき **準備金は増えない**。したがって時間とともに claim 総額が準備金を超えていき、「全員が一斉に戻そうとすると一部の swap-back が失敗する」状態になる。

### 3.2 Community 間 swap による偏り

ユーザーが Community A から Community B に Community Token を swap すると、raw 供給は A から B に移る。しかし **準備金は移動しない**（A にロックされた PCE はそのまま）。結果として B 側では「受け取った raw に対する準備金の裏付けが不足」する状態が生じる。

### 3.3 PCE 減衰による surplus

PCE は毎週 0.2% 減衰する（年間約 10%）ため、1 claim あたりの PCE 換算価値は時間とともに減る。これは逆に「準備金 > claim」の方向に振れる。

---

## 4. 設計原則をどう決めるか

問題を整理すると、「PCE 準備金は何を保証するのか」という設計原則が曖昧なまま実装が動いている状態にある。考えられる立場は 3 つ。

| 立場 | 意味 | 含意 |
|---|---|---|
| **A. 100% 裏付け保証** | 全 Community Token が PCE に 1:1 で戻せる | ARIGATO ミントは PCE の追加調達が必要になる |
| **B. 流動性プール** | 日常的な swap 需要をさばける程度の流動性があれば OK | 一斉引き出しは想定しない、fractional reserve |
| **C. 初期出資** | Community 立ち上げの stake のみ、ARIGATO は seigniorage 扱い | 現状に近い、claim > 準備金を容認 |

### 本提案の立場: B 流動性プール

- 100% 裏付けは技術的にも経済的にも過剰（ARIGATO は使えば使うほどミントされるので 1:1 を維持するコストが高い）
- 一方で「無保証」のまま放置するのは Community Token の信用を損なう
- 日常的な swap 需要を満たす流動性を確保しつつ、**一斉引き出しは日次 swap 上限で抑制、究極的には早い者勝ちで保証しない**、という立場を明示する

---

## 5. 具体的に何を変えるか

### 5.1 グローバル PCE プールへの統合

- Community ごとの `depositedPCEToken[community]` を廃止し、**全 Community 共通の単一 PCE プール**で準備金を管理する
- swap-back のチェックは、戻そうとする PCE 量がこのグローバルプール残高以下であるかどうかのみ

### 5.2 Community 間 swap の簡素化

- `swapTokens`（Community A → B）で準備金の移動を考慮する必要がなくなる
- raw 供給だけが移動し、準備金は全体プールに集約されているので、偏りが発生しない

### 5.3 日次 swap 上限の役割

- プロトコル全体の急激な引き出しを防ぐスロットリングとして、日次 swap 上限は引き続き機能する
- 細かな値の見直しはガバナンスで別途判断

---

## 6. 影響範囲

### 変わること

- swap-back の失敗条件が「個別 Community の準備金不足」から「グローバルプールの不足」に変わる
- Community 間 swap (`swapTokens`) が準備金の観点で単純化される
- `depositedPCEToken[community]` は swap ゲートから外れる（情報的役割として残すか削除するかは 7 章で検討）

### 変わらないこと

- 通常の送金、transfer、ARIGATO ミント、Community Token の減衰など基本操作は維持される
- 各 Community の raw / display 残高の見え方
- PCE Token 直接保有者への影響はない

### 実装面の留意点

- **ストレージレイアウト**: `localTokens[community].depositedPCEToken` は Beacon Proxy 互換性のため削除せず残置
- **ABI 互換**: `getDepositedPCETokens()` などのゲッターは維持
- **移行**: 既存の各 Community の `depositedPCEToken` の合計を初期グローバルプール残高として移行する

---

## 7. ガバナンスで決めるべきこと

1. **既存 `depositedPCEToken[community]` の扱い**
   - 情報的な履歴として残すか、廃止するか
2. **グローバルプール残高が枯渇した場合の挙動**
   - swap-back 失敗でユーザーに返却、でいいか
   - 枯渇しそうになった時点でガバナンス介入を発動するか
3. **日次 swap 上限の再調整**
   - グローバル化後の妥当値

---

## 8. ARIGATO 中立化提案との関係

本提案と並行して検討している「ARIGATO CREATION のコミュニティ間中立化」は、ARIGATO の日次ミント上限（`maxIncreaseOfTotalSupplyBp`）を全 Community 共通のガバナンス管理値にする提案。

- 本提案（グローバルプール化）を進めると、**ARIGATO ミントのペースはプール枯渇に直結する**
- したがって ARIGATO の bp をガバナンスで一元管理する必然性はむしろ**増す**
- 2 つは独立に成立するが、**両方同時に導入すると整合的で、運用の見通しが良くなる**

---

## 9. まとめ

- PCE 準備金の設計原則を「**流動性プール（fractional reserve）**」として明示する
- Community ごとの準備金分離を廃止し、**グローバル PCE プール**に統合する
- swap-back は全体プールに対してチェック、日次上限がスロットリング、究極は早い者勝ちで 100% 保証しない
- Community 間の偏りは消える。ARIGATO 由来の claim 膨張はプロトコル全体で吸収する
- 結果: **設計意図の明確化とシンプル化を両立**する変更

---

## 関連リポジトリ

- [peacecoin-protocol/core](https://github.com/peacecoin-protocol/core) — Solidity 実装
- [peacecoin-protocol/PIPs](https://github.com/peacecoin-protocol/PIPs) — PEACE COIN Improvement Proposals
