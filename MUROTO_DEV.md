# 室戸沖 出船判断システム — 開発引き継ぎドキュメント

**作成日**: 2026-04-07  
**最終更新**: 2026-04-08（v3 判定ロジック追加）  
**対象エリア**: 室戸沖（lat: 33.2 / lon: 134.2）  
**担当**: Cowork 引き継ぎ用

---

## 1. プロジェクト概要

室戸沖での釣り出船可否を、海況・気象データから自動判定するシステム。  
波高と風速トレンドを組み合わせた独自スコアリングで、5日先までの出船判定を提供する。

---

## 2. 現在の実装状況

### バージョン履歴

| バージョン | ファイル名 | 説明 | 状態 |
|---|---|---|---|
| v1 | `muroto_fishing_forecast.html` | スタンドアロンHTML。ネイビー×シアンのナビゲーション風デザイン。Canvas グラフ付き | ローカル実行不可（CORS制限） |
| v2 | `muroto_v2.html` | Claude.ai Artifact対応版。claude.ai上のウィジェットとして動作 | claude.ai専用 |
| **v3** | **`muroto_v3.html`** | **5ファクター判定ロジック搭載。南風・逆波・グレーゾーン複合補正。予約システム送信ステータス列追加** | **現行推奨版** |

### v1 の構成（muroto_fishing_forecast.html）

```
HTML単体ファイル（約1000行）
├── スタイル（CSS変数ベース、ダークテーマ）
├── ヘッダー（ロゴ・座標バッジ）
├── コントロールバー（実行ボタン・CSVボタン・ステータス）
├── サマリーカード × 4（最良条件・平均波高・出船可能枠・最大風速）
├── 海況テーブル（日付区切り・波高バー・風向矢印・判定バッジ）
├── Canvas折れ線グラフ × 2（波高・風速）
└── CSVプレビュー・ダウンロード
```

### v2 の構成（muroto_v2.html / Artifact）

```
Claude Artifact（Visualizer Widget）
├── Anthropic API (/v1/messages) 経由でデータ取得
│   └── web_search_20250305 ツールを使って Open-Meteo APIへアクセス
├── サマリーカード × 4
├── 海況テーブル（日付区切り・波高バー・風向矢印・判定バッジ）
└── CSV保存ボタン
```

---

## 3. データソース

### Open-Meteo Marine API（無料・認証不要）

```
https://marine-api.open-meteo.com/v1/marine
  ?latitude=33.2
  &longitude=134.2
  &hourly=wave_height
  &timezone=Asia%2FTokyo
  &forecast_days=5
```

取得フィールド: `wave_height`（合成波高、単位m）, `wave_direction`（波向き、度）

### Open-Meteo Weather API（無料・認証不要）

```
https://api.open-meteo.com/v1/forecast
  ?latitude=33.2
  &longitude=134.2
  &hourly=wind_speed_10m,wind_direction_10m,precipitation
  &wind_speed_unit=ms
  &timezone=Asia%2FTokyo
  &forecast_days=5
```

取得フィールド: `wind_speed_10m`（m/s）, `wind_direction_10m`（度）, `precipitation`（mm）

### 抽出対象時刻

- 毎日 **6時・9時・12時** の3スロット × 5日 = **最大15行**

---

## 4. データ構造（行データ）

```javascript
{
  t:    "2026-04-07T06:00",  // ISO8601 JST
  wave: 0.92,                // 波高 (m)
  wind: 4.2,                 // 風速 (m/s)
  dir:  180,                 // 風向 (度, 0=北)
  rain: 0.1,                 // 降水量 (mm)
  sc:   3                    // 判定スコア 0-3
}
```

---

## 5. 判定ロジック

> **v3より5ファクター判定を採用。v2以前の2ファクター判定は廃止。**

---

### v3 判定ロジック（5ファクター）

#### ファクター①: 波高ベーススコア（グレーゾーン拡張）

| 波高 | スコア | 判定 |
|---|---|---|
| ≤ 1.0m | 3 | ◎ 釣り日和 |
| ≤ 1.5m | 2 | ○ 出船可能（グレーゾーン下限） |
| ≤ 2.0m | 1 | △ 未定・要確認 |
| > 2.0m | 0 | × 出船不可 |

> v2までは `≤1.3m` が○基準だったが、40フィート船のグレーゾーン（1.0〜1.5m）を正確に評価するため `≤1.5m` に拡張。

---

#### ファクター②: 南風ペナルティ（室戸岬特性）

室戸岬沖は南〜南東方向からの風で沖合の波が成長しやすい地形特性を持つ。

| 条件 | 補正 |
|---|---|
| 風向 135°〜225°（南東〜南西） かつ 風速 ≥ 8m/s | **−1** |
| 風向 315°〜45°（北西〜北東） | ±0（北風有利タグ表示） |

---

