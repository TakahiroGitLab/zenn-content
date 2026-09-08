---
title: "Apps ScriptのChatは既定Cloudプロジェクトのまま読めた — ドキュメントは標準が要ると書いている"
emoji: "🚧"
type: "tech"
topics: ["googleappsscript", "gas", "googlechat", "googlecloud", "googleworkspace"]
published: true
---

毎週月曜の朝に動く Apps Script を書きました。2 週間後に自分が担当する予定をカレンダーから拾い、それが Google Chat の特定のスペースに**もう投稿されているか**を照合して、まだのものだけを知らせる、というものです。

Chat を読む部分は、**ドキュメントを読んだ時点で一度諦めました。** 前提条件を満たせないと分かったからです。

結論から書くと、**その判断は間違いでした。** 前提条件を満たさないまま呼んだら、普通に動きました。いまは毎週その機能が動いています。

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
function listSpaces() {
  return pagedApi('the list of Chat spaces', 'spaces', function (pageToken) {
    return Chat.Spaces.list({ pageSize: 1000, pageToken: pageToken });
  });
}
```

（`pagedApi()` はページングとリトライをまとめて引き受けている共通処理です。中身は後述します。）

実行ログに、参加しているスペースが 100 件ほど並びました。エラーは出ません。**標準 Cloud プロジェクトは作っていません。作れないままです。** 同じ日に改めてプロジェクト作成を試して、上のエラーが出ることも確認しています。

スペース一覧が読めることと、スペースの**中身**が読めることは別の権限です。そこで続けて `spaces.messages.list` も呼びました。

```javascript
function recentMessages(spaceName) {
  const since = addDays(new Date(), -CONFIG.LOOKBACK_DAYS);

  return pagedApi('the ' + CONFIG.PREOP_SPACE_DISPLAY_NAME + ' space',
    'messages', function (pageToken) {
      return Chat.Spaces.Messages.list(spaceName, {
        filter: 'createTime > "' + since.toISOString() + '"',
        pageSize: 1000,
        pageToken: pageToken
      });
    });
}
```

**498 件の投稿が返りました。** こちらも通ります。

つまり、こうなります。

- 標準 Cloud プロジェクト: **無い**（作成が組織ポリシーで拒否される）
- Chat API 構成ページの Chat アプリ: **無い**（構成ページに到達できない）
- `spaces.list` / `spaces.messages.list` をユーザー認証で呼ぶ: **どちらも通る**

照合機能はこれで動くようになり、いまは毎週のメールに「未提示 / 提示済み」が出ています。**前提条件を読んで諦めた機能が、呼んでみたら最初から使えた**というだけの話でした。

## 前提条件は誰に向けて書かれているか

ドキュメントが間違っている、という話ではないと思っています。読み手を取り違えていた、という話です。

Chat の Advanced Service のページが想定しているのは、**Chat アプリ（ボット）を作る人**です。前提条件の 1 つ目が「Chat API 構成ページで構成された Chat アプリ」であることからしてそうで、標準 Cloud プロジェクトはそのアプリを載せる器として要る。スペースにメッセージを投稿する、スラッシュコマンドに応答する、といった**アプリとして振る舞う**用途なら、この前提は本当に必要です。

一方、こちらがやりたかったのは、**自分の資格情報で、自分が読めるものを読む**ことでした。この場合、呼び出しの主体は Chat アプリではなく自分です。実際、[Chat API の認証ガイド](https://developers.google.com/workspace/chat/api/guides/auth)はユーザー認証とアプリ認証（サービスアカウント）を分けて説明していて、**ユーザー認証の側に標準プロジェクトを要求する記述はありません。**

Advanced Service のページの前提条件は、その 2 つを分けずにまとめて書いてあります。読むだけの用途で読むと、過剰な要求に見える。

## どこまで確かめたか

`spaces.list` と `spaces.messages.list` を、2026 年 9 月 8 日に、`chat.spaces.readonly` と `chat.messages.readonly` を宣言した V8 ランタイムのスクリプトから、Workspace アカウントのユーザー認証で呼びました。認可はプローブの初回実行時に求められ、通っています。

**これは仕様として保証された挙動ではありません。** ドキュメントが要ると書いているものを満たさないまま通っている以上、いつ塞がってもおかしくない。書き込み系（メッセージの投稿など）も試していません。実際に動かして確かめた日の記録として読んでください。

塞がれたときの逃げ道は残してあります。前段で管理者が止められると書いた同じ設定を、オンにしてもらえばいい。標準プロジェクトを作って Apps Script に紐付ける、それが正規の道筋です。OAuth 同意画面は[免除条件](https://support.google.com/cloud/answer/13464323)の "The app is only used by people in your Google Workspace or Cloud Identity organization. The project must be owned by the organization, and its OAuth Consent Screen must be configured for internal use." に当たるので、審査は要りません。

## Advanced Service にはステータスコードが無い

保証のない経路に乗ると決めたので、落ちたときの扱いを決める必要がありました。ここで Advanced Service のもう 1 つの制約に当たります。

Calendar も Chat も、ときどき 5xx を返します。常時動くものなら次の実行で取り返せますが、**週 1 回しか動かないものだと、次に気づくのは 7 日後、カンファはもう終わっています。** なのでリトライを入れました。

問題は、何をリトライすべきかの判定です。**Advanced Service から読めるのはメッセージ文字列だけで、ステータスコードがありません。** 404 と 503 を区別する手がかりが、人間向けの文言しかない。

そこで判定を反転させました。

```javascript
function worthRetrying(err) {
  return !CONFIG.PERMANENT_ERRORS.some(function (pattern) {
    return pattern.test(messageOf(err));
  });
}
```

```javascript
PERMANENT_ERRORS: [
  /not found/i,
  /permission/i,
  /forbidden/i,
  /unauthorized/i,
  /invalid/i,
  /has not been used|is disabled/i
],
```

見分けがつかないときに倒す方向を、**「1 秒無駄にする」対「1 週間失う」**で選んだ、ということです。ホワイトリストではなくブラックリストにしたのは、知らないエラーが来たときに再試行される側に落ちてほしいからです。

最後の 1 行は、この記事の主題そのものです。**`is disabled` は「Google Cloud Platform service has been disabled」にも一致します。** 前提条件を満たさない経路の上に乗っている以上、いつか本当に塞がれる日が来るとしたら、そのエラーはリトライで粘るべきものではありません。そのときはすぐに `WARNING:` としてメールに出る側に倒してあります。

ページングにも同じ話があります。`nextPageToken` が返らなくなるまで回す、というのは、**サービスが渡したばかりのトークンを返してくるまでは**正しい。そうなるとループは 6 分の実行時間上限まで走って、理由を何も残さずに死にます。上限ページ数を設け、同じトークンが 2 回来たことも検出して、どちらの場合もメール冒頭に `WARNING:` を残すようにしました。

## 読めなかった日に「提示済み」と出してはいけない

保証のない経路に乗るというのは、**落ちる日が来る前提で組む**ということです。ここが実装として一番効いた部分でした。

このスクリプトの場合、「未提示の症例はありません」という結果と、「そもそもスペースを読めていない」という結果は、**どちらも “知らせるものがない” という同じ形をしています。** 前者は安心してよく、後者は絶対にしてはいけない。

最初はフラグ 1 つで出力の系統を分けていました。いまは実行時に落ちても同じ表示に落ちます。

```javascript
function preopMessages(caseCount, notes) {
  if (!CONFIG.CHECK_PREOP_SPACE) {
    return {checked: false, messages: null};
  }

  // 照合するものが無いことと、照合できないことは違う。症例が 0 件なら
  // スペースを読む理由がないので、「確認を飛ばした」と言ってはいけない。
  if (!caseCount) {
    return {checked: true, messages: null};
  }

  try {
    return {checked: true, messages: recentMessages(preopSpaceName())};
  } catch (err) {
    notes.push(
      'WARNING: the ' + CONFIG.PREOP_SPACE_DISPLAY_NAME + ' space could ' +
      'not be read (' + String((err && err.message) || err) + '). The ' +
      'cases below are listed without saying whether they have been ' +
      'presented -- please check the space by hand.'
    );

    return {checked: false, messages: null};
  }
}
```

リトライで直らない失敗もあります。スペースの id が変わった、スペースから外された、スコープの再同意が要る、そして**この記事の経路そのものが塞がれた**。以前はそのどれもが実行全体を終わらせていました。つまり、スペース 1 つが答えなくなると、症例一覧も週の予定も道連れになる。**メール 1 通を、その 1 列のために捨てていた**わけです。

いまは `checked: false` を返して、メールは出ます。冒頭に理由が `WARNING:` で入り、症例欄は「確認していないので手で見てください」の表示に落ちます。

**倒れる向きが要点です。読めなかったときに「提示済み」と出ることは、絶対にありません。** 全症例が判定なしで並び、確認を促される。そこだけを固定するテストを書いてあります。

冒頭の `if (!caseCount)` はもう一段細かい区別です。関数のコメントにある通り、**症例が 0 件で読まなかったこと**と、**読もうとして失敗したこと**は別の状態で、前者を「確認を飛ばしました」という警告にしてしまうと、無い問題を報告することになります。

## まとめ

- Apps Script の Advanced Service は、通常は既定の Cloud プロジェクトのままでいい
- **Chat のページだけ、前提条件に標準 Cloud プロジェクトと Chat アプリの構成を要求している**
- ただしそれは Chat アプリ（ボット）を作る場合の前提。**ユーザー認証で読むだけなら、既定プロジェクトのまま `spaces.list` も `spaces.messages.list` も通った**（2026-09-08 実測、標準プロジェクトは作れない状態のまま）
- Workspace 管理者は「その他の Google サービス」で GCP そのものをオフにできる。オフだと標準プロジェクトは作れない
- **前提条件を読んで諦める前に、呼んでみる。** 満たせない前提が、その呼び出しに本当に要るとは限らない
- Advanced Service の例外にはステータスコードが無い。**リトライ判定はホワイトリストではなくブラックリスト**にして、見分けがつかないものは再試行する側へ倒す
- 保証された挙動ではないので、落ちた日の出力を決めておく。**読めなかったときに「問題なし」と読める表示にしないこと**

---

Apps Script の他の詰まりどころも書いています。

- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

検証環境: Google Apps Script（V8 ランタイム）、clasp 3.3.0、Google Workspace（Google Cloud Platform は 2026-09-09 時点でも管理者によりオフのまま。設定が変わっていないことは管理者に確認済み）。
