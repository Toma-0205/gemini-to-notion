# GEMINI.ME - Gemini to Notion Knowledge Archiver 開発ログ

## 2026-01-13 開発完遂 (旧: Gemini to X)

### 指示内容
Chrome拡張機能「Gemini to X Knowledge Archiver」の開発。Geminiの回答をX API v2 (Free Plan)を使用してスレッド形式で自動投稿する機能を実装。

### 実装完了ファイル

| ファイル | 説明 |
|---------|------|
| `manifest.json` | Manifest V3形式。gemini.google.comへのcontent script注入、storage/alarms権限設定 |
| `lib/crypto-js.min.js` | OAuth 1.0a署名生成用HMAC-SHA1ライブラリ（CDNからダウンロード） |
| `background.js` | OAuth 1.0a署名生成（Signature Base String構築、HMAC-SHA1）、X API v2投稿、24時間/月間制限管理 |
| `options.html/js` | APIキー4種の設定画面、制限状況表示、glassmorphism風UI |
| `content.js` | Gemini回答検出（MutationObserver）、「Xへ投稿」ボタン挿入、280ポイント分割アルゴリズム |
| `styles.css` | ボタン・トースト・確認ダイアログのスタイル |
| `popup.html/js` | ポップアップUI、残りリクエスト数表示 |

### 技術的ポイント

#### OAuth 1.0a署名生成（background.js）
```
1. OAuthパラメータ生成（consumer_key, token, nonce, timestamp等）
2. パラメータのアルファベット順ソート
3. Signature Base String = METHOD&URL&ParameterString
4. Signing Key = ConsumerSecret&TokenSecret
5. HMAC-SHA1(BaseString, SigningKey) → Base64エンコード
```

#### 制限管理ロジック
- 24時間制限：スライディングウィンドウ方式（タイムスタンプ配列で管理、24時間以上経過したログを自動削除）
- 月間制限：毎月1日にリセット（年月でリセット判定）
- chrome.storage.localに`postLogs`（タイムスタンプ配列）、`monthlyCount`、`monthlyResetDate`を保存

#### テキスト分割（content.js）
- Xの仕様に基づきポイント計算（全角2pt、半角1pt）
- 行単位で分割を試み、長い行は文字単位で分割
- リクエスト数最小化のため1ツイートあたりの文字数を最大化

### 使用方法
1. Chrome拡張機能として読み込み（chrome://extensions → 開発者モード → パッケージ化されていない拡張機能を読み込む）
2. オプション画面でX Developer Portalから取得した4つのAPIキーを設定
3. https://gemini.google.com でGeminiに質問
4. 回答エリアに表示される「Xへ投稿」ボタンをクリック
5. 確認ダイアログで内容を確認後、投稿実行

---

## 2026-01-13 バグ修正：ボタンが「処理中」で止まる問題

### 依頼内容
Geminiで「Xへ投稿」ボタンを押すと「処理中...」と表示されるが、その後何も起きない。

### 原因
`manifest.json` で `"type": "module"` を指定していたため、`background.js` がESモジュールとして扱われ、`importScripts('lib/crypto-js.min.js')` が動作しなかった。ESモジュールモードでは `importScripts` は使用できない。

### 修正内容
`manifest.json` から `"type": "module"` を削除。

```diff
  "background": {
-   "service_worker": "background.js",
-   "type": "module"
+   "service_worker": "background.js"
  },
```

### 修正後の動作確認方法
1. chrome://extensions を開く
2. 拡張機能の「更新」ボタンをクリック（または一度削除して再読み込み）
3. Geminiで再度「Xへ投稿」ボタンをクリック

---

## 2026-01-13 v2.0 Notion連携への完全移行 (プロジェクト始動)

### 概要
「Gemini to X」拡張機能をベースに、保存先を「Notion」へ変更し、Knowledge Archiverとして新生。

### 実装機能
- Notion API (Bearer Token) 認証
- 全文保存機能
- API Key + Database ID 設定

