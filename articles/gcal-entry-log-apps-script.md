---
title: "Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った"
emoji: "📅"
type: "tech"
topics: ["googleappsscript", "gas", "googlecalendar", "clasp", "javascript"]
published: false
---

Google カレンダーには「その予定が**いつ登録されたか**」で一覧する画面がありません。あるのは「いつ行われるか」で並ぶビューだけです。

外来の合間に入力した予定を、その日の終わりにまとめて見返したくて、Apps Script で作りました。

- **デモ**: https://takahirogitlab.github.io/gcal-entry-log/ （サンプルデータで動きます。Google アカウント不要）
- **ソース**: https://github.com/TakahiroGitLab/gcal-entry-log （MIT）

期間を指定すると、その期間に**登録された**予定が作成日時順に出ます。さらに「自分が作成したもの」と「他人に招待されたもの」をチェックボックスで切り替えられます。

この記事は、作る過程で詰まった Google Calendar API と Apps Script の仕様について書きます。同じところで止まる人がいると思うので。

## なぜこの軸が必要だったか

私は医師ですが、外来中は患者さんが次々に来るので、予定の入力は診察の合間に手で行うことになります。

そこで起きるのが、**入力した端から何を入力したか忘れる**という状態です。

そしてこれは記憶力だけの問題ではありませんでした。入力した内容を後からまとめて見返す手段がないと、「あとで内容を確認しよう」「あとで詳細を追記しよう」と思っていたことが、**そのまま思い出されずに終わります**。振り返る画面がないので、振り返るきっかけ自体が発生しないのです。

カレンダーを開いても、今日入力した予定は**未来の日付に散らばっています**。数週間先、数か月先にばらばらに存在するものを、まとめて確認することはできません。

欲しいのは「今日という日に、自分は何をカレンダーに入れたか」という一覧でした。これは作成日時を軸にしないと作れません。

### 他人が追加した予定も網羅的に確認したい

もう一つ要求がありました。**他人が自分を参加者として追加した予定**をまとめて確認したい、というものです。

自分が入れた予定は、自分の入力ミスを探す対象です。一方で他人が入れた予定は、そもそも自分が把握していない予定なので、内容に間違いがないか、日程に無理がないかを確認する必要があります。**見る目的が違います。**

漏れなく確認できることが重要なので、期間で区切って一覧できる形にしました。チェックボックスを 2 つ置いて、片方だけでも両方でも表示できます。この 2 つを混ぜずに切り替えられることが、実際に使ってみると想像以上に効きました。

## 詰まりどころ 1: Calendar API は作成日時で検索できない

これが最初の壁でした。`Calendar.Events.list` のパラメータを見ると:

- `timeMin` / `timeMax` — **開催日時**で絞る
- `updatedMin` — **最終更新日時**で絞る
- `created` で絞るものは **ない**

`created` はレスポンスには含まれるのに、クエリ条件にはできません。

### 回避策

`updatedMin` を範囲の開始日時に設定して広めに取り、`created` でクライアント側（正確にはサーバースクリプト側）で絞り込みます。

```javascript
let pageToken;
const events = [];

do {
  const response = Calendar.Events.list('primary', {
    // 予定は必ず「作成日時 <= 更新日時」なので、
    // updatedMin を範囲の開始に置けば取りこぼさない
    updatedMin: rangeStart.toISOString(),
    showDeleted: false,
    singleEvents: false,
    maxResults: 2500,
    pageToken: pageToken
  });

  if (response.items) {
    events.push(...response.items);
  }
  pageToken = response.nextPageToken;
} while (pageToken);

const matched = events.filter(event => {
  if (!event.created) return false;
  const created = new Date(event.created);
  return created >= rangeStart && created < rangeEndExclusive;
});
```

**なぜ取りこぼさないか**: ある予定が期間内に作成されたなら、その予定の `updated` は必ず作成日時以降です。つまり `updatedMin = 範囲の開始` で取得した集合は、求める集合を必ず含みます。

**代償**: 範囲の開始を過去に遡らせるほど、「それ以降に更新されたすべての予定」を取ってくることになります。1 週間なら問題ありませんが、数か月遡ると重くなります。Apps Script の実行時間上限は 1 回 6 分なので、そこに当たる可能性があります。

用途が「直近の確認」なので実用上は困っていませんが、汎用ツールとして作るならキャッシュなどの工夫が要ると思います。

### ついでに: 「いつ招待されたか」は取れない

「他人に招待されたもの」も同じく `created` で絞っています。ただしこれは**予定そのものの作成日時**であって、**自分が参加者に追加された日時**ではありません。

API に後者を取る手段がないためです。予定が 1 日に作られ、自分が 14 日に追加された場合、14 日を含む期間では拾えません。仕様として README にも書いてあります。

## 詰まりどころ 2: HTML の `<meta>` タグが消される

スマホで開いたら、PC 用のレイアウトがそのまま縮小表示されて文字が読めませんでした。

`index.html` に viewport を書いてあるのに効きません。

```html
<!-- これは効かない -->
<meta name="viewport" content="width=device-width, initial-scale=1">
```

**HtmlService は HTML ファイル内の `<meta>` タグを除去します。** サニタイズの一環です。viewport の指定がないので、モバイルブラウザは仮想的に約 980px 幅でレイアウトしてから全体を縮小表示します。だから「PC の画面がそのまま小さくなった」ように見えます。

正しくは `addMetaTag()` を使います。

