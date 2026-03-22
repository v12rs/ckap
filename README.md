# Ckap

**人間には優しく、ボットには容赦なく。**

APIキー不要。バックエンド不要。依存ライブラリ不要。`<script>` タグ1行で導入完了。

---

## インストール

```html
<script src="ckap.js"></script>
```

以上。

---

## クイックスタート

```html
<div id="my-captcha"></div>
<script src="ckap.js"></script>
<script>
  const id = Ckap.create('my-captcha');

  // フォーム送信時:
  const result = Ckap.verify(id);
  if (result.success) {
    // ✅ 人間と判定 — 送信処理へ
  }
</script>
```

---

## API

### `Ckap.create(containerId, options?)`

指定した `id` の要素にウィジェットをマウントします。

`verify()` に渡す**インスタンスID**を返します。

| オプション | 型 | デフォルト | 説明 |
|-----------|-----|----------|------|
| `label` | `string` | `'I am not a robot'` | チェックボックスのラベル |
| `checkColor` | `string` | `'#fafafa'` | チェックマークの色 |
| `borderColor` | `string` | `'#e4e4e7'` | ウィジェットの枠色 |
| `bgColor` | `string` | `'#ffffff'` | 背景色 |
| `labelColor` | `string` | `'#09090b'` | ラベルの文字色 |
| `expireMs` | `number` | `1800000` | 有効期限（ミリ秒）デフォルト30分 |
| `threshold` | `number` | `100` | 合格に必要な最低スコア（0〜300） |
| `onVerify` | `function` | `null` | 検証成功時に呼ばれるコールバック |

```js
const id = Ckap.create('my-captcha', {
  label: '私はロボットではありません',
  threshold: 120,
  expireMs: 10 * 60 * 1000, // 10分
  onVerify: (result) => {
    console.log('スコア:', result.score);
  }
});
```

---

### `Ckap.verify(instanceId)`

検証結果オブジェクトを返します。

```js
const result = Ckap.verify(id);

// result.success  — boolean
// result.score    — number (0〜300)、success: true のときのみ
// result.token    — string、改ざん防止トークン
// result.reason   — string、success: false のときのみ
```

| `reason` | 意味 |
|----------|------|
| `not_checked` | チェックボックスがまだオンになっていない |
| `expired` | 有効期限切れ |
| `token_mismatch` | トークンが改ざんされている |
| `bot_detected` | ハニーポットまたはDOM改ざんを検出 |
| `invalid_id` | インスタンスIDが見つからない |

---

### `Ckap.reset(instanceId)`

ウィジェットを未チェック状態にリセットします。フォーム送信後に使います。

```js
Ckap.reset(id);
```

---

### `Ckap.isVerified(instanceId)`

`boolean` だけ返す簡易チェック。

```js
if (Ckap.isVerified(id)) {
  // 送信処理へ
}
```

---

## スコアの仕組み

Ckapは各インタラクションを**300点満点**でスコアリングします。デフォルトの合格ラインは**100点**です。

`threshold` オプションで調整できます。

### 加点項目

| シグナル | 点数 |
|--------|------|
| Headless / Botフラグが一切なし | +40 |
| 自然なマウス動作（速度分散＋加速度変化あり） | +30 |
| マウス軌跡の曲率（非直線） | 最大 +30 |
| クリック座標がウィジェット内 | +20 |
| ハニーポットフィールドが空 | +15 |
| クリック圧力 > 0（実ハードウェア） | +15 |
| ページ表示からの経過時間（1.5秒以上） | 最大 +30 |
| スクロールイベントあり | +10 |
| キーボードイベントあり | +10 |
| rAFタイミングの揺らぎ（人間のブラウザ） | 最大 +20 |
| タブが画面に表示されていた | +5 |
| AudioContextフィンガープリントあり | +10 |
| Canvasフィンガープリントあり | +5 |
| WebGLフィンガープリントあり | +5 |
| ハードウェア情報あり（CPU・GPU） | +10 |
| 自然なポインター種別（mouse / touch） | +5 |

### 減点項目

| ペナルティ | 点数 |
|----------|------|
| Headless / Botフラグ1個につき | -20 |
| DOM属性の直接改ざん | -100 |
| ハニーポットに値が入っている | -200 |
| クリック座標がウィジェット外 | -30 |
| ページ表示から150ms以内のクリック | -40 |
| DevToolsが開いている | -10 |
| 短時間の連打（3回超） | -20 |
| クリック圧力がゼロ | -5 |

---

## Bot検出の仕組み

Ckapは**15種類以上の独立した検出チェック**を同時に実行します。