#### ファクター③: 逆波ペナルティ（風向 vs 波向きの干渉）

風の向きと波の進行方向が逆・交差すると三角波・急峻波が発生し、同じ波高でも体感が大きく悪化する。

| 風向と波向きの角度差 | 補正 |
|---|---|
| 120°以上（逆方向） | **−1**（逆波タグ） |
| 90°〜120°（交差波） | **−0.5**（交差波タグ） |
| 90°未満（同方向系） | ±0 |

波向きデータ: `wave_direction`（Open-Meteo Marine API）を使用。取得失敗時はこの補正を省略。

---

#### ファクター④: グレーゾーン複合補正（40フィート船特性）

1.0〜1.5mの「悩みどころ」波高帯で、②南風ペナルティと③逆波ペナルティが**同時に発生**した場合、追加で −1。

```
条件: wave > 1.0m AND wave ≤ 1.5m
      AND 南風ペナルティ発動
      AND 逆波ペナルティ発動
→ score -= 1（複合悪化タグ）
```

これにより「波高は1.2mだが南風8m/sで波向きが逆」という状況を正しく「× 出船不可」と評価できる。

---

#### ファクター⑤: 風速トレンド補正（v2から継続）

前スロット（3時間前）との風速差が 0.5m/s 以上の場合に補正を適用。

| 条件 | 補正 |
|---|---|
| 風速増加（+0.5m/s以上） | **−1**（風↑タグ） |
| 風速減少（−0.5m/s以上） | **+1**（風↓タグ） |

---

#### 最終スコアと予約システム送信ステータス

スコアは 0〜3 に丸め（Math.round + clamp）。

| スコア | 表示 | 予約システム送信ステータス |
|---|---|---|
| 3 | ◎ 釣り日和 | ✅ 出船可 |
| 2 | ○ 出船可能 | ✅ 出船可 |
| 1 | △ 未定・要確認 | ⚠ 要確認 |
| 0 | × 出船不可 | ❌ 中止 |

---

#### v3 calcScore 実装（JavaScript）

```javascript
function angleDiff(a, b) {
  let diff = Math.abs((a - b + 360) % 360);
  return diff > 180 ? 360 - diff : diff;
}

function calcScore(wave, windNow, windPrev, windDir, waveDir) {
  const tags = [];

  // ① 波高ベーススコア
  let s;
  if      (wave <= 1.0) s = 3;
  else if (wave <= 1.5) s = 2;
  else if (wave <= 2.0) s = 1;
  else                  s = 0;

  // ② 南風ペナルティ
  const inSouth = windDir >= 135 && windDir <= 225;
  const inNorth = windDir >= 315 || windDir <= 45;
  let southPenalty = false;
  if (inSouth && windNow >= 8) {
    s -= 1; southPenalty = true; tags.push({ cls: 'pt-south', lbl: '南風↑' });
  } else if (inNorth) {
    tags.push({ cls: 'pt-north', lbl: '北風◎' });
  }

  // ③ 逆波ペナルティ
  let opposePenalty = false;
  if (waveDir != null) {
    const diff = angleDiff(windDir, waveDir);
    if (diff >= 120) {
      s -= 1; opposePenalty = true; tags.push({ cls: 'pt-oppose', lbl: '逆波' });
    } else if (diff >= 90) {
      s -= 0.5; tags.push({ cls: 'pt-cross', lbl: '交差波' });
    }
  }

  // ④ グレーゾーン複合補正
  if (wave > 1.0 && wave <= 1.5 && southPenalty && opposePenalty) {
    s -= 1; tags.push({ cls: 'pt-compound', lbl: '複合悪化' });
  }

  // ⑤ 風速トレンド補正
  if (windPrev != null) {
    if      (windNow > windPrev + 0.5) { s -= 1; tags.push({ cls: 'pt-trend-up', lbl: '風↑' }); }
    else if (windNow < windPrev - 0.5) { s += 1; tags.push({ cls: 'pt-trend-dn', lbl: '風↓' }); }
  }

  return { score: Math.max(0, Math.min(3, Math.round(s))), tags };
}
```

---

## 6. CORSの問題と対処方針

### 問題

ブラウザのセキュリティ制限（CORS）により、`file://` プロトコルで開いたHTMLから外部APIへのfetchが失敗する。  
claude.ai の Artifact サンドボックス内でも同様の制限が存在する。

### 現在の対処（v2）

Anthropic API（`/v1/messages`）+ `web_search_20250305` ツールを経由してOpen-MeteoのデータをClaudeに取得させ、JSONとして返させる。

### 今後の推奨対処

| 方法 | 難易度 | 概要 |
|---|---|---|
| ローカルサーバー経由 | ★☆☆ | `python -m http.server` や VS Code Live Server で起動 |
| Vercel / Netlify デプロイ | ★★☆ | HTTPSホスティングすればCORS問題なし |
| バックエンドプロキシ | ★★★ | Node.js/Python サーバーがAPIを代理取得してフロントに渡す |
| Cowork デスクトップアプリ | ★★☆ | Electron等ではCORS制限なし |