```javascript
function doGet() {
  return HtmlService
    .createHtmlOutputFromFile('index')
    .setTitle('Calendar Entry Log')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}
```

これ 1 行で見え方が一変しました。Apps Script でスマホ対応の Web アプリを作るなら**最初に入れるべき行**だと思います。

### ついでのモバイル調整

viewport が効くようになると、今度は PC 用の数値が窮屈になります。メディアクエリで調整しました。地味に効いたのは:

- **日付入力を 16px 以上にする** — これ未満だと iOS が focus 時にページを勝手にズームします
- **タップ領域を 44px 以上確保する** — ボタンの高さが 34px しかありませんでした
- **`backdrop-filter` のぼかし半径を下げる** — スマホの GPU では blur がスクロールの負荷になります

## 詰まりどころ 3: push してもデプロイは更新されない

clasp で開発していると必ず一度は悩むところだと思います。

```bash
clasp push -f     # コードは更新された
```

なのに、共有している `/exec` の URL を開いても**古いまま**です。

`clasp push` が更新するのは **HEAD**（最新の保存状態）だけです。ウェブアプリの `/exec` URL は**特定のバージョンに固定**されており、バージョンは不変のスナップショットです。push しても、既存デプロイが指すバージョンは変わりません。

反映するには、同じデプロイ ID に対して新しいバージョンを作ります。

```bash
# デプロイ ID を確認
clasp list-deployments

# 同じ ID を指定すると URL を変えずに中身だけ更新される
clasp create-deployment -i <deployment-id> -d "v2"
```

`-i` を付け忘れると**新しいデプロイ（＝新しい URL）**ができてしまい、共有済みの URL は古いままです。ここは間違えやすいところです。

なお `@HEAD` のデプロイに対応する `/dev` URL は常に最新コードを配信します。動作確認は `/dev`、本番反映は `create-deployment -i`、と使い分けると安全です。

## 詰まりどころ 4: `executeAs` の意味を理解しておく

セキュリティに直結するので書いておきます。マニフェストの設定です。

```json
"webapp": {
  "executeAs": "USER_ACCESSING",
  "access": "DOMAIN"
}
```

- **`executeAs: USER_ACCESSING`** — スクリプトは**アクセスした人の権限**で動きます。`Calendar.Events.list('primary')` はその人自身のカレンダーを読みます
- **`executeAs: USER_DEPLOYING`** — スクリプトは**デプロイした人の権限**で動きます

このツールは前者です。だから `access` を広げても、他人に自分のカレンダーが見えることはありません。逆に言えば、**この安全性は `access` ではなく `executeAs` が担保しています**。

ドロップダウン 1 つを切り替えた瞬間、全訪問者に自分のカレンダーが表示されるアプリになります。カレンダーやメールを扱う Apps Script では、ここを最初に確認したほうがいいです。

## UI で考えたこと

### チェックボックスの toggle で毎回サーバーに行かない

最初の実装は、役割のチェックボックスを切り替えるたびにサーバー関数を呼んでいました。当然、毎回 Calendar API のページングをやり直すので待たされます。すでに持っているデータに対するフィルタなのに、です。

サーバーは常に両方の役割を返し、クライアントが取得済みのデータを絞り込む形に変えました。

```javascript
// 役割の toggle はキャッシュを描画し直すだけ。
// 範囲が変わったときだけカレンダーを再検索する。
function onRoleChange() {
  if (cacheMatchesInputs()) {
    render();
    return;
  }
  loadEvents();
}
```

### 「古い結果」を古いと分からせる

上の変更で新しい問題が出ました。**日付を変えて Refresh を押さないと、前の期間の結果が最新のものとして画面に残ります。**

最初は警告文を出すだけにしましたが、これは見逃されます。文字は読まれません。

そこで、日付入力が表示中の結果と食い違ったら:

- リストを `opacity: 0.42` + `grayscale` + `blur(2px)` で**曇りガラス化**する
- `pointer-events: none` で**クリックできなくする**（古い予定のリンクを踏めないように）
- Refresh ボタンをオレンジにして脈打たせる

の 3 つを同時に起こすようにしました。

`pointer-events: none` を入れたのがポイントで、薄くしただけだと「見えにくいけど押せる」状態が残ります。「今は無効」を操作でも示したほうが確実です。

### 余談: CSS の "liquid glass" は背景とセット

見た目は iOS/macOS 風のガラスにしました。`backdrop-filter: blur(20px) saturate(180%)` が中心ですが、**背後に色がないとただの灰色板になります。**

背景に大きな `radial-gradient` を数枚敷いて初めてガラスに見えます。ぼかす対象がなければぼかしても意味がない、というだけの話なのですが、最初これに気づかずに「なんか濁ってるだけだな」と思っていました。

`prefers-reduced-transparency`（macOS/iOS の「透明度を下げる」設定）でのフォールバックも入れてあります。

## まとめ

- Calendar API は**作成日時で検索できない**。`updatedMin` で広く取って `created` で絞る
- HtmlService は `<meta>` を消す。viewport は **`addMetaTag()`** で
- `clasp push` は HEAD を更新するだけ。`/exec` の反映は **`create-deployment -i <id>`**
- `executeAs: USER_ACCESSING` が安全性を担保している。変えてはいけない

コード全体は GitHub にあります。MIT なので、同じ困りごとがある方はそのまま使ってください。

https://github.com/TakahiroGitLab/gcal-entry-log
