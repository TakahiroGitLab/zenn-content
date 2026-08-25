---
title: "CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない"
emoji: "🧊"
type: "tech"
topics: ["css", "html", "frontend", "ui", "design"]
published: true
---

iOS や macOS のすりガラス風の表現を CSS でやろうとして、`backdrop-filter: blur()` を書いてみたものの、**なんとなく濁っているだけ**の板ができて終わった、という経験はないでしょうか。私はそうなりました。

原因は単純で、**ぼかす対象がなかった**からです。

実際に動くものがあります。

- **デモ**: https://takahirogitlab.github.io/gcal-entry-log/
- **ソース**: https://github.com/TakahiroGitLab/gcal-entry-log （MIT）

この記事では、それらしく見せるために必要な 5 つの層を、実際のコードで分解します。

## liquid glass の正体は 5 層

CSS に屈折はないので、Apple のあの表現を厳密に再現することはできません。ただ、次の 5 つを重ねると、かなり近いところまで行きます。

| 層 | 役割 |
| --- | --- |
| **1. 背景** | ぼかす対象。**これがないと成立しない** |
| 2. `backdrop-filter` | 背後をぼかし、彩度を上げる |
| 3. 半透明のフィル | ガラス本体 |
| 4. スペキュラエッジ | 縁の光沢 |
| 5. 影とハイライト | 浮遊感 |

**重要度は上から順**です。特に 1 と 2 はセットで、片方だけでは何も起きません。

## 1. 背景 — ここが本体

最初に躓いたのがここでした。真っ白や単色グレーの背景の上でいくらぼかしても、**ぼかす色がないので灰色の板**にしかなりません。当たり前なのですが、`backdrop-filter` という名前から「それらしくなる魔法」を期待してしまいます。

巨大な `radial-gradient` を数枚重ねて、色のある背景を作ります。

```css
body {
  background-color: #e9eef7;

  background-image:
    radial-gradient(60rem 40rem at 10% -12%,
      rgba(167, 139, 250, 0.55), transparent 60%),
    radial-gradient(50rem 36rem at 94% 4%,
      rgba(244, 114, 182, 0.40), transparent 62%),
    radial-gradient(46rem 34rem at 72% 98%,
      rgba(94, 234, 212, 0.38), transparent 60%);

  background-attachment: fixed;
}
```

ポイントが 3 つあります。

- **とにかく大きく**（`60rem` など）。小さいと「模様」に見えてしまい、ガラス越しに滲む感じが出ません
- **薄く**（`alpha` は 0.4 前後）。濃いと背景そのものが主張します
- **`background-attachment: fixed`**。スクロールしてもブロブが動かないので、カードのほうが「ガラス板として上を滑っていく」ように見えます

## 2. backdrop-filter — `saturate` を忘れない

```css
backdrop-filter: blur(20px) saturate(180%);
```

`blur` だけでも一応ガラスにはなりますが、**`saturate` を足すと一気にそれらしくなります。** ぼかすと色は平均化されて薄くなるので、彩度を持ち上げて戻してやる、という理屈です。Apple の表現が「ぼやけているのに色が濃い」のはこの効果です。

Safari には `-webkit-` プレフィックスが必要です。

```css
-webkit-backdrop-filter: blur(20px) saturate(180%);
backdrop-filter: blur(20px) saturate(180%);
```

## 3〜5. カード本体

```css
.event {
  position: relative;

  background: rgba(255, 255, 255, 0.55);

  -webkit-backdrop-filter: blur(20px) saturate(180%);
  backdrop-filter: blur(20px) saturate(180%);

  border: 1px solid rgba(255, 255, 255, 0.55);
  border-radius: 18px;

  box-shadow:
    0 8px 24px rgba(28, 40, 64, 0.10),
    0 1px 2px rgba(28, 40, 64, 0.06),
    inset 0 1px 0 rgba(255, 255, 255, 0.75);
}
```

影は 3 本重ねています。**広く柔らかい影**で浮かせ、**輪郭のごく近くに薄い影**を置いて接地感を出し、`inset` の 1px で**上端に光の当たった縁**を作ります。この 3 本目が地味に効きます。

## スペキュラエッジ — 縁だけにグラデーションを敷く

ガラスらしさで一番効くのが、**縁が均一に光らず、光源側だけ明るく、反対側へ消えていく**表現です。

`border` に `linear-gradient` は指定できないので、`mask-composite` を使って「縁 1px だけを残す」という手を使います。

```css
.event::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  padding: 1px;

  background:
    linear-gradient(
      150deg,
      rgba(255, 255, 255, 0.95),
      rgba(255, 255, 255, 0.25) 38%,
      rgba(255, 255, 255, 0) 62%
    );

  -webkit-mask:
    linear-gradient(#000 0 0) content-box,
    linear-gradient(#000 0 0);
  -webkit-mask-composite: xor;

  mask:
    linear-gradient(#000 0 0) content-box,
    linear-gradient(#000 0 0);
  mask-composite: exclude;

  pointer-events: none;
}
```

