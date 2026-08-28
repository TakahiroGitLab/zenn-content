---
title: "Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している"
emoji: "📱"
type: "tech"
topics: ["googleappsscript", "gas", "html", "css", "javascript"]
published: true
---

Apps Script で作ったウェブアプリをスマホで開いたら、PC 用のレイアウトがそのまま縮小されて、文字が読めないほど小さく表示される。

`index.html` に viewport のメタタグはちゃんと書いてあるのに、です。

```html
<!-- 書いてあるのに効かない -->
<meta name="viewport" content="width=device-width, initial-scale=1">
```

原因は HTML でも CSS でもありません。**HtmlService がそのタグを消しています。**

## 症状

viewport の指定がないと、モバイルブラウザは仮想的に約 980px 幅でページをレイアウトし、それを画面幅に収まるよう全体縮小して表示します。

つまり、こうなります。

- レイアウトは崩れていない。PC と同じ配置のまま
- ただし全体が一様に小さい
- ピンチで拡大すれば読める
- メディアクエリが発火しない（ブラウザは 980px 幅だと思っているため）

「CSS が効いていない」ように見えますが、実際には**メディアクエリの条件を満たしていないだけ**です。ここが分かりにくい理由でもあります。ブレークポイントを疑って CSS をいじり続けても直りません。

## 原因

[公式ドキュメント](https://developers.google.com/apps-script/reference/html/html-output)にはっきり書かれています。

> Meta tags included directly in an Apps Script HTML file are ignored.

Apps Script の HTML ファイルに直接書いたメタタグは無視される、と。HtmlService がサニタイズの一環として除去します。

自分で書いた HTML に確かに存在するタグが、配信時には消えている。ローカルのエディタを見ているかぎり気づけないので、厄介です。

## 対処

`HtmlOutput.addMetaTag()` を使います。

```javascript
function doGet() {
  return HtmlService
    .createHtmlOutputFromFile('index')
    .setTitle('My App')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}
```

これだけです。HTML 側の `<meta>` は書いても無害ですが効かないので、消しておいたほうが混乱がありません。

Apps Script でスマホから使うウェブアプリを作るなら、**最初に書く 1 行**だと思います。

## 使えるメタタグは 4 つだけ

意外と知られていない制約です。`addMetaTag()` で追加できるのは、以下の 4 つに限られます。

| タグ名 | 用途 |
| --- | --- |
| `viewport` | 表示領域の指定 |
| `apple-mobile-web-app-capable` | iOS でホーム画面に追加したときフルスクリーン化 |
| `mobile-web-app-capable` | 同上（非 Apple 系） |
| `google-site-verification` | サイト所有権の確認 |

`description` や `og:*` は**追加できません**。Apps Script のウェブアプリで OGP を設定してリンクカードを出す、といったことは、この API ではできないということです。

## viewport が効いた後に出てくる問題

ここからは実際に直したときの話です。viewport が正しく効くようになると、今度は**PC 用に決めた数値が窮屈だと分かります**。それまでは全体が縮小されていたので見えていなかった問題が表に出てきます。

地味に効いた調整を 3 つ挙げます。

**入力欄のフォントサイズを 16px 以上にする**

これ未満だと、iOS Safari は入力欄にフォーカスした瞬間**ページ全体を勝手にズーム**します。ユーザーが操作したわけでもないのに画面が拡大され、そのまま戻りません。14px にしていて気づかない、というのがありがちです。

```css
@media (max-width: 640px) {
  input[type="date"] {
    font-size: 16px;   /* これ未満だと iOS がズームする */
  }
}
```

**タップ領域を 44px 以上にする**

`padding: 8px 14px` のボタンは高さが 34px 程度にしかなりません。指で押すには小さいです。`min-height: 44px` を指定するだけで、体感がはっきり変わります。

**`backdrop-filter` のぼかし半径を下げる**

すりガラス風の表現を使っている場合、スマホの GPU では blur がスクロールの負荷になります。PC で 20px にしていたものを 12px 程度に落とすと、見た目をほぼ保ったままスクロールが軽くなります。

```css
@media (max-width: 640px) {
  :root {
    --blur: blur(12px) saturate(170%);
  }
}
```

## 確認の仕方

**必ず実機で見てください。** PC ブラウザのデバイスモードは viewport の指定を尊重するので、`addMetaTag` を書き忘れていても「正しく」表示されてしまうことがあります。問題を再現できません。

Apps Script では、テスト用の `/dev` URL が常に最新コードを配信します。ここをスマホで開くのが確実です。

```bash
clasp push -f
# 「デプロイ > デプロイをテスト」で出る /dev URL をスマホで開く
```

ただし `/dev` URL は**スクリプトの編集権限がある人しか開けません**。自分の端末で確認する分には問題ありませんが、他の人に見てもらうにはデプロイが必要です。

## まとめ

- Apps Script の HTML ファイルに書いた `<meta>` は**無視される**
- viewport は `HtmlOutput.addMetaTag('viewport', ...)` で指定する
- 追加できるメタタグは **4 種類だけ**。`description` や OGP は設定できない
- viewport が効き出すと、入力欄の 16px とタップ領域 44px が次の課題になる
- 確認は実機で。デバイスモードでは問題が再現しない

---

この話は、Google カレンダーの予定を「いつ登録したか」で一覧するツールを作っているときに踏んだものです。Calendar API に作成日時での検索がない件や、`clasp push` してもデプロイが更新されない件も含めて、別の記事にまとめてあります。

[Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

見た目のすりガラス表現についても分けて書いています。

[CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない](https://zenn.dev/takagit/articles/css-liquid-glass-backdrop-filter)

同じツールを Apple Calendar 向けに作り直した話も書いています。

- [Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った](https://zenn.dev/takagit/articles/apple-calendar-entry-log-eventkit)
- [XcodeなしでSwiftUIのMacアプリを作る — .appは手で組めるが、ad-hoc署名だけは省略できない](https://zenn.dev/takagit/articles/swiftui-app-without-xcode)

コードは MIT で公開しています。

https://github.com/TakahiroGitLab/gcal-entry-log
