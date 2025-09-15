Note Pipeline (Mastra互換) – GitHub Actions

このリポジトリの `.github/workflows/note-pipeline.yaml` は、@mastra/ の挙動に準拠して、以下のパイプラインをGitHub Actionsで実行します。

1) リサーチAgent: Claude Code SDK の WebSearch / WebFetch によるリサーチレポート作成
2) 執筆Agent: Anthropic Claude 4.0 Sonnet でタイトル/本文/タグ(JSON)を生成
3) ファクトチェックAgent: Tavily を使った検証結果を反映し本文を修正
4) ドラフトAgent: Playwrightで note.com に下書き/公開（storageState を利用）

---

事前準備（リポジトリSecrets）

- ANTHROPIC_API_KEY（必須）
- TAVILY_API_KEY（必須）
- NOTE_STORAGE_STATE_JSON（必須: 後述の note-state.json をJSON文字列として格納）

---

実行方法

GitHub Actions > Note Pipeline (Mastra-like) を手動実行し、以下の入力を与えます。

- theme: 記事テーマ（必須）
- target: 想定読者（必須）
- message: 伝えたい核メッセージ（必須）
- cta: 読後のアクション（必須）
- tags: カンマ区切りタグ（任意）
- is_public: true/false（公開 or 下書き保存）
- dry_run: true/false（投稿スキップ）

---

note-state.json の取得手順（Playwright storageState）

note.com へのログイン情報は Playwright の storageState(JSON) を用います。以下の手順で取得してください。

1. ローカルで Playwright を準備

```bash
npm init -y
npm i playwright
npx playwright install chromium
```

2. スクリプト `login-note.mjs` を作成

```javascript
import { chromium } from 'playwright';
import fs from 'fs';

const STATE_PATH = './note-state.json';

const EMAIL = process.env.NOTE_EMAIL;
const PASSWORD = process.env.NOTE_PASSWORD;

if (!EMAIL || !PASSWORD) {
  console.error('Set NOTE_EMAIL and NOTE_PASSWORD env vars.');
  process.exit(1);
}

const wait = (ms) => new Promise(r => setTimeout(r, ms));

(async () => {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();

  await page.goto('https://note.com/login');

  // ログイン方法に応じて適宜セレクタを調整してください（メール/Google/Twitter等）。
  await page.fill('input[type="email"]', EMAIL);
  await page.fill('input[type="password"]', PASSWORD);
  await page.click('button:has-text("ログイン")');

  // ログイン完了待ち（ユーザーの環境により調整）
  await page.waitForURL(/note\.com\/?$/, { timeout: 120000 }).catch(()=>{});
  await wait(2000);

  // 保存
  await context.storageState({ path: STATE_PATH });
  console.log('Saved:', STATE_PATH);

  await browser.close();
})();
```

3. 実行してファイルを得る

```bash
NOTE_EMAIL="あなたのメール" NOTE_PASSWORD="あなたのパスワード" node login-note.mjs
# カレントディレクトリに note-state.json が生成される
```

4. Secret に保存

- NOTE_STORAGE_STATE_JSON に note-state.json の中身全体を貼り付けます（JSON文字列）。
  - 例: VSCodeで開いて全選択コピー → GitHub リポジトリ Settings > Secrets and variables > Actions > New repository secret

5. 動作イメージ

- ワークフロー実行時に NOTE_STORAGE_STATE_JSON を runnerTemp/note-state.json に展開し、Playwright の storageState として使用します。
- 公開フラグが false の場合は「下書き保存」、true の場合は「公開」を行います。

---

注意事項

- storageState は期限切れ・無効化されることがあります。ログイン情報が切れた場合は、同手順で再取得してください。
- note.com 側のUI変更でセレクタが変わる場合があります。その際は post.mjs 内のセレクタ調整が必要です。
- 生成AIの出力は誤情報を含む場合があります。公開前に最終レビューをおすすめします。

---

参考
- Claude Code SDK: `https://github.com/anthropics/anthropic-claude-code`
- AI SDK (Anthropic): `https://sdk.vercel.ai/docs`
- Tavily: `https://docs.tavily.com/`
- Playwright: `https://playwright.dev/`


