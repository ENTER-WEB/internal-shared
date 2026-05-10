# Claude Code 作業指示書（このリポジトリ専用）

このリポジトリは ENTER WEB 事業部の社内共通マニュアル置き場。
Claude Code がこのフォルダを開いた状態でユーザーの指示を受けたとき、本ファイルに沿って動作すること。

---

## トリガー：ユーザーがこう言ったら

### 「{案件略号} のデモサイトにパスワード保護を設定して」

または同等の依頼（「デモのpw保護」「staticryptで守って」等）。

→ §A の手順を実行する。

### 「internal-shared に「○○のマニュアル」を追加して」

→ §B の手順を実行する。

---

## §A. デモサイト パスワード保護のセットアップ

詳細仕様は `docs/demo-password-protect.html` 全文を必ず読むこと。
以下は実行サマリ。

### 前提チェック（自動）

```bash
# gh CLI ログイン状態とactiveアカウントを確認
gh auth status

# ENTER-WEB org への push 権限があるか確認
gh api orgs/ENTER-WEB/memberships/$(gh api user --jq .login) 2>/dev/null \
  || echo "WARN: org membership unconfirmed. Try: gh auth refresh -s admin:org"
```

権限が無い場合はユーザーに伝えて停止。

### 実行手順

1. **案件略号を確認**：ユーザーから受け取った略号を `SLUG` 変数に保存（小文字3〜4文字）
2. **2リポジトリ作成**：
   ```bash
   gh repo create ENTER-WEB/${SLUG}-site --private --description "Source for ${SLUG} demo (do not edit Pages branch)"
   gh repo create ENTER-WEB/${SLUG}-demo --public  --description "Encrypted demo for ${SLUG}. Source: ENTER-WEB/${SLUG}-site"
   ```
3. **PW生成**：
   ```bash
   DEMO_PW=$(openssl rand -base64 16 | tr -dc 'A-Za-z0-9' | head -c 16)
   ```
4. **GitHub Secrets 登録**：
   ```bash
   echo "$DEMO_PW" | gh secret set DEMO_PW --repo ENTER-WEB/${SLUG}-site
   # DEPLOY_TOKEN: ユーザーに Fine-grained PAT 発行を依頼（Pages repo: Contents=R/W、90日）して登録
   ```
5. **Workflow設置**：`docs/demo-password-protect.html` の §04 にある `.github/workflows/deploy-demo.yml` のテンプレートを取得し、`SLUG` を実値に置換して新リポジトリに commit→push
6. **Pages有効化**：
   ```bash
   gh api -X POST repos/ENTER-WEB/${SLUG}-demo/pages \
     -f "source[branch]=main" -f "source[path]=/"
   ```
7. **完了報告**：
   - 公開URL: `https://enter-web.github.io/${SLUG}-demo/`
   - パスワード（テキスト）: `${DEMO_PW}` ← **ユーザー画面のみに表示。ファイルやSlackに残さない**
   - 「次にやること」（半自動）：
     - 1Password の Vault `Enter-Web-Standard` に Login アイテム作成（Title: `[{SLUG}] デモサイト`、PW=上記、URL=公開URL、Tags `client:{SLUG} type:demo`）
     - Item Sharing リンクを発行（有効期限7日、メール認証必須）
     - クライアントへのメール下書きを生成して提示

### 半自動：1Password とメール

- **Claude は手で実行しない**：1Password アプリ操作はユーザーが手動で行う
- Claude は手順テキストとメール下書きを返答するのみ
- 1Password CLI (`op`) が利用可能であれば、ユーザーに「自動化しますか？」と聞いて選択肢を提示

---

## §B. 新マニュアル追加

### ヒアリング

- 何のマニュアルか（テーマ）
- 元になる情報源（既存ドキュメント・URL・口頭）
- コピペ対象（コマンド・コード・メール文面など）
- 隠しトグル化したい補足セクション

### HTML作成

`docs/{slug}.html` に以下のフォーマットで生成：

- **デザイン**: `docs/css/vaizo-editorial.css` 参照（白黒＋ゴールドアクセント、Inter+Noto Sans JP+JetBrains Mono）
- **構造の型**: `docs/demo-password-protect.html` をテンプレートとして流用
  - 結論先プロンプトセクション（`<section class="conclusion">`）— 必ずページ最上部
  - 鉄則 / 構成図 / 手順 / トラブル対応 / 関連ドキュメント / フッター
- **必須要素**:
  - コピペ対象は `<div class="code">` または `<div class="email">` で囲み、`<button class="code__copy">Copy</button>` を付与
  - 補足説明は `<details><summary>...</summary><div class="howto__body">...</div></details>` で隠しトグル化
  - 末尾に Copy ボタン用 `<script>` を含める

### PDF生成

Playwright で隠しトグル全展開してPDF化。テンプレート：

```js
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto("file:///" + path.resolve("docs/{slug}.html").replace(/\\/g, "/"), { waitUntil: "networkidle" });
await page.evaluate(() => {
  document.querySelectorAll("details").forEach(d => d.open = true);
  const nav = document.querySelector(".top-nav");
  if (nav) nav.style.display = "none";
});
await page.pdf({ path: "pdf/V_ENTER_WEB_{Title}_v1.0_YYYY-MM-DD.pdf", format: "A4", printBackground: true });
```

### 配置・push・index更新

```bash
# README.md と docs/index.html の「収録マニュアル」セクションに新マニュアルを追記
git add . && git commit -m "feat: {マニュアル名} 追加" && git push origin main
```

### Drive アップロード（管理者のみ）

ユーザーが管理者で `~/drive_upload/token.json`（drive.file scope）を持っている場合のみ実行。
無ければスキップしてよい。

```python
# Drive folder ID for ENTER WEB は管理者にOnly
# 詳細は管理者のローカルメモリ参照
```

### ユーザーへ報告

- GitHub Pages 公開URL: `https://enter-web.github.io/internal-shared/{slug}.html`
- (該当時) Drive PDF閲覧URL
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

## 環境前提

- **GitHub**: ユーザーの gh CLI で ENTER-WEB org に push 権限のあるアカウントが active
- **Drive アップロード**: 管理者のローカル token がある場合のみ。無ければスキップ
- **Pages配信**: enter-web.github.io/internal-shared/ で配信中（main /docs）
- **OS依存しない**：Windows / macOS / Linux で同等に動作するよう、絶対パスをハードコードしない