### 行動分析
- **マウス速度分散** — ボットは一定速度で動く。人間は加速・減速する
- **マウス軌跡の曲率** — ボットは直線移動。人間は曲線を描く
- **クリック座標検証** — プログラム的なクリックは要素外に着地しやすい
- **クリックタイミング** — ページ表示から150ms以内のクリックはフラグ
- **ポインター圧力** — 実ハードウェアは圧力 > 0 を返す
- **連打検出** — 3回超の連打はペナルティ
- **スクロール・キーボード** — 実ユーザーは通常ページ全体を操作する

### 環境フィンガープリント
- **Canvasフィンガープリント** — ブラウザの描画特性をハッシュ化してトークンに埋め込む
- **WebGLフィンガープリント** — GPUベンダー・レンダラー・各種パラメータ
- **AudioContextフィンガープリント** — ハードウェア音声処理の特性値（非同期取得）
- **フォント検出** — 17種のフォントをCanvas計測で検出
- **スクリーンフィンガープリント** — 解像度・colorDepth・pixelRatio・タイムゾーン・プラットフォームを複合判定

### Headless / 自動化検出

| チェック項目 | 検出対象 |
|------------|---------|
| `navigator.webdriver` | Selenium、WebDriver |
| `window.callPhantom` | PhantomJS |
| DOM属性 `webdriver` | Seleniumの注入属性 |
| `window.__nightmare` | NightmareJS |
| UAに `HeadlessChrome` | Puppeteer、Playwright headless |
| UAに `PhantomJS` | PhantomJS |
| `navigator.languages` が空 | Headless環境 |
| `navigator.plugins` が空 | Headless Chrome |
| `permissions.query` の改ざん | Puppeteer stealthプラグイン |
| `window.chrome` が存在しない | Chrome UAなのにChromeオブジェクトなし |
| モバイルUAなのにタッチイベントなし | UAスプーフィング |
| `<iframe>` 内での実行 | 埋め込み自動化 |
| `userAgent` プロパティが書き換え可能 | PuppeteerによるUA偽装 |

### 改ざん対策
- **MutationObserver** — `aria-checked` 属性を監視。JSで直接書き換えようとした瞬間にフラグを立てる
- **トークン整合性** — Canvas・WebGL・Audio・Screenの全フィンガープリントを結合してトークンに埋め込む。別のブラウザ・マシンからのトークン再利用は失敗する
- **ハニーポットフィールド** — 実ユーザーには見えない隠し `<input>`。全フィールドを埋めるボットは必ず引っかかる
- **rAFタイミング分析** — `requestAnimationFrame` の揺らぎを計測。Headless環境はフレームタイミングが異常に均一になる

---

## スコアの目安

| 環境 | 典型スコア |
|------|----------|
| デスクトップの実ユーザー | 150〜240 |
| モバイルの実ユーザー | 100〜180 |
| Puppeteer（デフォルト） | 0〜40 |
| Puppeteer-extra-stealth | 20〜70 |
| Selenium | 0〜30 |

---

## トークン検証

`verify()` が返す `token` は、セッションのフィンガープリントを埋め込んだ改ざん防止文字列です。Ckapはクライアントサイドのみで動作しますが、オプションでサーバーサイド検証を追加することもできます。

```js
// クライアント
const { success, token } = Ckap.verify(id);
if (success) {
  fetch('/submit', {
    method: 'POST',
    body: JSON.stringify({ token, ...formData })
  });
}
```

```js
// サーバー（Node.js — オプション）
function decodeCkapToken(token, instanceId) {
  const key = instanceId.slice(3, 11);
  const xored = xorDecode(atob(token), key);
  const payload = JSON.parse(xored);
  const age = Date.now() - payload.vt;
  return age < 30 * 60 * 1000; // 30分以上前なら拒否
}
```

---

## ブラウザ対応

| ブラウザ | 対応 |
|---------|------|
| Chrome 80+ | ✅ |
| Firefox 75+ | ✅ |
| Safari 13+ | ✅ |
| Edge 80+ | ✅ |
| iOS Safari 13+ | ✅ |
| Android Chrome | ✅ |
| IE 11 | ❌ |

---

## 限界について

Ckapはクライアントサイドのみで動作します。十分な時間と知識があれば（暇人ONLY）、ソースを解析してバイパスを作ることは理論上可能です。高セキュリティが求められる用途（金融取引・大規模アカウント作成など）では、サーバーサイド検証と組み合わせてくださいね。

**Ckapが防げるもの**
- スクリプトキディや汎用ボット
- Headlessブラウザ自動化（Puppeteer・Playwright・Selenium）
- 単純なフォーム送信スクリプト
- PhantomJS・NightmareJSスクレイパー
- ほとんどの設定のPuppeteer-stealth

**Ckapが防げないもの**
- ソースを全部読んで対策を打てる熟練した超暇人なアホの攻撃者
- 完全に計装された実ブラウザのリモート操作
- 人力クリックファーム

---

## ライセンス

MIT