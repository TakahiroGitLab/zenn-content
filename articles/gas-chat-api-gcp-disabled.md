---
title: "Apps ScriptのChatは既定Cloudプロジェクトのまま読めた — ドキュメントは標準が要ると書いている"
emoji: "🚧"
type: "tech"
topics: ["googleappsscript", "gas", "googlechat", "googlecloud", "googleworkspace"]
published: false
---

毎週月曜の朝に動く Apps Script を書きました。2 週間後に自分が担当する予定をカレンダーから拾い、それが Google Chat の特定のスペースに**もう投稿されているか**を照合して、まだのものだけを知らせる、というものです。

Chat を読む部分は、**ドキュメントを読んだ時点で一度諦めました。** 前提条件を満たせないと分かったからです。

結論から書くと、**その判断は間違いでした。** 前提条件を満たさないまま呼んだら、普通に動きました。

## ドキュメントは標準 Cloud プロジェクトを要求する

Apps Script の Advanced Service は、エディタの「サービス」から足すだけで使えます。裏で既定の Cloud プロジェクトが自動で作られ、API の有効化までついてくる。[拡張サービスのドキュメント](https://developers.google.com/apps-script/guides/services/advanced)にもこうあります。

> If using a default Google Cloud project (created automatically by Apps Script), skip this step. The API is enabled automatically when you add the service in Step 1.

Calendar も Drive も Sheets もこれで足ります。ところが [Chat の Advanced Service のページ](https://developers.google.com/apps-script/advanced/chat)だけ、前提条件にこう書かれています。

> An Apps Script Google Chat app configured on the Chat API configuration page in the Google Cloud console. The app's Apps Script project must use a standard Google Cloud project instead of the default one created automatically for Apps Script projects.

2 つ要求されています。**標準 Cloud プロジェクト**を使うこと、そして Cloud コンソールの **Chat API 構成ページで構成された Chat アプリ**であること。後者は[クイックスタート](https://developers.google.com/workspace/chat/quickstart/apps-script-app)を見ると、アプリ名・アバター URL・説明・接続設定を埋める手続きです。つまり Chat「アプリ」を定義しろ、と。

ボットを作りたいわけではなく、自分の権限で自分が参加しているスペースを読みたいだけでした。それでも前提条件は前提条件です。

## その標準 Cloud プロジェクトが作れない

作ろうとすると、こうなります。

```
Google Cloud Platform service has been disabled.
Please contact your administrator to turn the service on in the
Google Workspace Admin console.
```

Workspace の管理者は、Google Cloud そのものを組織ごと止められます。[管理コンソールの「その他の Google サービス」](https://support.google.com/a/answer/181865?hl=ja)です。

```
管理コンソール → メニュー → アプリ → その他の Google サービス
  → Google Cloud Platform → サービスのステータス
  → オン（すべてのユーザー）／オフ（すべてのユーザー）
```

組織部門ごとに設定できるので、「一部の部署だけオフ」という状態もあります。オフの配下にいると、プロジェクトが作れません。

ここで詰みました。前提条件の 1 つ目が満たせない以上、2 つ目の構成ページには進みようがない。**そう判断して、照合機能を止めました。**

## 前提条件を満たさないまま呼んだら、通った

しばらくして、確かめずに諦めていることに気づいて、実行してみました。マニフェストは Chat を宣言したままです。

```json
{
  "dependencies": {
    "enabledAdvancedServices": [
      { "userSymbol": "Calendar", "serviceId": "calendar", "version": "v3" },
      { "userSymbol": "Chat",     "serviceId": "chat",     "version": "v1" }
    ]
  },
  "oauthScopes": [
    "https://www.googleapis.com/auth/chat.spaces.readonly",
    "https://www.googleapis.com/auth/chat.messages.readonly"
  ]
}
```

呼んだのはスペース一覧です。

```javascript
function probeSpaces() {
  let pageToken;

  do {
    const response = Chat.Spaces.list({ pageSize: 1000, pageToken: pageToken });

    (response.spaces || []).forEach(function (space) {
      console.log(space.name, space.spaceType, space.displayName || '(no name)');
    });

    pageToken = response.nextPageToken;
  } while (pageToken);
}
```

実行ログに、参加しているスペースが 100 件ほど並びました。エラーは出ません。**標準 Cloud プロジェクトは作っていません。作れないままです。** 同じ日に改めてプロジェクト作成を試して、上のエラーが出ることも確認しています。

つまり、こうなります。

- 標準 Cloud プロジェクト: **無い**（作成が組織ポリシーで拒否される）
- Chat API 構成ページの Chat アプリ: **無い**（構成ページに到達できない）
- `Chat.Spaces.list()` をユーザー認証で呼ぶ: **通る**

## 前提条件は誰に向けて書かれているか

ドキュメントが間違っている、という話ではないと思っています。読み手を取り違えていた、という話です。

Chat の Advanced Service のページが想定しているのは、**Chat アプリ（ボット）を作る人**です。前提条件の 1 つ目が「Chat API 構成ページで構成された Chat アプリ」であることからしてそうで、標準 Cloud プロジェクトはそのアプリを載せる器として要る。スペースにメッセージを投稿する、スラッシュコマンドに応答する、といった**アプリとして振る舞う**用途なら、この前提は本当に必要です。

一方、こちらがやりたかったのは、**自分の資格情報で、自分が読めるものを読む**ことでした。この場合、呼び出しの主体は Chat アプリではなく自分です。実際、[Chat API の認証ガイド](https://developers.google.com/workspace/chat/api/guides/auth)はユーザー認証とアプリ認証（サービスアカウント）を分けて説明していて、**ユーザー認証の側に標準プロジェクトを要求する記述はありません。**

Advanced Service のページの前提条件は、その 2 つを分けずにまとめて書いてあります。読むだけの用途で読むと、過剰な要求に見える。

## どこまで確かめたか

確かめたのは `Chat.Spaces.list()` が 1 回通った、という事実だけです。日付は 2026 年 9 月 8 日、`chat.spaces.readonly` と `chat.messages.readonly` を宣言した V8 ランタイムのスクリプト、Workspace アカウント。

`spaces.messages.list` はまだ呼んでいません。スペース一覧が読めることと、スペースの中身が読めることは別の権限です。**ここを確かめずに「Chat は読める」と一般化するのは、前提条件を読んで諦めたのと同じ間違いです。**

そして、これは仕様として保証された挙動ではありません。ドキュメントが要ると書いているものを満たさずに通っている以上、いつ塞がってもおかしくない。実際に動かして確かめた日の記録として読んでください。

## 「確認していない」を「問題なし」と同じ形で出さない

照合を止めていた期間に決めたことが 1 つあって、これは Chat が読めるようになっても残します。

このスクリプトの場合、「未提示の予定はありません」という結果と、「そもそも照合していない」という結果は、**どちらも “知らせるものがない” という同じ形をしています。** 前者は安心してよく、後者は絶対にしてはいけない。

なので、フラグ 1 つで出力の系統ごと分けました。

```javascript
if (report.checkedSpace) {
  appendPresentationStatus(lines, report);   // 未提示 / 提示済み に分ける
} else {
  appendCasesOnly(lines, report);            // 並べて、確認していないと言う
}
```

照合できなかったときは、見出しが「未提示」ではなく「自分の担当分」に変わり、末尾にこれが付きます。

```javascript
lines.push(
  'NOTE: the ' + CONFIG.PREOP_SPACE_DISPLAY_NAME + ' space was not ' +
  'checked, so whether these have been presented is unknown. Please ' +
  'confirm by hand.'
);
```

これは諦めのための仕組みではなく、**API 呼び出しが落ちた日にも要る**ものです。前提条件を満たしていない以上、いつ落ちてもおかしくないので、むしろ今のほうが要ります。

## まとめ

- Apps Script の Advanced Service は、通常は既定の Cloud プロジェクトのままでいい
- **Chat のページだけ、前提条件に標準 Cloud プロジェクトと Chat アプリの構成を要求している**
- ただしそれは Chat アプリ（ボット）を作る場合の前提。**ユーザー認証で読むだけなら、既定プロジェクトのまま `Chat.Spaces.list()` が通った**（2026-09-08 実測、標準プロジェクトは作れない状態のまま）
- Workspace 管理者は「その他の Google サービス」で GCP そのものをオフにできる。オフだと標準プロジェクトは作れない
- **前提条件を読んで諦める前に、呼んでみる。** 満たせない前提が、その呼び出しに本当に要るとは限らない
- 保証された挙動ではないので、落ちた日に何を出力するかは決めておく

---

Apps Script の他の詰まりどころも書いています。

- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

検証環境: Google Apps Script（V8 ランタイム）、clasp 3.3.0、Google Workspace（Google Cloud Platform は管理者によりオフのまま）。