---
| 項目 | 変更前（X版） | 変更後（Notion版） |
|------|--------------|-------------------|
| 認証方式 | OAuth 1.0a (HMAC-SHA1署名) | Bearer Token |
| API | X API v2 (POST /2/tweets) | Notion API (POST /v1/pages) |
| テキスト処理 | 280pt分割（スレッド投稿） | 分割なし（全文保存） |
| 設定項目 | APIキー4種 | API Key + Database ID（2種） |
| 制限管理 | 17回/24h、500回/月 | なし（Notion無料枠で十分） |

### 修正ファイル

| ファイル | 変更内容 |
|---------|---------|
| `manifest.json` | v2.0.0、host_permissions を `api.notion.com` に変更 |
| `background.js` | OAuth削除、Notion API呼び出し、ページ作成ペイロード生成、要約/TODO抽出 |
| `options.html/js` | 入力を2項目に簡素化、設定手順ガイド追加 |
| `content.js` | 分割ロジック削除、プロンプト抽出機能追加、ボタンを「Notionへ保存」に変更 |
| `styles.css` | クラス名とカラーをNotionテーマに変更 |
| `popup.html/js` | ステータス表示のシンプル化 |

### Notionデータベース構造

保存先データベースには以下のプロパティ（列）が必要：

| 列名 | タイプ | 内容 |
|-----|-------|------|
| 名前 | タイトル | プロンプト冒頭50文字 |
| 回答 | テキスト | Gemini回答全文（2000文字まで） |
| 概要 | テキスト | 最初の2-3文を自動要約 |
| やること | テキスト | TODO抽出（あれば） |
| 時期 | 日付 | 保存日 |

### 使用方法
1. Notion Integrationsでインテグレーション作成、トークン取得
2. 保存先データベースにインテグレーションを招待
3. オプション画面でAPI KeyとDatabase IDを設定
4. Geminiで「Notionへ保存」ボタンをクリック

---

## 2026-01-13 v3.0 Gemini API統合・スレッド全体保存

### 依頼内容
1. 単一回答ではなく、チャットスレッド全体（全プロンプト＋全回答）を保存
2. Gemini API (gemini-1.5-flash) で会話を構造化要約
3. 保存前に編集可能なプレビューを表示

### 新機能フロー
```
1. 「Notionへ保存」ボタン押下
2. content.js: チャット全体をDOM抽出（extractEntireThread）
3. background.js: Gemini APIに送信、JSON形式で構造化データ生成
4. content.js: 編集可能プレビューダイアログ表示
5. 確定 → Notion APIでページ作成
```

### 変更ファイル

| ファイル | 変更内容 |
|---------|---------|
| `manifest.json` | v3.0.0、Gemini API権限追加 |
| `background.js` | Gemini API呼び出し（gemini-1.5-flash）、JSON構造化プロンプト |
| `content.js` | 全スレッド抽出、編集可能プレビューダイアログ、グローバル固定ボタン |
| `options.html/js` | Gemini API Key入力欄追加（計3項目） |
| `styles.css` | 編集フォームスタイル、固定位置ボタン |
| `popup.html` | v3.0機能説明 |

### Gemini APIプロンプト設計
会話データを送信し、以下のJSON形式で出力を指定：
```json
{
  "title": "会話全体の核心を突いたタイトル",
  "summary": "3行程度の要約",
  "content": "詳細な議事録",
  "todos": "ネクストアクション",
  "date": "YYYY-MM-DD"
}
```

### 設定項目（options.html）
1. Gemini API Key（Google AI Studioで取得）
2. Notion API Key
3. Notion Database ID

---

## 2026-01-13 Hotfix: Gemini API Endpoint 修正

### 依頼内容
`models/gemini-1.5-flash is not found for API version v1` エラーへの対応。
Gemini APIのエンドポイントを `v1` から `v1beta` に変更。

### 修正内容
`background.js` の `GEMINI_API_ENDPOINT` 定数を修正。

