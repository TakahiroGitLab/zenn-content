---
title: "Apps ScriptからGoogle Chatを読めない — Chatだけ標準Cloudプロジェクトが要る"
emoji: "🚧"
type: "tech"
topics: ["googleappsscript", "gas", "googlechat", "googlecloud", "googleworkspace"]
published: false
---

毎週月曜の朝に動く Apps Script を書きました。やることは 2 つです。

1. 2 週間後に自分が担当する予定をカレンダーから拾う
2. それが Google Chat の特定のスペースに**もう投稿されているか**を照合し、まだのものだけを知らせる

1 は動きました。2 は、**1 行も実行できないまま終わりました。**

止まったのは Chat API の使い方ではありません。その手前、**Cloud プロジェクトを作るところ**です。

## 症状

Apps Script エディタの「サービス」から Chat を追加します。追加できます。マニフェストにも入ります。

```json
{
  "dependencies": {
    "enabledAdvancedServices": [
      { "userSymbol": "Calendar", "serviceId": "calendar", "version": "v3" },
      { "userSymbol": "Chat",     "serviceId": "chat",     "version": "v1" }
    ]
  },
  "oauthScopes": [
    "https://www.googleapis.com/auth/calendar.readonly",
    "https://www.googleapis.com/auth/chat.spaces.readonly",
    "https://www.googleapis.com/auth/chat.messages.readonly"
  ]
}
```

コードも書けます。`clasp push` も通ります。ここまで、何ひとつ怒られません。

ドキュメントを読み進めると、Chat の Advanced Service には**標準 Cloud プロジェクトが要る**と書いてあります。ではプロジェクトを作ろう、と Cloud コンソールへ行くと、そこで止まります。

```
Google Cloud Platform service has been disabled
```

管理者に連絡してください、と。作れないので、その先の手順は 1 つも実行できません。

## Advanced Service は普通、Cloud プロジェクトを意識しない

ここが盲点でした。

