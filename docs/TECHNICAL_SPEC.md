# さぁジャッジ！ 技術仕様書

> カードゲームのルールをクイズで学べるWebアプリケーション

---

## 1. プロジェクト概要

| 項目 | 内容 |
|------|------|
| サービス名 | さぁジャッジ！ |
| ドメイン | `sirjudge.com` |
| 対象ゲーム | ポケモンカードゲーム（他ゲーム追加予定） |
| クイズ形式 | ◯×（True / False）形式 |

---

## 2. システム構成

```
┌─────────────────────────────────────────┐
│              ユーザーブラウザ            │
│                                         │
│  index.html（親ページ）                 │
│    └── iframe（quiz.html）              │
│          ↕ postMessage                  │
└────────────────┬────────────────────────┘
                 │ Fetch API
                 ▼
┌────────────────────────────────────────┐
│  Google Apps Script（Web App URL）     │
│  ・スプレッドシートからJSON返却        │
│  ・pokemon-card.comリンクプレビュー取得│
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Google スプレッドシート               │
│  シート名：「問題集」                  │
│  （問題データベース）                  │
└────────────────────────────────────────┘
```

---

## 3. ホスティング

| 項目 | 内容 |
|------|------|
| プラットフォーム | Cloudflare Workers（Static Assets） |
| 設定ファイル | `wrangler.jsonc` |
| Workers名 | `morning-field-11aa` |
| 互換日付 | `2026-01-08` |
| デプロイ対象 | リポジトリルート全体（`"./"` をassets directoryに指定） |

---

## 4. フロントエンド技術スタック

| 項目 | 内容 |
|------|------|
| **言語** | HTML5, CSS3, JavaScript (ES6+) |
| **フレームワーク** | なし（バニラJS） |
| **CSSリセット** | Josh's Custom CSS Reset |
| **フォント** | Noto Sans JP（Google Fonts, weight 100〜900） |
| **レスポンシブ** | 対応（breakpoints: 768px, 1024px） |
| **データ保存** | localStorage |

### 外部連携

| 連携先 | 用途 | 通信方法 |
|--------|------|----------|
| Google Apps Script | クイズ問題データ取得・リンクプレビュー取得 | Fetch API (JSON) |
| Google Fonts | Webフォント配信 | `<link>` |
| X (Twitter) | 結果共有 | Intent URL (`twitter.com/intent/tweet`) |

---

## 5. ページ詳細

### 5-1. トップページ（`index.html`）
- ゲームタイトル一覧を表示
- ポケモンカードゲームへのリンクと「coming soon」プレースホルダーを配置

### 5-2. ポケモンモード選択（`games/pokemon/index.html`）
- 「ふつうにジャッジ」「れんぞくジャッジ」の2モードへ遷移
- 設定ページへのリンクを表示

### 5-3. クイズ埋め込みページ（`games/pokemon/ox/index.html`）
- `quiz.html` を `<iframe>` に埋め込むラッパーページ
- URLパラメータ `?mode=normal` または `?mode=renzoku` でモード切替
- `postMessage` を利用してiframeの高さを動的に同期
- OGP / Twitter Card メタタグを設定

### 5-4. クイズ本体（`quiz.html`）
- クイズUIのコアロジックをすべて内包
- iframe埋め込み・スタンドアロン両対応

### 5-5. 設定ページ（`games/pokemon/settings/index.html`）
- クイズのフィルタ条件・表示設定をUIで管理
- `localStorage` に設定を保存・読み込み

---

## 6. クイズ機能仕様

### 6-1. クイズモード

| モード | 説明 |
|--------|------|
| ふつうにジャッジ | 設定した出題数（デフォルト10問）でランダム出題 |
| れんぞくジャッジ | 全問題をシャッフルして出題、1問でも不正解でゲームオーバー |

### 6-2. URLパラメータ（`quiz.html`）

| パラメータ | 説明 |
|-----------|------|
| `embed` | `1` のとき埋め込みモードで動作 |
| `embedId` | iframeとの `postMessage` 識別子 |
| `mode` | `normal` or `renzoku` |
| `title` | ヘッダー表示タイトル |
| `data` | Google Apps Script Web App URL |
| `shareTitle` | SNS共有テキスト用タイトル |
| `shareHashtag` | SNS共有ハッシュタグ |
| `shareUrl` | SNS共有URL |
| `debugId` | 特定問題IDを直接デバッグ表示 |

### 6-3. 設定項目（`localStorage: "quizSettings"`）

| 設定 | 型 | デフォルト |
|------|----|---------|
| `regulation` | string | `"all"` |
| `difficulty` | string[] | `["かんたん","ふつう","むずかしい"]` |
| `questionCount` | number | `10` |
| `showRuby` | boolean | `true` |
| `autoScroll` | boolean | `true` |

---

## 7. バックエンド（Google Apps Script）

### 7-1. エンドポイント

`GET ?limit=200` — 問題一覧JSON取得