```javascript
// 変更前
const GEMINI_API_ENDPOINT = 'https://generativelanguage.googleapis.com/v1/models/gemini-1.5-flash:generateContent';

// 変更後
const GEMINI_API_ENDPOINT = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent';
const GEMINI_API_ENDPOINT = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent';
```
これに伴い、モデル `gemini-1.5-flash` が正常に呼び出せるようになる。

---

## 2026-01-13 v4.0 プロンプト注入戦略への変更

### 依頼内容
Gemini API無料枠の制限が厳しいため、API呼び出しを廃止。
代わりに、「要約用のプロンプト」を生成し、Geminiのチャット入力欄に自動で貼り付ける方式に変更。

### 新フロー
1. 右下の「✨ まとめてNotion用にする」ボタンをクリック
2. 拡張機能がスレッド全体を抽出し、JSON出力用プロンプトを作成
3. Geminiの入力欄にプロンプトが自動で貼り付けられる
4. ユーザーがEnterを押して送信
5. GeminiがJSON形式で回答を生成
6. 生成された回答の「Notionへ保存」ボタンを押す
7. JSONが自動パースされ、編集ダイアログが表示される → 保存

### 利点
- Gemini API Keyが不要（設定項目から削除）
- Google AI Studioのクォータ制限を受けない
- Gemini Advanced等の高性能モデルをUI経由で利用可能

### 修正ファイル
- `content.js`: プロンプト生成・注入ロジック、JSONパースロジック追加
- `background.js`: Gemini API関連コード全削除
- `options.html/js`: Gemini API Key設定削除
- `manifest.json`: 権限削除

---

## 2026-01-13 v4.1 - v4.3 UI改善・プロパティエラー修正・プロンプト強化

立て続けにユーザビリティと互換性の向上を行いました。

### v4.1 (UI & 権限)
- **UI変更**: 固定ボタンを廃止し、各回答の下に「Notionへ保存」「✨ まとめ作成」ボタンを配置。
- **Notion権限**: `Could not find database` エラー対策として、オプション画面にインテグレーション招待手順を明記。

### v4.2 (保存ロジック修正)
- **プロパティエラー解消**: `回答 is not a property` 等のエラーに対応。
- **実装**: カスタムプロパティ（概要、日付etc）への依存を排除し、すべての情報を**ページ本文**に集約して保存するロジックに変更。
- **タイトル自動検知**: データベースのタイトル列名（`Name`等）を自動検知して保存。

### v4.3 (プロンプト強化)
- **要約プロンプト刷新**: 「優秀な秘書、データアナリスト」として振る舞うプロンプトに変更。
- **目的**: 議事録やTODOの抽出精度を向上。

```text
あなたは優秀な秘書であり、データアナリストです...
JSON形式のみを出力し、解説や前置きは一切不要です...
```

---

## 2026-01-14 v4.4 UI調整

### 依頼内容
ボタンの文言変更、アイコン削除、および配置の入れ替え。

### 変更点
- **文言変更**: 「✨ まとめて作成」→「まとめを作成」、「📓 Notionへ保存」→「Notionへ保存」
- **アイコン削除**: ボタン内のアイコンを削除し、テキストのみのシンプルな表示に変更。

---

## 2026-01-14 v4.5 プロンプト強化・UX改善

### 依頼内容
1. プロンプトに「MECE（漏れなくダブりなく）」の指示を追加し、網羅性を向上させる。
2. 保存完了時のトースト通知から、保存先のNotionページに直接遷移できるようにする。

### 変更点
- **`content.js`**:
    - **プロンプト**: `generateSummaryPrompt` に「点はMECEを徹底し、取りこぼしがないようにしてください」の一文を追加。
    - **トースト通知**: `showToast` 関数を拡張し、第3引数でURLを受け取れるように変更。URLがある場合は「↗ 開く」リンクを表示。
    - **保存処理**: Notion保存成功時に、返却された `pageUrl` をトーストに渡すよう変更。
