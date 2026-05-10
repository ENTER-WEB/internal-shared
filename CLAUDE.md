# Claude Code 作業指示書（このリポジトリ専用）

このリポジトリは ENTER WEB 事業部の社内共通マニュアル置き場。
ユーザーが「マニュアルを追加して」と言ったら、以下の手順で完了させること。

---

## マニュアル新規追加・更新の手順

### 1. ヒアリング

- 何のマニュアルか（テーマ）
- 元になる情報源（既存ドキュメント・URL・口頭）
- コピペ対象（コマンド・コード・メール文面など）
- 隠しトグル化したい補足セクション

### 2. HTML作成

`docs/{slug}.html` に以下のフォーマットで生成：

- **デザイン**: `docs/css/vaizo-editorial.css` 参照（白黒＋ゴールドアクセント、Inter+Noto Sans JP+JetBrains Mono）
- **構造の型**: `docs/demo-password-protect.html` をテンプレートとして流用
  - Hero（タイトル・サブタイトル・リード）
  - TOC（目次）
  - §00 鉄則（Iron Rules：絶対NG / 必ず守る）
  - §以降は内容に応じて
  - 関連ドキュメント
  - フッター
- **必須要素**:
  - コピペ対象は `<div class="code">` または `<div class="email">` で囲み、`<button class="code__copy">Copy</button>` を必ず付与
  - 補足説明は `<details><summary>...</summary><div class="howto__body">...</div></details>` の隠しトグル化
  - ページ末尾に Copy ボタン用 `<script>` を必ず含める（既存ファイルからコピー）

### 3. PDF生成

```bash
cd C:/Users/nnkre/enter-web   # 親ハンドブックrepo
PDF_OUT="C:/Users/nnkre/AppData/Local/Temp/internal-shared-work/V_ENTER_WEB_{Title}_v{X.Y}_{YYYY-MM-DD}.pdf" \
  node scripts/gen-manual-pdf.mjs
```

スクリプトは `docs/demo-password-protect.html` 固定なので、別マニュアルなら `gen-manual-pdf.mjs` 内の `HTML` 変数を一時的に書き換えるか、引数化する。

### 4. このrepoに配置

```bash
cd C:/Users/nnkre/AppData/Local/Temp/internal-shared-work/internal-shared
# 無ければ: gh repo clone ENTER-WEB/internal-shared
cp {新HTML} docs/{slug}.html
cp {生成PDF} pdf/V_ENTER_WEB_{Title}_v{X.Y}_{YYYY-MM-DD}.pdf
# README.md と docs/index.html の「収録マニュアル」一覧に追記
git add . && git commit -m "feat: {マニュアル名} 追加" && git push origin main
```

### 5. Drive アップロード

ENTER WEB フォルダ（ID: `1I6ERLluC2LR4i6kpYgp8qTJYzQDITzV7`）にPDFをアップロード：

```python
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

creds = Credentials.from_authorized_user_file(
    r"C:\Users\nnkre\drive_upload\token.json",
    scopes=["https://www.googleapis.com/auth/drive.file"],
)
if creds.expired and creds.refresh_token:
    creds.refresh(Request())
build("drive", "v3", credentials=creds).files().create(
    body={"name": "{ファイル名}.pdf", "parents": ["1I6ERLluC2LR4i6kpYgp8qTJYzQDITzV7"], "mimeType": "application/pdf"},
    media_body=MediaFileUpload("{ローカルPDFパス}", mimetype="application/pdf", resumable=True),
    fields="id, name, webViewLink",
    supportsAllDrives=True,
).execute()
```

### 6. ユーザーへ報告

- GitHub Pages 公開URL
- Drive の PDF 閲覧URL
- 一言サマリ

---

## デザイン規約（VAIZO Editorial）

- **配色**: 白黒のみ＋ゴールド（`--accent-gold: #b89b6a`）アクセント。NGカラーは赤（`#c8302e`）
- **フォント**:
  - 英文タイトル: Inter 900
  - 和文本文: Noto Sans JP 700
  - コード: JetBrains Mono
- **罫線**: ヘアライン（`var(--rule-hairline)`）
- **Hero**: 大きな英文タイトル（clamp 40〜80px）、JPサブタイトル、リード文、Update日付
- **セクション番号**: `01 / Section Name` 形式（英大文字＋ゴールド）
- **コードブロック**: ダーク背景（`#0a0a0a` / `#f5f3ee`）
- **メールブロック**: ライト背景（クリーム）
- **隠しトグル `<details>`**: `+/−` マーカーがゴールドで切り替わる

---

## アカウント・権限

- **GitHub**: `r-sakurai-vaizo` アカウントで実行（`gh auth switch --user r-sakurai-vaizo`）
- **Drive**: `~/drive_upload/token.json`（drive.file scope、アップロード可）
- **Pages**: enter-web.github.io/internal-shared/ で配信中（main /docs）

---

## ファイル命名規則

- HTML: `docs/{kebab-case-slug}.html`
- PDF: `pdf/V_ENTER_WEB_{PascalCase}_v{X.Y}_{YYYY-MM-DD}.pdf`
  - 例: `V_ENTER_WEB_DemoPasswordProtect_v1.0_2026-05-10.pdf`

---

## 親リポジトリ（参照用）

- **VAIZO-jp/enter-web**: V/ENTER WEB ハンドブック本体（Public、外部公開用）
  - ローカル: `C:\Users\nnkre\enter-web\`
  - HTMLマニュアルの正本は `assets/static/*.html`、ビルド出力が `docs/`
  - 本リポジトリ（internal-shared）はこのHTMLをミラー＆PDF配布する位置づけ