Apps Script の Advanced Service は、エディタから足すだけで使えます。裏で既定の Cloud プロジェクトが自動で作られ、API の有効化までついてきます。[公式ドキュメント](https://developers.google.com/apps-script/guides/services/advanced)にもこうあります。

> If using a default Google Cloud project (created automatically by Apps Script), skip this step. The API is enabled automatically when you add the service in Step 1.

Calendar も Drive も Sheets もこれで足ります。実際、このスクリプトの Calendar 側は最後まで既定プロジェクトのままで動いています。

だから「Chat も同じだろう」と思って進みます。**マニフェストに書けてしまうので、なおさらそう思います。**

## Chat だけが例外

[Chat の Advanced Service のページ](https://developers.google.com/apps-script/advanced/chat)には、前提条件としてこう書かれています。

> An Apps Script Google Chat app configured on the Chat API configuration page in the Google Cloud console. The app's Apps Script project must use a standard Google Cloud project instead of the default one created automatically for Apps Script projects.

2 つ要求されています。

- **標準 Cloud プロジェクト**を使うこと（Apps Script が自動で作る既定のものではなく）
- Cloud コンソールの **Chat API 構成ページで構成された Chat アプリ**であること

後者が地味に重くて、[クイックスタート](https://developers.google.com/workspace/chat/quickstart/apps-script-app)を見ると、構成ページで埋めるのはアプリ名・アバター URL・説明・接続設定です。つまり **Chat「アプリ」を定義する手続き**です。

こちらはボットを作りたいわけではなく、自分の権限で自分が参加しているスペースを読みたいだけでした。それでも前提条件からは外れられません。Chat API では、読む側もアプリとして登録されている必要がある、という構えになっています。

要するに、**Chat だけは「Advanced Service を足す」では終わらない**。既定プロジェクトのまま完結する他のサービスと、ここが違います。

## 管理者は GCP そのものをオフにできる

そして標準 Cloud プロジェクトを作るには、当然ながら Google Cloud が使えなければなりません。

Workspace の管理者は、これを組織ごと止められます。[管理コンソールの「その他の Google サービス」](https://support.google.com/a/answer/181865?hl=ja)です。

```
管理コンソール → メニュー → アプリ → その他の Google サービス
  → Google Cloud Platform → サービスのステータス
  → オン（すべてのユーザー）／オフ（すべてのユーザー）
```

組織部門ごとに設定できるので、「一部の部署だけオフ」という状態もあります。オフになっていると、その配下のユーザーは Cloud プロジェクトを作れません。冒頭のエラーはこれです。

**費用が理由とは限りません。** [Calendar API のクォータのページ](https://developers.google.com/workspace/calendar/api/guides/quota)には "All standard use of the Google Calendar API is available at no additional cost." とあります（ただし上限を超えた分の課金は 2026 年後半に予定されている、とも書かれています）。通常の利用の範囲では、止める理由が請求書になるわけではない、ということです。

なので、依頼するとしたら**頼むことは一点だけ**になります。

- 管理コンソール → アプリ → その他の Google サービス → **Google Cloud Platform をオン**
- 対象は組織部門で絞れる（全社に開ける必要はない）
- OAuth 同意画面を**「内部」**にすれば、Google の審査は要らない — [審査の免除条件](https://support.google.com/cloud/answer/13464323)に "The app is only used by people in your Google Workspace or Cloud Identity organization. The project must be owned by the organization, and its OAuth Consent Screen must be configured for internal use." とあります。**プロジェクトが組織所有であること**まで含めて条件です

自分はこれが通りませんでした。以下は、通らなかった側の話です。

## 諦めるときに、何を出力するか

ここからは実際に手を動かした部分です。

機能を 1 つ落とすときにいちばんまずいのは、**落としたことが出力から分からない**ことだと思っています。

このスクリプトの場合、照合をやめると何が起きるか。「未提示の予定はありません」という結果と、「そもそも照合していない」という結果は、**どちらも “知らせるものがない” という同じ形をしています。** 前者は安心してよく、後者は絶対にしてはいけない。

なので、フラグ 1 つで出力の系統を分けました。

```javascript
// Reading the space needs the Chat advanced service, which needs a
// standard Cloud project, which this Workspace's admin has switched
// off ("Google Cloud Platform service has been disabled").
CHECK_PREOP_SPACE: false,
```

`true` のとき（本来やりたかったこと）は、こう出ます。

```
== Not presented yet: 1 ==
  - Tue 09-22  ...

== Already presented: 3 ==
  - ...
```

`false` のときは、**同じ画面を痩せさせるのではなく、別の文面にします。**

```javascript
/**
 * The space could not be consulted, so this lists the cases and says so
 * rather than implying anything about whether they were presented.
 */
function appendCasesOnly(lines, report) {
  lines.push('== Your cases: ' + report.mine.length + ' ==');

  report.mine.forEach(function (r) {
    lines.push('  - ' + caseLine(r.surgeryCase, report.tz));
  });

  lines.push('');
  lines.push(
    'NOTE: the ' + CONFIG.PREOP_SPACE_DISPLAY_NAME + ' space was not ' +
    'checked, so whether these have been presented is unknown. Please ' +
    'confirm by hand.'
  );
}
```

見出しが「未提示」ではなく「自分の担当分」に変わり、末尾に「照合していないので手で確認してください」が付く。**確認していないことを、確認した結果と同じ形で出さない。** これだけです。

分岐も、出力を組み立てる側で 1 回だけ持たせています。

```javascript
if (report.checkedSpace) {
  appendPresentationStatus(lines, report);
} else {
  appendCasesOnly(lines, report);
}
```

もう一つ決めたのは、**動かせないコードを消さないこと**です。Chat を読む `PreopPosts.js` は書いたまま残してあり、README に「一度も実行していない」と書いてあります。消せば、いつか GCP が開いたときにまた調べ直すことになる。残すなら、動作を保証していないことが分かるようにしておく必要がある。この 2 つは両立します。

## まとめ

- Apps Script の Advanced Service は、**普通は既定の Cloud プロジェクトのままで足りる**
- **Chat だけは標準 Cloud プロジェクトを要求する。** 自分の権限で読むだけでも、Chat API 構成ページでアプリとして構成する必要がある
- マニフェストには**書けてしまう**。書けた ＝ 使える、ではない
- Workspace 管理者は「その他の Google サービス」で **GCP そのものをオフにできる**。オフだと標準プロジェクトが作れず、Chat は手前で詰む
- 頼むことは「GCP をオンにする」の一点。組織部門で絞れて、OAuth 同意画面が「内部」（かつプロジェクトが組織所有）なら Google の審査も要らない
- 機能を落としたら、**出力にそう書く**。「確認して何もなかった」と「確認していない」を同じ見た目にしない

Chat が読めるようになったら、この記事には続きが要ります。いまのところ、前提条件のところで止まっています。

---

Apps Script の他の詰まりどころも書いています。

- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

検証環境: Google Apps Script（V8 ランタイム）、clasp 3.3.0、Google Workspace（Google Cloud Platform が管理者によりオフ）。
