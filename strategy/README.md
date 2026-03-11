# AI討伐トーナメント - 戦略エンジン v2 ドキュメント

---

## 1. ファイル構成

```
Poker-AI/
├── index.html                ← メインアプリ（UIは変更なし）
└── strategy/
    └── strategy.json         ← 戦略データ（ここだけ差し替えればOK）
```

---

## 2. 差し替えた関数の説明

### 削除した旧関数
| 旧関数名 | 役割 | 問題点 |
|---|---|---|
| `loadRanges()` | CSVをfetch | CSV3ファイルに分散・形式が不統一 |
| `parseCSV()` | CSV→オブジェクト変換 | 独自パーサーで脆弱 |
| `loadFallbackRanges()` | 埋め込みフォールバック | 巨大なハードコード |
| `getPreflopGTO()` | プリフロップ判定 | if文の固定ロジック |
| `getPostflopGTO()` | ポストフロップ判定 | if文の固定ロジック |

### 追加した新関数
| 新関数名 | 役割 |
|---|---|
| `loadStrategy()` | `strategy.json` をfetchしてSTRATにセット |
| `normalizeHand(c1,c2)` | カード2枚 → `"AA"` / `"AKs"` / `"KQo"` 形式に正規化 |
| `normPos(pos)` | `EP`→`UTG` など戦略JSONキー用ポジションに変換 |
| `buildPreflopSituationKey(hand,pos)` | `{section, key}` を返す。`rfi` / `vs3bet` / `bb_defense` を自動判定 |
| `classifyBoard(comm)` | コミュニティカード → `"dry"` / `"mid"` / `"wet"` |
| `classifyStrength(c1,c2,comm)` | ハンド強度 → `"nut"` / `"strong"` / `"medium"` / `"weak"` / `"air"` |
| `buildPostflopKey(...)` | `"flop:IP:medium:dry:check"` 形式のキー生成 |
| `lookupStrategy(section,key)` | JSONから該当エントリを取得。なければポジションフォールバック |
| `resolveSize(sizeStr,pot,bb,callAmt)` | `"0.5pot"` / `"2.5BB"` / `"2.5x"` → chips数値に変換 |
| `getPreflopRecommendation(hand,pos)` | プリフロップ推奨を返す |
| `getPostflopRecommendation(...)` | ポストフロップ推奨を返す |
| `updateAnalysis()` | UIに結果を表示（UIは変更なし） |

---

## 3. 局面キーの仕組み

### プリフロップキー

```
{ポジション}:{ハンド}
例: "BTN:AKs"  "BB:QQ"  "UTG:72o"
```

シチュエーションは自動判定：
- **rfi**: ヒーローがまだレイズしていない（オープン候補）
- **vs3bet**: ヒーローがレイズ済み＆相手もレイズ済み
- **bb_defense**: BBポジション＆相手がレイズ済み

### ポストフロップキー

```
{ストリート}:{IP/OOP}:{ハンド強度}:{ボードテクスチャ}:{facing}
例: "flop:IP:medium:dry:check"
    "turn:OOP:nut:wet:bet"
    "river:IP:air:dry:check"
```

| 要素 | 値 |
|---|---|
| street | `flop` / `turn` / `river` |
| IP/OOP | `IP`（BTN,CO,HJ）/ `OOP`（それ以外）|
| hand_strength | `nut` / `strong` / `medium` / `weak` / `air` |
| board | `dry`（wetness 0-1）/ `mid`（2）/ `wet`（3以上）|
| facing | `bet`（相手がベット済み）/ `check` |

---

## 4. JSON更新ルール

### 基本構造

```json
{
  "_meta": { "version": "1.0", "updated": "2026-03-11" },
  "preflop": {
    "rfi":         { "BTN:AKs": { "action":"RAISE","freq":100,"size":"2.5BB","reason":"説明" } },
    "vs3bet":      { "BTN:AKs": { "action":"RAISE","freq":100,"size":"4bet", "reason":"説明" } },
    "bb_defense":  { "BB:AKs":  { "action":"3BET", "freq":100,"size":"3bet", "reason":"説明" } }
  },
  "postflop": {
    "flop:IP:nut:dry:check": { "action":"BET","freq":90,"size":"0.75pot","reason":"説明" }
  }
}
```

### フィールド定義

| フィールド | 型 | 説明 |
|---|---|---|
| `action` | string | `RAISE` / `3BET` / `4BET` / `CALL` / `FOLD` / `CHECK` / `BET` |
| `freq` | number | GTOでのアクション頻度 0〜100（%）。100未満の場合はUI上に表示される |
| `size` | string \| null | ベットサイズ。後述の形式を使用 |
| `reason` | string | UI上の推奨理由テキスト |

### sizeフィールドの書き方

| 書き方 | 意味 | 例 |
|---|---|---|
| `"Xpot"` | ポットのX倍 | `"0.5pot"` → ポットの50% |
| `"XBB"` | BBのX倍 | `"2.5BB"` → BB×2.5 |
| `"Xx"` | コール額のX倍 | `"2.5x"` → 相手のベット×2.5 |
| `"4bet"` | BB×9 固定 | 4betサイズ |
| `"3bet"` | BB×8 固定 | 3betサイズ |
| `null` | サイズなし | CHECK/FOLD/CALL |

### ハンド強度の基準（変更する場合はJSも修正）

| 分類 | 条件 |
|---|---|
| `nut` | セット以上・SF・クワッズ・フルハウス |
| `strong` | フラッシュ・ストレート・2ペア・オーバーペア・トリップス |
| `medium` | トップペア（グッドキッカー含む） |
| `weak` | ミドルペア以下・フラッシュドロー |
| `air` | ノーペア・ノードロー |

### ポジションフォールバック

JSONにキーが存在しない場合、自動的に以下の順で広いポジションに落ちます：
```
HJ → CO → BTN
MP → CO → BTN
```
つまり `"BTN"` のエントリだけ書けば大半はカバーできます。

### ボードフォールバック

`wet` / `mid` のキーが存在しない場合、自動的に `dry` で再検索します。
まず `"flop:IP:medium:wet:check"` を探し、なければ `"flop:IP:medium:dry:check"` を使います。

---

## 5. データ更新手順（GTOWizard参照時）

1. GTOWizardで対象シチュエーションを開く
2. 各ハンドの推奨アクションと頻度(%)を確認
3. `strategy.json` の該当セクションを更新
4. `"freq"` が混合（例:55%コール/45%フォールド）の場合は**多い方**を `action` に書く
5. `"reason"` に確認日や出典を残しておくと管理しやすい
6. GitHubにプッシュ → デプロイ完了

```json
// 更新例: GTOWizardでBTN JJ vs 3betを確認したとき
"BTN:JJ": {
  "action": "CALL",
  "freq": 55,
  "size": null,
  "reason": "JJ vs 3bet → 55%コール/45%4bet [GTOWizard確認 2026-03]"
}
```