#### クエリパラメータ

| パラメータ | 説明 |
|-----------|------|
| `id` | 特定IDの問題のみ取得 |
| `difficulty` | 難易度でフィルタ |
| `limit` | 取得件数上限 |
| `public=1` | `answer`・`note` を除外して返す |
| `preview=<URL>` | リンクプレビュー情報を返す |

### 7-2. スプレッドシートのカラム定義

| スプレッドシート列名 | JSONキー |
|--------------------|---------|
| 問題ID | `id` |
| 難易度 | `difficulty` |
| 問題文 | `text` |
| レギュ | `rule` |
| 形式 | `format` |
| 正答 | `answer` |
| 解説 | `note` |
| 参考リンク1〜5 | `link1`〜`link5` |
| カード名1〜5 | `link1_label`〜`link5_label` |

#### レスポンス例

```json
{
  "count": 42,
  "items": [
    {
      "id": "001",
      "difficulty": "ふつう",
      "text": "場のポケモンに「ダメカン」を乗せることができる？",
      "rule": "スタンダード",
      "format": "OX",
      "answer": "o",
      "note": "ダメカンはバトル場・ベンチ問わず乗せられます。",
      "links": [
        { "url": "https://www.pokemon-card.com/...", "label": "ピカチュウ" }
      ]
    }
  ]
}
```

### 7-3. リンクプレビュー機能

- `?preview=<URL>` で `pokemon-card.com` のカード画像URLを取得
- `<img class="fit">` かつ `/assets/images/card_images/` パスを持つ画像を優先抽出
- `large` → `medium` → `small` の優先順でスコアリング
- PCはホバー時にツールチップ表示、スマホはボトムシート（`<dialog>`）で表示

---

## 8. iframe通信プロトコル（postMessage）

```
親ページ (ox/index.html)  ←→  子ページ (quiz.html)
│                                    │
│ ── quiz-embed-ping ──────────────→ │  高さ再通知要求
│ ←── quiz-embed-size ─────────────  │  高さ同期 (height)
│ ←── quiz-scroll-bottom ──────────  │  下スクロール要求
│ ←── quiz-scroll-top ─────────────  │  上スクロール要求
```

| メッセージタイプ | 送信元 | 内容 |
|----------------|--------|------|
| `quiz-embed-size` | `quiz.html` | iframeの現在の高さを親に通知 |
| `quiz-scroll-bottom` | `quiz.html` | 回答後、解説へのスクロールを要求 |
| `quiz-scroll-top` | `quiz.html` | 次の問題表示後、ページ上部へのスクロールを要求 |
| `quiz-embed-ping` | 親ページ | ロード・回転時に子へ高さ再通知をリクエスト |

- メッセージは `window.postMessage` で送受信
- `id` フィールドで埋め込みインスタンスを識別
- 高さ同期は `ResizeObserver` + デバウンス付き

---

## 9. ルビ（ふりがな）記法

| 記法 | 説明 | 例 |
|------|------|----|
| パイプ付き | スペース・記号含む基底対応 | `｜基底《よみ》` |
| パイプ無し | 直前の連続文字列が基底 | `漢字《よみ》` |

- 設定で非表示時はルビ記法を除去してテキストのみ表示

---

## 10. レスポンシブ対応

| ブレークポイント | 説明 |
|----------------|------|
| `< 768px` | モバイルレイアウト（カード1列） |
| `768px〜` | タブレット以上（カード2列） |
| `1024px〜` | PC（最大幅1200px、余白拡大） |

### デバイス固有の対応
- **タッチデバイス** (`hover: none`): リンクプレビューはボトムシートモーダルで表示、スワイプで閉じ可能
- **PC** (`hover: hover`): リンクプレビューはマウスホバーでツールチップ表示
- **iOS版Chrome** (`CriOS`): 回答後の自動スクロールを無効化（互換性対策）

---

## 11. SNS共有機能

- **X（Twitter）**：`twitter.com/intent/tweet` にテキスト・ハッシュタグ・URLを渡して投稿
- **クリップボードコピー**：`navigator.clipboard.writeText()`（フォールバックあり）

---

## 12. セキュリティ・その他

- 全ての動的テキスト出力は `esc()` でHTMLエスケープ処理
- Apps ScriptのURLフェッチ時に `User-Agent` ヘッダーを付与
- iframeの `scrolling="no"` とCSS `overflow:hidden` でスクロールを親ページに委譲
- iOS Chrome（CriOS）検出によるスクロール挙動の条件分岐

---

## 13. キーボードショートカット（quiz.html）

| キー | 動作 |
|------|------|
| `O` | ◯（正しい）を選択 |
| `X` | ✕（間違い）を選択 |
| `→` (ArrowRight) | 次の問題 / 結果を見る |
| `Escape` | 確認ダイアログを閉じる |
| `Enter` | 確認ダイアログでOK |

---

*最終更新：2026年2月*