---

## 7. 今後の開発ロードマップ

### Phase 1: 安定動作（優先度: 高）

- [ ] **ローカル動作の解決**: Python/Node簡易サーバー付きのREADME作成
- [ ] **エラーハンドリング強化**: API応答なし・データ欠損時のフォールバック表示
- [ ] **時刻の拡張**: 対象時刻を 3時・6時・9時・12時・15時・18時 に拡張するオプション
- [ ] **自動更新**: 起動時に自動でデータ取得、ページ表示と同時に結果表示

### Phase 2: 判定精度向上（優先度: 中）

- [ ] **うねり・風波の分離表示**: `swell_wave_height` と `wind_wave_height` を個別表示
- [ ] **危険風向の判定**: 室戸沖特有の危険方位（例: 南東〜南）を設定して警告
- [ ] **瞬間風速の考慮**: `wind_gusts_10m` を取得して突風リスクを表示
- [ ] **波周期の表示**: `wave_period` を取得して波の性質（急峻か緩やかか）を表示

### Phase 3: UX・可視化強化（優先度: 中）

- [ ] **週間カレンダービュー**: 日ごとのベスト判定をカレンダー形式で一覧表示
- [ ] **グラフの改善**: v1のCanvasグラフをv2にも移植（Chart.js推奨）
- [ ] **プッシュ通知**: 釣り日和の前日にブラウザ通知
- [ ] **ダークモード/ライトモード切り替え**

### Phase 4: データ拡張（優先度: 低）

- [ ] **潮汐データ連携**: 国土地理院または気象庁の潮位データ取得
- [ ] **月齢データ**: 満月・新月付近の大潮情報を表示
- [ ] **水温データ**: Open-Meteo Ocean API の `sea_surface_temperature` 追加
- [ ] **複数地点対応**: 室戸岬・甲浦・高知港 などのプリセット地点選択
- [ ] **釣果予測AI**: 過去の海況データと釣果を学習させたスコア予測モデル

### Phase 5: アプリ化（優先度: 将来）

- [ ] **Webアプリ化**: React + Viteでの再実装、Vercelデプロイ
- [ ] **PWA対応**: ホーム画面に追加・オフラインキャッシュ
- [ ] **スマホ最適化**: モバイルファーストのレイアウト調整
- [ ] **Coworkデスクトップアプリ化**: 毎朝自動取得・通知

---

## 8. ファイル構成

```
muroto-fishing-system/
├── MUROTO_DEV.md                  ← このファイル（引き継ぎドキュメント）
├── muroto_fishing_forecast.html   ← v1: スタンドアロン版（デザイン完成形）
├── muroto_v2.html                 ← v2: Claude Artifact対応版（2ファクター判定）
└── muroto_v3.html                 ← v3: 5ファクター判定・現行推奨版 ★
```

---

## 9. 技術スタック

| 項目 | 内容 |
|---|---|
| フロントエンド | Vanilla HTML/CSS/JS（フレームワーク不使用） |
| データ取得 | Open-Meteo Marine API + Weather API（無料・無認証） |
| API経由取得（v2） | Anthropic `/v1/messages` + `web_search_20250305` ツール |
| グラフ | Canvas 2D API（v1独自実装） |
| フォント | Noto Sans JP, Share Tech Mono, Rajdhani（Google Fonts） |
| CSVエクスポート | Blob + URL.createObjectURL（BOM付きUTF-8） |

---

## 10. 判定カラーコード

| 判定 | 表示 | v1カラー | v2カラー |
|---|---|---|---|
| スコア3 | ◎ 釣り日和 | `#00ffb3` | green ramp |
| スコア2 | ○ 出船可能 | `#00c8ff` | blue ramp |
| スコア1 | △ 条件付き | `#ffb300` | amber ramp |
| スコア0 | × 出船不可 | `#ff4444` | red ramp |

---

## 11. 既知の問題・注意事項

1. **Open-Meteo Marine API の wave_height** は風波＋うねりの合成値。うねりが卓越する場合は実際より過小評価になる可能性がある。
2. **風向補正の「前時刻」** は同日内の前スロット（3時間前）を参照。日付をまたぐ場合も連続して補正が適用される。
3. **v2のAnthropic API経由取得** はClaudeがウェブ検索でAPIレスポンスを取得する方式のため、取得に10〜30秒かかる場合がある。
4. **スマホ表示** はv1のグラフ列（2カラム）が縦積みになるが、テーブルが横スクロールなしで収まるか要確認。

---

*最終更新: 2026-04-08 by Claude Sonnet 4.6（v3 5ファクター判定追加）*
