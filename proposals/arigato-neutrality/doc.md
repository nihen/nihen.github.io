# ARIGATO CREATION のコミュニティ間中立化

- **対象読者**: ガバナンス投票者（非エンジニア）
- **目的**: PCE プロトコルの「価値中立性」についての現状整理と、唯一の歪みを直す提案

---

## 0. この文書の要点

- PCE と Community Token の価値関係は、**ほぼすべての面で Community 選択に対して中立**（どの Community を選んでも、同じ PCE を預ければ同じ PCE が戻る）
- **唯一の例外が ARIGATO CREATION**（送金時に送信者にトークンがミントされる報酬機能）で、ここだけ Community 選択で期待値が変わる
- 提案: ARIGATO CREATION の「日次ミント上限」を決める **1 つのパラメータだけをプロトコル全体の共通値（ガバナンス管理）に移し**、Community 間の非中立を解消する
- 他のパラメータ（「どう配分するか」）は Community ごとの自由のまま残す

---

## 1. 前提: PCE と Community Token の関係

PCE プロトコルには 3 種類のトークン概念がある。

- **PCE Token**: プロトコル全体の基軸トークン。取引・送金可能で、保有残高は時間経過では減衰しない
- **Community Token**: Community ごとに発行される地域通貨的トークン
- **PCE BASE Token**: Community Token の**裏付け価値**（= その Community Token が最終的に何 PCE と交換できるか）。時間経過で減衰し、PCE 全体の減衰ルールが適用される対象はこちら。**オンチェーンに実在する ERC20 トークンではなく、swap 計算の中に暗黙的に反映される概念上の値**

Community Token は PCE を**担保にして発行**される。PCE を預けると Community Token が返り（swap）、Community Token を戻すと PCE が返ってくる。ロックされるのは **PCE Token** だが、発行された **Community Token が持つ「裏付け価値」が PCE BASE Token に相当する**。

プロトコル全体には **4 つのダイナミクス** がある。

1. **PCE BASE Token の減衰**（PCE 側の時間減衰）
2. **Community Token の減衰**（表示残高の時間減衰）
3. **ARIGATO CREATION**（送金時の追加ミント＝増加）
4. **swap**（PCE と Community Token の双方向交換）

Community ごとに独自のパラメータを設定でき、それが地域通貨としての特色を作る。

---

## 2. PCE BASE Token の減衰

**PCE Token を手元に保有している人の残高は減衰しない**。100 PCE を持っていれば、何年経っても `balanceOf` は 100 PCE のまま変わらない。

減衰は **Community Token の裏付け価値 = PCE BASE Token** に適用される。PCE を swap して Community Token を受け取った時点から、その Community Token が「最終的に何 PCE と交換できるか」は時間とともに目減りしていく。

- **減衰タイミング**: 毎週水曜日（UTC）に 1 回
- **減衰率**: 1 回あたり 0.2%（係数 998/1000）
- **どこで効くか**: Community Token を PCE に swap back するときのレート

### 数値イメージ

PCE は 0.2%/週で減衰するので、1 年あたり約 10% の目減りになる。

| | 1 年後の PCE 換算価値 |
|---|---|
| Bob: 100 PCE をそのまま保有 | **100 PCE**（変化なし） |
| Alice: 100 PCE を Community Token に swap して 1 年後に戻す | **約 90 PCE** |

つまり **PCE 減衰のコストは、PCE BASE Token を保有している人（= Community Token を保有している人）だけが負担**する設計になっている。PCE を直接保有している人は一切影響を受けない。

---

## 3. Community Token の減衰

Community ごとに設定可能なパラメータ:

- `decreaseIntervalDays`: 何日おきに減衰するか
- `afterDecreaseBp`: 1 回あたり何% 残るか（bp = 1/10000）

たとえば「7 日おきに 99.8% 残る（0.2% 減衰）」のような設定ができる。

### 減衰するのは「表示残高」だけ

内部的には 2 種類の残高がある。

- **raw balance（内部残高）**: 減衰しない、不変の数値
- **display balance（表示残高）**: raw balance に Community の現在 factor を掛けた、画面に出る数値

減衰で縮むのは display balance だけで、raw balance は一切動かない。

PCE に swap back するときは **raw balance が基準**となる。このため、Community Token の表示残高が減衰で縮んでも **PCE BASE Token（裏付け価値）は目減りしない**。戻ってくる PCE 量に効くのは、2 章 で説明した PCE BASE Token 自体の減衰のみに限られる。

---

## 4. ARIGATO CREATION

ARIGATO CREATION は、送金のたびに**送信者**へトークンを追加ミントする仕組みを指す。

### 4.1 ミント量の決まり方

ざっくりと:

- 送金額が大きいほど、多くミントされる
- 「使用率」（送金額が残高に占める割合）が Community の理想値（`maxUsageBp`）に近いほど、倍率が高い
- 理想値から離れるほどペナルティで倍率が下がる（ペナルティ強度は `changeBp`）
- 送金に添えるメッセージが長いほど、倍率が高い（最大 10 文字で上限）
- 最大倍率の上限は `maxIncreaseBp`

式を簡略化すると:

```
ミント量 = 送金額 × increaseBp
increaseBp = maxIncreaseBp − 使用率乖離ペナルティ × メッセージ補正
```

**注**: メッセージ長ボーナスのロジックは実装に存在するが、現行の `transfer` 系関数からは `messageCharacters = 1` で固定呼び出しされているため、現在は機能していない（別 PIP でパラメータ化予定）。

### 4.2 段階的な上限

ミントには以下の上限がある。

1. **コミュニティ全体の日次上限**: Community 当日の総発行量の `maxIncreaseOfTotalSupplyBp`（例: 1%）
2. **ゲスト全体の上限**: 上記の 1/10
3. **通常ユーザーの個人上限**: 「全体上限 × その人の保有比率」
4. **ゲスト個人上限**: 全体上限の 1%（フラット、コード内ハードコード値で Community ごとの設定ではない）

### 4.3 パラメータ一覧

以下のパラメータは Community ごとに設定できる。

| パラメータ | 意味 |
|---|---|
| `maxIncreaseBp` | 最大増加率（送金額に対する上限倍率） |
| `maxUsageBp` | 理想使用率（これに近い送金で報酬 max） |
| `changeBp` | 使用率乖離のペナルティ強度 |
| `maxIncreaseOfTotalSupplyBp` | **Community 全体の日次ミント上限（本提案の対象）** |

---

## 5. 価値中立性の分析

### 5.1 2 つの観点

Community 選択による公平性を、2 つの独立した観点で見る。

| 観点 | 意味 |
|---|---|
| **個人の中立性** | 同じ人が同じ行動をしたとき、どの Community を選んでも PCE 換算の期待値が同じ |
| **コミュニティ間の中立性** | Community を集団として見たとき、PCE 換算の日次ミント**上限**が Community 間で公平 |

以下、各メカニズムをこの 2 軸で評価する。

### 5.2 減衰・希釈 — 両方とも中立

Community 固有に設定できる 2 つの係数があるが、**どちらも PCE 換算価値には影響しない**。

**1) Community Token の減衰**

Community ごとに `afterDecreaseBp` / `decreaseIntervalDays` で設定される。3 章 で見たとおり、減衰は表示残高だけに効き、**PCE BASE Token（裏付け価値）は目減りしない**。Community ごとの減衰設定がどんな値でも、最終的に戻ってくる PCE 価値は変わらない。

**2) 希釈率**

Community 作成時に 0.1〜1000 の範囲で設定できる、「1 PCE で Community Token を何単位発行するか」を決める係数。swap 時に可逆で打ち消されるため、単なる額面の選び方にすぎない。

- 希釈率 = 1000 → 1 PCE で 1000 Community Token
- 希釈率 = 0.1 → 1 PCE で 0.1 Community Token

---

```
PCE → Community A → 1 年後に戻す = ~90 PCE
PCE → Community B → 1 年後に戻す = ~90 PCE
（Community 選択で差が出ない）
```

上記の「~90 PCE」は 2 章 で説明した **PCE BASE Token の減衰**（時間経過で約 10%/年）を反映している。この時間経過による目減りは**全 Community に共通して同じように効く**ため、Community 選択の意思決定には影響しない。

**判定**: 個人の中立性 ○ / コミュニティ間の中立性 ○

### 5.3 ARIGATO CREATION — 両方とも非中立

同じ個人が、同じ PCE 相当の送金を Community A と B で行ったとき、**ミントされる量（PCE 換算）が Community ごとに違う**。これは Community のパラメータ設定で変わり、swap で打ち消されない。

- **個人レベル**: 同じ送金でも Community 選択で報酬が違う
- **コミュニティレベル**: Community 全体の日次発行ペース（PCE 換算）も Community 選択で違う

**判定**: 個人の中立性 ✕ / コミュニティ間の中立性 ✕ ← 本提案の対象

---

## 6. 方針: コミュニティ間だけ中立化する

本提案は **コミュニティ間の中立性だけを確保**し、個人の中立性は Community の自治に委ねる。

### 理由

- 「同じ行動に対して Community ごとに報酬率が違う」ことは、**Community 内の分配施策**（どういう送金を推奨するか、メッセージ文化をどう奨励するか）として Community 運営者に任せるのが自然
- Community の経済設計の自由度を残せる
- 一方、**プロトコル全体の発行ペース**（= Community 間で PCE 換算の上限が揃っているか）は、プロトコル公平性の基礎なのでガバナンスで管理すべき

### 具体的に何を変えるか