仕組みはこうです。

1. 疑似要素を親と同じ大きさ・同じ角丸で重ねる
2. `padding: 1px` を入れる。これで **border-box と content-box の間に 1px の隙間**ができる
3. マスクを 2 枚指定する。1 枚目は content-box 基準、2 枚目は border-box 基準
4. `mask-composite: exclude` で**両者の差分＝ちょうど縁の 1px だけ**が残る

角丸に沿ってグラデーションが回り込むので、`border` では作れない光沢になります。

**`@supports` で囲むのを忘れないでください。** `mask-composite` に対応していないブラウザでは、マスクが効かず**カード全体が白い板で覆われます**。フォールバックとして最悪です。

```css
@supports ((-webkit-mask-composite: xor) or (mask-composite: exclude)) {
  .event::before { /* ... */ }
}
```

## テーマ色を変数 1 か所にまとめる

ここまでの色を直書きすると、色を変えたいときに 20 箇所直すことになります（実際になりました）。**RGB を三つ組で持ち、`rgb(... / alpha)` で使う**と、1 か所で全部が追従します。

```css
:root {
  --accent-rgb: 124 58 237;
  --accent-deep: #6d28d9;

  --blob-1: rgba(167, 139, 250, 0.55);
  --blob-2: rgba(244, 114, 182, 0.40);
  --blob-3: rgba(94, 234, 212, 0.38);
}

/* 使う側 */
button        { background: rgb(var(--accent-rgb) / 0.88); }
input:focus   { box-shadow: 0 0 0 3px rgb(var(--accent-rgb) / 0.18); }
.badge        { background: rgb(var(--accent-rgb) / 0.14); }
```

ボタン、フォーカスリング、バッジ、影のいずれもが同じ 1 行から派生します。テーマ変更が数値の書き換えだけで済むようになります。

## アクセシビリティとフォールバック

ここは省略されがちですが、**すりガラスは「読みにくさ」と隣り合わせ**なので必要です。

**透明度を下げる設定**（macOS / iOS のアクセシビリティ設定）を尊重します。

```css
@media (prefers-reduced-transparency: reduce) {
  body {
    background-image: none;
    background-color: #eef1f7;
  }

  .panel, .event {
    -webkit-backdrop-filter: none;
    backdrop-filter: none;
    background: rgba(255, 255, 255, 0.96);
  }
}
```

**`backdrop-filter` 非対応**なら、ほぼ不透明にします。半透明のまま放置すると、背景と文字が重なって読めません。

```css
@supports not ((backdrop-filter: blur(1px)) or
               (-webkit-backdrop-filter: blur(1px))) {
  .panel, .event { background: rgba(255, 255, 255, 0.93); }
}
```

**モバイルではぼかし半径を下げます。** blur はスマホの GPU にとって重い処理で、スクロールがもたつく原因になります。

```css
@media (max-width: 640px) {
  :root { --blur: blur(12px) saturate(170%); }
}
```

## 応用: 「今は無効」をガラスで表す

見た目の話だけで終わらせず、**状態の表現**にも使えます。

作ったツールでは、日付を変えたのに再検索していないとき、表示中のリストが「古い結果」になります。文字で警告を出しても読まれないので、**リスト自体を曇りガラス化**しました。

```css
#events.stale {
  opacity: 0.42;
  filter: grayscale(0.85) blur(2px);
  pointer-events: none;
}
```

`blur` を足すのがポイントで、単に薄くするより「触れないもの」に見えます。そして **`pointer-events: none` を必ずセットにしてください。** 薄いだけだと「見えにくいが押せる」状態が残り、古い情報のリンクを踏めてしまいます。見た目と挙動を一致させることで、初めて「無効」が伝わります。

## まとめ

- **背景が主役**。色のある背景を敷かないと `backdrop-filter` は灰色の板を作るだけ
- `blur` に **`saturate` を足す**。ぼかしで抜けた色を戻すと一気にそれらしくなる
- 縁の光沢は **`mask-composite: exclude`** で 1px だけ残す。`@supports` で囲む
- 色は **RGB 三つ組 + `rgb(... / alpha)`** で 1 か所に集約する
- **`prefers-reduced-transparency` と `@supports` のフォールバックを必ず書く**
- モバイルでは blur 半径を下げる

---

このデザインは、Google カレンダーの予定を「いつ登録したか」で一覧するツールに施したものです。作る過程で踏んだ Apps Script まわりの落とし穴は、別の記事にまとめてあります。

- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)
- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
