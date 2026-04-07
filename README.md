# 🎣 室戸沖 出船判断システム
### MUROTO OFFSHORE FORECAST SYSTEM

室戸沖（高知県）への遊漁船の出船可否を、リアルタイムの気象・海況データから自動判定するWebシステムです。
Open-Meteo の無料APIを使用し、5日間・最大15スロット（6時・9時・12時）の予報を提供します。

---

## 🌐 デモ

> GitHub Pages で公開後、以下のようなURLでアクセスできます：
> `https://<your-username>.github.io/<repository-name>/`

---

## 📁 ファイル構成

```
muroto-fishing-forecast/
├── index.html                   # ランディングページ（システム概要・判定基準説明）
├── muroto_fishing_forecast.html # メインアプリ（海況取得・判定・グラフ・CSV出力）
├── .gitignore
└── README.md
```

---

## 🔐 パスワード

両ページにアクセス制限が設定されています。
正しいパスワードを入力するとコンテンツが表示されます。

---

## 🚀 GitHub Pages へのデプロイ手順

### 1. リポジトリ作成

```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

### 2. GitHub Pages を有効化

1. GitHubのリポジトリページを開く
2. **Settings** → **Pages**
3. **Source** を `Deploy from a branch` に設定
4. **Branch** を `main` / `(root)` に設定して **Save**

しばらくすると `https://<your-username>.github.io/<repo-name>/` で公開されます。

> **重要**: GitHub Pages は HTTPS で配信されるため、Open-Meteo APIへの直接fetchが正常に動作します。ローカルの `file://` では CORS制限によりデータ取得できません。ローカルで確認する場合は `python -m http.server` 等でローカルサーバーを起動してください。

---

## 📊 機能

| 機能 | 説明 |
|---|---|
| リアルタイム海況取得 | Open-Meteo Marine API + Weather API を直接fetch |
| 5日間予報 | 6時・9時・12時 × 5日 = 最大15スロット |
| 出船判定スコアリング | 波高＋風速トレンドによる4段階評価 |
| 波高・風速グラフ | Canvas 2D による折れ線グラフ表示 |
| CSV出力 | BOM付きUTF-8で予報データをダウンロード |
| 自動取得 | HTTPS環境では起動時に自動でデータ取得 |

---

## ⚖️ 判定ロジック

### ステップ1: 波高による基本スコア

| 波高 | スコア | 判定 |
|---|---|---|
| ≤ 1.0m | 3 | ◎ 釣り日和 |
| ≤ 1.3m | 2 | ○ 出船可能 |
| ≤ 1.7m | 1 | △ 条件付き |
| > 1.7m | 0 | × 出船不可 |

### ステップ2: 風速トレンド補正

前時刻（3時間前）との風速差が **0.5m/s 以上** の場合に補正を適用します。

- 風速が増加 → スコアを 1段階 **悪化**
- 風速が減少 → スコアを 1段階 **改善**

---

## 🔌 データソース

| API | 取得データ | 認証 |
|---|---|---|
| [Open-Meteo Marine API](https://open-meteo.com/en/docs/marine-weather-api) | 波高 (wave_height) | 不要・無料 |
| [Open-Meteo Weather API](https://open-meteo.com/en/docs) | 風速・風向・降水量 | 不要・無料 |

**対象座標**: 室戸沖 `lat: 33.2 / lon: 134.2`

---

## 🛠 技術スタック

- Vanilla HTML / CSS / JavaScript（フレームワーク不使用）
- Open-Meteo API（無料・認証不要）
- Canvas 2D API（グラフ描画）
- Google Fonts（Noto Sans JP / Share Tech Mono / Rajdhani）

---

## ⚠️ 注意事項

- `wave_height` は風波＋うねりの**合成波高**です。うねりが卓越する場合は実際より過小評価になる可能性があります。
- 本システムの判定はあくまで参考値です。出船可否の最終判断は必ず船長・現地の状況確認のうえ行ってください。
- Open-Meteo API の利用規約に従ってご使用ください。

---

## 📄 ライセンス

MIT License

---

*データ提供: [Open-Meteo](https://open-meteo.com/) — Open-Source Weather API*