**`maxIncreaseOfTotalSupplyBp` を Community ごとの設定から、PCE プロトコル全体の共通値（ガバナンス管理）に移す**。

- 現状: 各 Community が `TokenSetting.maxIncreaseOfTotalSupplyBp` を独自に設定
- 変更後: PCE Token に単一のグローバル値を持ち、全 Community がそれを参照

他のパラメータ (`maxIncreaseBp`, `maxUsageBp`, `changeBp`) は **Community ごとに設定可能なまま**。

### 既知のトレードオフ: Community 選択のインセンティブ歪み

個人中立性を Community の自治に委ねることで、ユーザーが「報酬率が有利な Community（高 `maxIncreaseBp` / 使用率条件が緩い設定）を選ぶ」動機は残る。ただしコミュニティ間の総量（日次ミント上限）はプロトコルで揃うため、**プロトコル全体の価値希薄化は防げる**。Community 間のユーザー獲得競争は引き続き Community 運営者の工夫次第で、これは地域通貨の「特色」として許容する方針を取る。

---

## 7. なぜこれで中立化されるのか

各 Community の日次ミント上限は、実装上は次の式で決まる。

```
日次ミント上限(raw) = midnightTotalSupply(raw) × maxIncreaseOfTotalSupplyBp
```

raw 量を PCE 換算に直すと、次の形になる。

```
日次ミント上限(PCE 換算) = Community の PCE BASE Token 総量 × maxIncreaseOfTotalSupplyBp
```

PCE BASE Token 総量は Community Token 全体の裏付け価値であり、**ARIGATO ミントによる増加・Community 間 swap による移動・PCE の時間減衰をすべて織り込んだ値**になっている。したがって `maxIncreaseOfTotalSupplyBp` を Community 間で共通にすれば、各 Community の裏付け価値（PCE 換算）に対して常に同じ比率が自動的に適用される。

例:

| | Community A | Community B |
|---|---|---|
| PCE BASE Token 総量 | 10,000 PCE 相当 | 1,000 PCE 相当 |
| 共通の上限比率 | 1%（= 100 bp） | 1%（= 100 bp） |
| 日次ミント上限（PCE 換算） | 100 PCE 相当 | 10 PCE 相当 |

規模が違う Community 同士でも、**裏付け価値に対する同じ比率が揃う**ため、PCE プロトコル全体から見て公平になる。

---

## 8. 影響範囲

### 変わること

- Community 運営者は **`maxIncreaseOfTotalSupplyBp` を自分で設定できなくなる**
- プロトコル全体の ARIGATO 発行ペースが **ガバナンス投票で一元管理** される
- 既存の Community に保存されている当該設定値は**ミント計算で参照されなくなる**（契約内には残る）

### 変わらないこと

- raw 残高、表示残高、減衰、swap、送金など**基本操作は維持される**（ARIGATO CREATION のミント挙動だけが新しい上限ルールに従う）
- Community ごとの `maxIncreaseBp`, `maxUsageBp`, `changeBp` は**従来どおり Community 運営者が設定可能**
- 個人の送金に対する ARIGATO 報酬率は、従来どおり Community ごとに異なる（= Community 内の分配施策）

### 実装面の留意点

- **ストレージレイアウト**: `TokenSetting.maxIncreaseOfTotalSupplyBp` は Beacon Proxy の互換性維持のため**削除せず残置**
- **ABI 互換**: `createToken` / `setTokenSettings` の引数は維持（値は無視される）
- **ガス代増加**: 毎 transfer で Community → PCEToken のクロスコントラクト呼び出しが追加される。ただし `updateFactorIfNeeded` で既に PCEToken を呼んでいるため warm CALL となり、追加コストは主に新規ストレージスロットの SLOAD（約 2,200 gas）程度。ベンチマークでの実測を推奨

---

## 9. ガバナンスで決めるべきこと

本提案が可決された場合、ガバナンスとして続く判断:

1. **初期値をいくつにするか**
   - 現状の各 Community の設定値（例: 1% / 0.5% 等）を調査し、移行時の妥当値を決める
   - デプロイ直後に初期値が 0 のままだと全 Community の ARIGATO CREATION が停止するため、**同一トランザクション内で初期値設定まで含める**
2. **変更頻度のルール**
   - どのくらいの頻度で見直すか、どの指標を見て判断するか
3. **移行期の扱い**
   - 既存 Community への周知と、設定ズレへの説明

---

## 10. まとめ

- PCE プロトコルの価値中立性は、ARIGATO CREATION の 1 点を除いてすでに成立している
- その 1 点のうち、**コミュニティ間の公平性に関わる部分（日次ミント上限）のみをプロトコル全体で統一**する
- 個人差や分配方法は Community の自治として残す
- 結果: **プロトコル公平性とコミュニティ自由度を両立**する変更
