---
title: "Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った"
emoji: "🍎"
type: "tech"
topics: ["swift", "swiftui", "eventkit", "macos", "applescript"]
published: true
---

以前、Google カレンダーの予定を「**いつ登録したか**」で一覧するツールを Apps Script で作りました。

[Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

同じものが Apple Calendar 側にも欲しくなったので、EventKit で作り直しました。SwiftUI の Mac アプリと、同じ内容を吐くコマンドラインの 2 つです。

![Entry Log のウィンドウ。期間・プリセット・役割・カレンダーの絞り込みと、作成日時順に並んだ予定の一覧](/images/entry-log-demo.png)
*架空のデータを表示するデモモードのスクリーンショットです。理由は後述します。*

**ソース**: https://github.com/TakahiroGitLab/apple-cal-entry-log （MIT）

移植というより作り直しでした。**やりたいことは同じなのに、詰まる場所が一つも重なりません。** この記事はその詰まりどころの記録です。

## 何を作ったか

期間を指定すると、その期間に**登録された**予定が作成日時順に並びます。自分が書いたものか、招待されたものかも出ます。ここまでは Google 版と同じです。

違うのは、Web アプリではなくローカルのアプリだという点です。

- カレンダーは iCloud にありますが、読むのは Mac 上の EventKit 経由
- ネットワークに何も出ません
- 行をダブルクリックすると Calendar.app の該当イベントが開きます
- どのカレンダーを一覧に含めるかをチェックボックスで選べます

Xcode は使っていません。この点は最後に触れます。

## 詰まりどころ 1: EventKit も作成日時では検索できない。しかも 1 クエリ 4 年まで

Google Calendar API に `created` での検索がなかったのと同じで、**EventKit も開催日時でしか検索できません。**

```swift
store.predicateForEvents(withStart: start, end: end, calendars: calendars)
```

指定できるのは「いつ**行われる**か」だけです。`creationDate` はプロパティとして存在するのに、述語には使えません。

なので Google 版と同じ方針になります。**開催日時で広めに取り、`creationDate` で後から絞る。**

ここで固有の制約が出てきます。**1 つの述語は 4 年を超える期間を張れません。**

### 一度、プロジェクトを畳みかけた

この制限を私は「**EventKit は 4 年しか検索できない**」と読みました。

そう読むと詰みです。10 年前に登録した予定は原理的に見つからず、しかも「見つからなかった」ことがユーザーには分かりません。**取りこぼしが黙って起きる道具**は作る意味がないので、中断を宣言しました。

これは誤読でした。制限は**述語 1 本あたり**にかかるもので、述語を何本投げるかには何の制約もありません。読み替えれば、ただのループです。

```swift
public struct FetchPlanner: Sendable, Equatable {

    /// EventKit refuses a single predicate spanning more than four years.
    public static let maximumWindowInMonths = 48

    /// A decade either side.
    public static let standard = FetchPlanner(yearsBefore: 10, yearsAfter: 10)

    // 指定された期間の前後 10 年を、4 年以下の窓に切り分ける
    private func split(from start: Date, to end: Date, calendar: Calendar)
        -> [FetchWindow]
    { /* ... */ }
}
```

前後 10 年で **6 クエリ**。ローカルの数千件を舐めるだけなので **0.2 秒程度**です。「4 年」という数字を見た瞬間に諦めなくてよかった、というのがこのプロジェクト最大の学びでした。

### 窓は隣接させるが、重複は必ず出る

窓は重ねずに繋げていますが、それでも重複します。**EventKit は窓に「重なる」予定を返す**ので、境界をまたぐ予定は両方の窓から返ってきます。

ただしこれは困りません。**繰り返し予定のために、どのみち重複排除が必要**だからです。

```swift
// 繰り返し予定は各回が別の EKEvent として列挙されるが、
// calendarItemIdentifier は全回で共通。
// 「いつ書いたか」の一覧なので、週1の外来は1回書いたもの。
collected.add(event)   // 内部で calendarItemIdentifier をキーに畳む
```

繰り返し予定の全回が同じ識別子・同じ作成日時を持つ、という性質はここでは味方です。**あとで敵になります**（詰まりどころ 5）。

## 詰まりどころ 2: 認可の前に作った EKEventStore は、認可後も空を返し続ける

これが一番デバッグしづらい種類のバグでした。

**コマンドラインでは動くのに、アプリでは 0 件。** 権限は許可済み。期間を広げても `No entries were created in this range.`。

原因は初期化の順番でした。

```swift
// これをやると詰む
let store = EKEventStore()          // 起動時に作る
try await requestAccess()           // ここで初めて許可される
// → この store は、以降ずっと「カレンダーが 0 個」と答え続ける
```

`EKEventStore` は**作られた時点の認可状態を握ります**。許可される前に作られたインスタンスは、あとから許可が下りても中身が見えないままです。コマンドラインが動いていたのは、そちらは既に許可済みの状態で store を作っていたからでした。

対策は 2 つです。**store は初回利用時に作る**（＝認可より後になる）ことと、**読む前に毎回 `reset()`** すること。

```swift
actor CalendarReader {

    /// Made on first use rather than at startup, so that it is never
    /// made before the user has granted access.
    private var source: EventKitSource?

    func log(createdIn range: DayRange, timeZone: TimeZone) -> Reading {
        let source = self.source ?? EventKitSource()
        self.source = source

        source.refresh()      // store.reset()
        return source.log(createdIn: range, planner: .standard, timeZone: timeZone)
    }
}
```

`reset()` は副産物として、**1 分前に Calendar.app で追加した予定がリロードで出る**ようにもなります。付けておいて損はありません。

### 切り分けの目印

同じ症状に当たったときのために書いておくと、**「カレンダーが 0 個」と「カレンダーは N 個あるが該当なし」を区別できるようにする**のが一番早い切り分けでした。

前者なら権限か store の問題、後者なら本当に期間内に何もないだけです。空表示のメッセージを分けてから、原因まで 1 分でした。

## 詰まりどころ 3: `creationDate` は read-only。だからテストが書けない

`EKCalendarItem.creationDate` は**読み取り専用**です。

つまり、**「先週の火曜日に作成された予定」をテスト用に作ることができません。** ローカルにカレンダーを作って予定を流し込んでも、作成日時は全部「今」になります。公開カレンダーを複製しても同じです。

この一点で設計が決まりました。**判断のロジックは EventKit のない場所に置くしかありません。**

| ディレクトリ | 中身 |
| --- | --- |
| `Sources/CalEntryCore` | すべての判断。純粋な Foundation のみ |
| `Sources/CalEntryKit` | EventKit のアダプタ。判断は一切しない |
| `Sources/EntryLogApp` | SwiftUI のウィンドウ |
| `Sources/EntryLogCLI` | コマンドライン |
| `Sources/CoreTests` | テスト（実行可能ファイルとして） |

`CalEntryKit` がやるのは `EKEvent` → `CalendarEntry` の変換だけです。期間の判定も役割の判定も表示の整形も、すべて `CalEntryCore` にあります。合成データを流し込めるので、156 個のチェックが Xcode なしで走ります。

**制約から出発した分割ですが、結果としてこれが正解でした。** 「EventKit を import している行を数える」というだけで、テストできない範囲がひと目で分かります。

## 詰まりどころ 4: EventKit は「誰が作ったか」を記録していない

Google Calendar API には `creator.self` があります。作成者が自分かどうかを、API がそのまま教えてくれます。

**EventKit にはありません。** あるのは `organizer` だけで、しかも**招待者のいる予定にしか存在しません**。ひとりで書いた予定の `organizer` は `nil` です。

そこで、こう判定しています。

| 状態 | 役割 |
| --- | --- |
| organizer が自分 | `created` |
| organizer が他人で、自分が参加者リストにいる | `invited` |
| organizer が他人で、自分はリストにいない | どちらでもない |
| organizer が nil、書き込み可能なカレンダー | `created` |
| organizer が nil、読み取り専用のカレンダー | どちらでもない |

```swift
public var role: EntryRole? {
    if let organizer {
        if organizer.isCurrentUser { return .created }
        if attendees.contains(where: \.isCurrentUser) { return .invited }
        return nil
    }
    return calendarIsWritable ? .created : nil
}
```

上 2 つはこの順で見るので、**自分が主催して自分も出る予定は 1 回だけ**数えられます。

**穴も書いておきます。** 書き込み権限付きで共有されたカレンダーに、他人が招待者なしで書いた予定は、自分が書いたものと区別できません。EventKit がその予定の書き手を保存していないからです。原理的に無理なので、諦めています。

### 読み取り専用カレンダーは検索しない

祝日カレンダーや誕生日カレンダーは検索対象から外しています。

**読み取り専用のカレンダーにある予定は、自分が書いたものでも自分への招待でもありえません。** 招待は「承諾できる場所」に届く必要があるからです。除外しても失うものはなく、そのぶん速くなります。

### 招待が 0 件のときにフィルタを出さない

私はカレンダーを共有したことがないので、`invited` は常に 0 件です。

最初はバグを疑いましたが、正しい 0 でした。ここでの判断は「役割判定を消す」ではなく「**表示しない**」です。

**役割の名前もチェックボックスも、2 種類以上が実際に存在するときだけ出します。** 常に空のフィルタは死んだコントロールですし、この形なら共有カレンダーが増えた日にひとりでに現れます。設定項目を増やさずに済みました。

## 詰まりどころ 5: Calendar.app の該当イベントに飛ばす

一覧の行をダブルクリックしたら、Calendar.app のその予定を開きたい。ここに一番時間を溶かしました。

### 効かなかったもの

**`ical://ekevent/<id>`** — Calendar.app は開きますが、**id は無視されて表示は動きません**。Spotlight が検索結果から予定を開くときに使う `?method=show&options=more` を付けても同じでした。

**AppleScript の `whose uid`** — 動きはしますが、**全カレンダーを走査するので 2 分待っても終わりません**。

```applescript
-- これは走査。数千件で実用にならない
set e to first event whose uid = "..."
```

### 効いたもの

**識別子で直接引く形なら、同じ AppleScript が 0.27 秒で返ります。**

```applescript
tell application "Calendar"
    activate
    switch view to month view
    show (event id "A1B2C3D4-..." of calendar "Work")
end tell
```

`whose` を書くか `event id ... of calendar ...` と書くかで、**2 桁以上違います**。前者は走査、後者は直接参照です。

`id` に渡すのは `EKEvent.calendarItemExternalIdentifier`（＝iCalendar の UID）、`calendar` にはカレンダー名を渡します。`show` は**月表示を保ったまま**該当イベントを選択してくれます。

### 繰り返し予定だけは無理

**繰り返し予定の全回が 1 つの UID を共有しています**（詰まりどころ 1 で味方だったのと同じ性質です）。そして AppleScript から「何回目」を指定する手段がありません。

結果、7 月 31 日の予定を `show` すると **4 月 3 日の同じ予定がハイライトされます**。手元では 4 か月ぶん 133 件のうち 86 件が繰り返し予定でした。

なので繰り返し予定は `show` せず、月に飛ばすだけにしています。**ハイライトなしで正しい月のほうが、ハイライト付きで間違った月よりましだ**という判断です。

### 日表示ではなく月表示にした理由

もう一つ。**`view calendar at` は日付だけを見て、時刻を無視します。**

```applescript
view calendar at d   -- d の時刻部分は捨てられる
```

日表示でこれをやると、16 時の予定をクリックしたのに**朝 8 時あたりが堂々と表示されます**。ユーザーから見れば別の予定を開いたようにしか見えません。

月表示には間違える時刻がないので、月表示にしました。

## 詰まりどころ 6: カレンダーを 0 個指定すると、全部返ってくる

カレンダーごとのチェックボックスを付けたときの話です。

仕事のカレンダーが入っている端末では、個人の予定を洗い出したいときに仕事の予定が混ざります。そこで一覧に含めるカレンダーを選べるようにしました。**「All」と「None」も付けました。**

そして **None を押すと、全部表示されました。**

```swift
// これが全カレンダーを返す
store.predicateForEvents(withStart: a, end: b, calendars: [])
```

`calendars:` は `[EKCalendar]?` で、**`nil` が「制限なし」**です。ところが**空配列も「制限なし」として扱われます**。「0 個のカレンダーを検索する」つもりの指定が、「全部検索する」になります。

`nil` と `[]` を区別しないぶん、意図と正反対の結果が黙って返るので厄介です。配列を組み立てる側で塞ぎました。

```swift
// EventKit reads an empty array as no restriction and returns
// everything, which is the opposite of what asking for no calendars
// means.
if let calendars, calendars.isEmpty { return [] }
```

## 計測してから決める: 読む前に絞るか、読んでから絞るか

このチェックボックスには設計の選択肢が 2 つありました。

1. **チェックの外れたカレンダーは読まない**（述語から外す）
2. **全部読んでおいて、表示のときに絞る**

1 のほうが筋が良さそうに見えます。読まなくていいものを読まないのだから速い。最初は 1 で実装しました。

ただしこの方式には代償があります。**チェックを 1 つ触るたびに再読み込みが走ります。** 使ってみると、これが想像以上に体験を損ねました。

そこで測りました。手元のカレンダー 8 個、前後 10 年・6 クエリでの実測です。

| 読む対象 | 時間 | 件数 |
| --- | --- | --- |
| 全 8 個 | **0.19 秒** | 2085 |
| 7 個（1 個除外） | **0.15 秒** | 2053 |
| 1 個だけ | 0.02 秒 | 68 |

**1 個外して節約できるのは 40 ミリ秒でした。** 体感できません。一方で、失っているのは「チェックが即座に反映される」という体験全体です。

2 に変えました。**速いほうを選んだつもりが、測ってみたら速さの差が存在しなかった**という話です。数千件のローカルストアを舐めるコストを、私は過大に見積もっていました。

（なお「読まない」ことにプライバシー上の意味があるケースもあります。ここは同一マシン上の自分のカレンダーなので、その理由は立ちません。）

## SwiftUI 側で踏んだもの

EventKit と関係ない、UI 側の落とし穴も 3 つありました。

### `.stepperField` の矢印は「選択中の要素」を動かす

日付フィールドの横の上下矢印を押したら、**年が変わりました**。

macOS の `DatePicker` の既定スタイル（`.stepperField`）は、**フィールド内で選択されている要素**を増減します。何もクリックしていない状態では先頭の要素が選ばれていて、この環境ではそれが年でした。最初の一押しで 12 か月動きます。

「日付を 1 日ずらす」が主な操作なので、フィールドを `.field`（矢印なし）にして、`Stepper` を自分で置きました。

```swift
DatePicker("", selection: selection, displayedComponents: .date)
    .labelsHidden()
    .datePickerStyle(.field)

Stepper("",
        onIncrement: { model.nudge(edge, byDays: 1) },
        onDecrement: { model.nudge(edge, byDays: -1) })
    .labelsHidden()
```

ついでに**フィールドの並び順もロケールで固定**しました。既定では Mac のリージョン設定に従うので、`8/25/2026` のように月が先に来ます。この 2 つのフィールドだけ `ja_JP` を与えて `2026/08/25` に揃えました。

### 選択可能なテキストの中身と高さを、同時に差し替えると落ちる

メモは 1 行に切り詰めて表示し、**マウスオーバーで全文に開く**ようにしました。

これが落ちました。毎回ではなく、4〜5 回ホバーすると落ちます。しかも**改行を含む長いメモのときだけ**です。

原因は組み合わせでした。その行には `.textSelection(.enabled)` が付いていて、**選択可能なテキストは専用のテキストビューで描かれます**。その下から**文字列と行の高さを同時に**抜き替えていました。改行があると高さの変化が大きいぶん、確実に踏みます。

3 つ直して収まりました。

- **メモの行だけ選択不可に**する（他の行は選択できるまま）
- **改行を潰して 1 段落にする**。20 行のメモがそのまま 20 行に開くと、下の行が画面外へ飛ぶ
- 開いても**最大 8 行**で止める

### 実データのスクリーンショットは公開できない

記事に貼る画像を撮ろうとして気づきました。**動いているところの画面には、人名も、住所も、メモの中身も、全部写ります。**

カレンダーのメモは、玄関の暗証番号が書かれている可能性が、予定のリマインダーと同じくらいあります。**画面に出すということはスクリーンショットに写るということ**です。モザイクで消すには消す箇所が多すぎました。

なので**デモモード**を作りました。この記事の冒頭の画像がそれです。

```swift
/// `--demo` fills the window with an invented calendar and never
/// touches EventKit.
static let isDemo = CommandLine.arguments.contains("--demo")
```

架空の予定を持っているだけで、EventKit を一度も開きません。メニューから切り替えられて、有効な間はフッタに `demo data` のバッジが出ます。**タイトルバーに書かなかったのは、起動時に決まるタイトルは切り替えた瞬間に嘘になるからです。**

副産物として、**権限を与えていない Mac でも画面を確認できる**ようになりました。データが絡む道具を作るなら、最初から入れておいていい仕組みだと思います。

## Xcode は使っていない

この Mac には Xcode が入っていますが、ライセンス同意が済んでおらず、同意には `sudo` が要ります。使えるのは Command Line Tools だけ、という状態でした。

結論だけ書くと、**SwiftUI の Mac アプリは Command Line Tools だけで作れます。** `.app` は手で組めますし、テストは XCTest が入っていないので 100 行のハーネスを自分で書きました。この記事の 156 checks はそれです。

ただし **ad-hoc 署名だけは省略できません**。識別子が変わるとカレンダーの許可がビルドのたびに消えます。

この辺りは EventKit と関係がないので、別記事にしました。

[XcodeなしでSwiftUIのMacアプリを作る — .appは手で組めるが、ad-hoc署名だけは省略できない](https://zenn.dev/takagit/articles/swiftui-app-without-xcode)

## まとめ

同じ道具を 2 つのプラットフォームで作って、詰まりどころが一つも重ならなかったのが面白いところでした。

- **EventKit も作成日時では検索できない。** 開催日時で広く取って後から絞る
- **述語 1 本は 4 年まで。ただし本数に制限はない。** 「4 年しか検索できない」と読むと詰むが、読み替えればループ
- **認可前に作った `EKEventStore` は、認可後も空のまま。** 初回利用時に作り、読む前に `reset()`
- **`creationDate` は read-only。** テスト用の作成日時を作れないので、判断は EventKit のない層に置くしかない
- **EventKit は作成者を記録しない。** `organizer` は招待者がいる予定にしかない
- **Calendar.app へのジャンプは `event id ... of calendar ...`。** `whose uid` は走査で 2 分、直接参照なら 0.27 秒
- **繰り返し予定は全回が同じ UID。** 重複排除では味方、特定の回を開くときは敵
- **`calendars: []` は「0 個」ではなく「全部」。** `nil` と同じ扱いになる
- **最適化は測ってから。** 読む前に絞る設計は、測ったら 40 ミリ秒しか稼いでいなかった
- **選択可能なテキストの中身と高さを同時に変えると落ちる**
- **実データのスクリーンショットは公開できない。** デモモードは後付けより最初から

---

Google カレンダー版と、その過程で踏んだ落とし穴も別記事にまとめてあります。

- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)
- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない](https://zenn.dev/takagit/articles/css-liquid-glass-backdrop-filter)
