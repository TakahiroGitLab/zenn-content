---
title: "空白で桁を揃えたメールは崩れる — Gmailはプレーンテキストを等幅で表示しない"
emoji: "📐"
type: "tech"
topics: ["gmail", "googleappsscript", "gas", "email", "html"]
published: false
---

毎週月曜の朝、自分のカレンダーの1週間ぶんをメールで送る Apps Script を書きました。時刻と予定名を並べただけの、素朴なテキストです。

```
Mon 09-22
  09:00    外来
  14:00    術前検討カンファ
  16:30    病棟回診
```

エディタの中では揃っています。実行ログでも揃っています。**Gmail で受け取ると崩れます。**

## 先に書いておくと、これは古い話です

この現象自体は新しい発見ではありません。Gmail が登場して間もない頃から言われていて、[2007年の記事](https://dannyman.toldme.com/2007/03/08/gmail-fixed-width/)があり、受信側で等幅に戻すためのブラウザ拡張が今も複数あります（[gmail-fixed-font](https://github.com/jparise/gmail-fixed-font) など）。メーリングリストで長く暮らしてきた人にとっては、むしろ積年の不満のほうでしょう。

それでも書くのは、**見つかる記事のほとんどが「受信側でどう等幅に戻すか」だから**です。この記事は逆側 — **メールを自動生成する側が、桁揃えを諦めたあとに何を設計するか**の話です。等幅にできない相手にどう読ませるか、という問題は拡張機能では解決しません。

## 仕様も「たいてい」としか言っていない

`text/plain` はフォントを規定しません。[RFC 2646](https://www.rfc-editor.org/rfc/rfc2646.txt) の 2 章にこうあります。

> Text/Plain is usually displayed as preformatted text, often in a fixed font.

`usually` と `often` です。**等幅で表示されることは、仕様上どこにも保証されていません。** 同じ RFC の 3.1 章は、むしろこう続けます。

> Many modern programs use a proportional-spaced font and CRLF to represent paragraph breaks.

Gmail はこの「多くの現代的なプログラム」の側にいます。プレーンテキストのメール本文を、他の UI と同じプロポーショナルフォントで描きます。

つまり「プレーンテキストだから等幅」は、こちらの思い込みでした。

ここで一度、崩れる理由を正確に言っておきます。**空白の幅が変わるからではありません。** 空白はフォントの中では一定の幅を持っていて、4 個並べればいつでも同じ幅です。変わるのは**その手前に置いた文字列の幅**のほうです。

このスクリプトの時刻の列には、2 種類のものが入ります。

```javascript
function startLabel(event, tz) {
  if (!event.start || !event.start.dateTime) {
    return 'all day';
  }

  return Utilities.formatDate(new Date(event.start.dateTime), tz, 'HH:mm');
}
```

`09:00` は 5 文字、`all day` は 7 文字。どちらも 9 文字まで空白で埋めれば、**等幅なら**次の文字はぴったり同じ位置から始まります。比例フォントでは始まりません。`09:00` の 5 文字と `all day` の 7 文字は、そもそも描画幅が違うからです。

**文字数で揃えた列は、ピクセルでは揃わない。** これが起きていることのすべてです。

## 桁揃えをやめて、構造をインデントに移す

最初にやったのは、表をやめることです。

いま書いたことの裏返しとして、**崩れないものが1つあります。行頭の字下げです。**

行頭の空白には手前に何もありません。だから同じ数の空白で始まる行は、比例フォントでも必ず同じ位置から始まります。崩れるのは、幅の違う文字の**後ろ**に置いたものだけです。

つまり「この行は親より内側にある」という関係だけは、そのまま生き残ります。そこで、時刻と予定名を列で並べるのをやめ、日付ごとのブロックにしました。

```
Mon 09-22

  09:00    外来
  14:00    術前検討カンファ

Tue 09-23

  09:00    手術
```

日付が見出しになり、その日の予定が字下げされて続く。**列が揃っていなくても、どの予定がどの日のものかは読めます。** 揃っているように見せるのを諦めて、構造だけ伝えるかたちです。

## 1行に数字の塊を2つ以上置かない

崩れた状態で一番読みにくくなるのは、**数字が連続する行**でした。

```
20260929  2000003  下顎腫瘤切除
```

桁が揃っていれば表として読めますが、揃っていないとどこまでが日付でどこからが ID か分かりません。人間は数字の切れ目を、位置で判断しているからです。

対策は 2 つです。まず**日付を語にしました**。

```
Tue 09-29  下顎腫瘤切除  ID 2000003
```

`20260929` ではなく `Tue 09-29` と書きます。**曜日は数字ではなく語として読める**ので、それだけで隣の数字との境界がはっきりします。

次に、**ID を行末へ移しました。** カレンダーのタイトルでは ID が先頭にあるのですが、そのまま出すと日付のすぐ隣に数字の塊が 2 つ並びます。

```javascript
function caseLine(surgeryCase, tz) {
  const withoutId = String(surgeryCase.title)
    .replace(CONFIG.SURGERY_TITLE_ID, '')
    .trim();

  return [
    dayLabel(surgeryCase.date, tz),
    withoutId || surgeryCase.title,
    surgeryCase.patientId ? 'ID ' + surgeryCase.patientId : 'no ID'
  ].join('  ');
}
```

タイトルから ID を抜いて、行末に付け直しています。`ID` という語を前に置いているのも同じ理由で、**数字の前に語があると、そこが切れ目だと分かります。**

## 終了時刻は行末に置く

予定の時間帯も、最初は `09:00-10:30  外来` と行頭に置いていました。これも数字の塊が 2 つ続きます。

いまは開始時刻だけを行頭に置き、時間帯は行末に回しています。

```
  09:00    外来  (09:00-10:30)
```

冗長に見えますが、こうすると**行頭は常に「時刻ひとつ」だけ**になります。崩れた状態でも、各行の左端に何があるかは一定です。読む人が目で追う列は、そこ 1 本だけあればいい。

## 日をまたぐ予定は、終了日の曜日を出す

これは桁揃えとは別ですが、同じ「読み間違い」の話なので書いておきます。

当直は 21:00 に始まって翌 02:00 に終わります。素直に書くとこうなります。

```
  21:00    当直  (21:00-02:00)
```

**2時間の予定に見えます。** 実際は 5 時間です。

```javascript
const endLabel = sameDay
  ? Utilities.formatDate(end, tz, 'HH:mm')
  : ' ' + Utilities.formatDate(end, tz, 'EEE HH:mm');
```

日付が変わるときだけ、終了側に曜日を付けます。

```
  21:00    当直  (21:00- Sat 02:00)
```

日付をまるごと出さないのは、また数字が増えるからです。**曜日だけで足ります。**

## タイトルの末尾の空白を落とす

細かいですが、実際に踏みました。予定のタイトルは人が手で入力するので、**末尾に空白が入っていることがあります。**

```javascript
return '  ' + padRight(startLabel(event, tz) + mark, 9) + tag +
  // Titles are typed by hand and often carry a trailing space, which
  // would push the span an extra column out.
  (event.summary || '(no title)').trim() +
  spanLabel(event, tz);
```

`trim()` しないと、その予定だけ行末の時間帯が 1 文字ぶん右にずれます。プロポーショナルフォントでも**空白 1 個ぶんは見えます。**

なお `padRight` は残してあります。等幅で表示される環境なら実際に揃うので、あって困るものではありません。**ただしそれに依存していない、というのがこの記事の主旨です。**

## 結局、HTML も送ることにしました

ここまでやって、それでもテキストには限界がありました。最終的には HTML とプレーンテキストの**両方を1通に入れて**送っています。

```javascript
MailApp.sendEmail({
  to: CONFIG.EMAIL_TO || Session.getActiveUser().getEmail(),
  subject: subjectFor(report),
  body: body,
  htmlBody: CONFIG.EMAIL_HTML ? reminderHtml(report) : undefined
});
```

`body` と `htmlBody` の両方を渡すと multipart になり、Gmail は HTML を、描けない環境はテキストを表示します。**テキスト側も同じ内容を全部持っています。**

HTML にしたのは 3 つのためです。

- **見落としてはいけない行を太字にする**（テキストには強調がない）
- **予定へのリンクを張る**
- **時刻の桁を揃える**

3 つ目がまさにこの記事の話で、**テキストでは最後まで解決できませんでした。**

## HTML にしても、まだ揃わなかった

そして、ここが一番面白かったところです。

HTML 側では時刻を固定幅の列に入れました。これで揃うはずでした。

```css
display:inline-block; width:56px;
```

ところが、自分が執刀する手術には `*` を付けていて、これを時刻と同じ列に入れていました。`09:00 *` と `09:00` では、**同じ 56px の箱に入れても中身の幅が違います。** その結果、`*` の付いた行だけ予定名の開始位置が数ピクセル右にずれました。

**HTML にして消したはずのズレが、そのまま戻ってきたわけです。**

```javascript
// The time and the star get a fixed column each. Put the star in with
// the time and a starred row's title begins a few pixels right of an
// unstarred one, which is the misalignment HTML was meant to remove.
const HTML_TIME = 'display:inline-block;width:56px;' + HTML_MUTED;

const HTML_MARK = 'display:inline-block;width:18px;color:#c2410c;' +
  'font-weight:700;';
```

時刻と `*` に、**それぞれ別の固定幅の列**を与えて直しました。

教訓としてはテキストのときと同じです。**1 つの箱に 2 つのものを入れると、揃わなくなる。** 等幅かどうかの問題ではなく、幅が可変なものを並べたときに必ず起きることでした。

## まとめ

- **`text/plain` はフォントを規定していません。** RFC 2646 でさえ "usually"、"often" としか書いていない
- Gmail はプレーンテキストをプロポーショナルフォントで表示する。**古くから知られた話**で、受信側で戻す拡張機能もある
- しかし**送る側**の対処は別問題。**空白で作った列は捨てて、構造をインデントと見出しに移す**
- **1行に数字の塊を2つ以上置かない。** 日付は `20260929` ではなく `Tue 09-29`（曜日は語として読める）、ID は行末に `ID` を添えて置く
- 行頭は「時刻ひとつ」だけにして、時間帯は行末へ
- 手入力のタイトルは `trim()` する。**空白1個でも見える**
- 揃えたいなら HTML を併送する。`body` と `htmlBody` の両方を渡せば multipart になる
- **HTML でも、1つの箱に2つのものを入れると揃わない。** 時刻とマークには別々の固定幅の列を与える

---

同じスクリプトの、Chat を読む側の話も書いています。

[Apps ScriptのChatは既定Cloudプロジェクトのまま読めた — ドキュメントは標準が要ると書いている](https://zenn.dev/takagit/articles/gas-chat-api-gcp-disabled)

Apps Script の他の詰まりどころも書いています。

- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

検証環境: Google Apps Script（V8 ランタイム）、Gmail のウェブ版（2026-09-09）。RFC 2646 の引用は [rfc-editor.org の原文](https://www.rfc-editor.org/rfc/rfc2646.txt)から。
