# 開発ノート — 心移植 術後管理ツール

## なぜこのアーキテクチャか

**単一 HTML ファイル（CSS・JS インライン、外部依存ゼロ）**

- ビルドツール不要・ホスティング依存ゼロ → GitHub Pages に `index.html` を置くだけで完結
- オフライン動作可（ブラウザキャッシュのみで使える）
- tehaishi-manual と同パターンで運用が統一される

---

## データ構造

### プロトコール期間テーブル（17 区分）

```js
const PERIODS = [
  {
    label: '0〜1週',
    dayMin: 0, dayMax: 7,
    biopsy: '1週', biopsyFull: false,
    predMg: 20, predNote: null,
    tacOld: [12, 14],           // 旧機器目標トラフ [下限, 上限] ng/mL
    cyaOld: [160, 180],
    mmfLt50: [500, 1000],       // BW < 50 kg
    mmfGe50: [500, 1000],       // BW ≥ 50 kg
  },
  // ... 17 区分
];
```

プロトコール原文の目標値はすべて **旧機器値**。表示時に換算式で新値に変換する。

### 換算式

```js
const CONVERT = {
  tac: { a: 0.964, b: -0.029 },
  cya: { a: 1.124, b:  2.23  },
  evl: { a: 1.519, b:  1.498 },
};
// Y(新) = a × X(旧) + b
function oldToNew(drug, val) { return CONVERT[drug].a * val + CONVERT[drug].b; }
```

`toFixed(1)` で丸める。IEEE 754 の銀行丸め（1.124×130+2.23 = 148.35 → "148.3"）は正常動作。

### 生検スケジュール

```js
function biopsyStrToDays(str, pod) {
  if (m = str.match(/^(\d+)週$/)) return parseInt(m[1]) * 7;
  if (m = str.match(/^(\d+)年$/)) return parseInt(m[1]) * 365;
  if (str === '1年毎') return Math.ceil((pod + 1) / 365) * 365;
  if (str === '2年毎') return Math.ceil((pod + 1) / 730) * 2 * 365;
}
```

- "N週" は移植日と **同じ曜日**（N×7 日後）になる → 表示は `target日 - target+7日`
- `pod + 1` により「記念日ちょうど」のとき翌サイクルを指す
- `new Date(txDateVal + 'T00:00:00')` でローカル時刻固定（YYYY-MM-DD 文字列を UTC 解釈しない）

### CMV 投与量テーブル

```js
const VALGAN = [  // バリキサ Ccr 別（5 段階）
  { ccrMin: 60, ccrMax: Infinity, init: '900 mg×2/日', maint: '900 mg×1/日' },
  ...
];
const GANCI = [   // デノシン Ccr 別（mg/kg、5 段階）
  { ccrMin: 70, ccrMax: Infinity, initDose: 5.0, initInt: 12, maintDose: 5.0, maintInt: 24 },
  ...
];
```

デノシン透析患者の維持量は **0.625 mg/kg**（0.635 は誤植）。

---

## 主要計算関数

```js
function computePOD(txDateVal) {
  // 今日と移植日の差（日数）。移植当日 = POD 0
  // setHours(0,0,0,0) でローカル日付ベースに揃える
}

function findPeriod(days) {
  // PERIODS の dayMin <= days < dayMax で線形検索（O(17)）
}

function selectMMF(period, weight) {
  return weight < 50 ? period.mmfLt50 : period.mmfGe50;
}
```

---

## XSS 対策

ユーザー入力（体重・Ccr・移植日）は全て `esc()` を通してから innerHTML に挿入。

```js
function esc(s) {
  return String(s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

---

## GitHub Pages デプロイ

```bash
# 初回（プライベートリポは有料プランのみ Pages 対応 → パブリックで運用）
gh repo create tehaishi-tx-monitor --public
gh repo edit --visibility public --accept-visibility-change-consequences

# Pages 有効化
gh api repos/{owner}/{repo}/pages -X POST \
  -f build.type=legacy -f source.branch=main -f source.path=/

# 検索エンジン除外（robots.txt）
# User-agent: *
# Disallow: /
```

リポジトリはパブリックだが robots.txt で検索エンジンをブロック。センシティブなデータは含まない（入力値は永続化しない）。

---

## プロトコール更新時のメンテ手順

1. `阪大心移植プロトコール.md` を参照して `PERIODS` 配列の該当行を修正
2. 換算式が変わった場合は `CONVERT` オブジェクトを更新
3. 手順・文章系（ステロイド漸減・EVL・拒絶プロトコル）は `§5` アコーディオン内の固定 HTML を直接編集
4. `git commit` → GitHub Pages が自動デプロイ（数分）

---

## 設計で避けたこと

| 避けたもの | 理由 |
|---|---|
| React / Vue 等の SPA | ビルドチェーンが重い・要件に対して過剰 |
| ローカルストレージ永続化 | 「都度入力」が要件・個人情報リスク回避 |
| 複数ファイル分割 | 単一 HTML の方がオフライン・配布・印刷が簡単 |
| 認証機能 | プライベートリポ運用 + robots.txt で対応 |
