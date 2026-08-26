---
title: "Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った"
emoji: "🍎"
type: "tech"
topics: ["swift", "swiftui", "eventkit", "macos", "applescript"]
published: false
---

以前、Google カレンダーの予定を「**いつ登録したか**」で一覧するツールを Apps Script で作りました。

[Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)

同じものが Apple Calendar 側にも欲しくなったので、EventKit で作り直しました。SwiftUI の Mac アプリと、同じ内容を吐くコマンドラインの 2 つです。

**ソース**: https://github.com/TakahiroGitLab/apple-cal-entry-log （MIT）

移植というより作り直しでした。**やりたいことは同じなのに、詰まる場所が一つも重なりません。** この記事はその詰まりどころの記録です。

## 何を作ったか

期間を指定すると、その期間に**登録された**予定が作成日時順に並びます。自分が書いたものか、招待されたものかも出ます。ここまでは Google 版と同じです。

違うのは、Web アプリではなくローカルのアプリだという点です。

- カレンダーは iCloud にありますが、読むのは Mac 上の EventKit 経由
- ネットワークに何も出ません
- 行をダブルクリックすると Calendar.app の該当イベントが開きます

Xcode は使っていません。この点は最後に書きます。

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

前後 10 年で **6 クエリ**。ローカルの数千件を舐めるだけなので **1 秒程度**です。「4 年」という数字を見た瞬間に諦めなくてよかった、というのがこのプロジェクト最大の学びでした。

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

`CalEntryKit` がやるのは `EKEvent` → `CalendarEntry` の変換だけです。期間の判定も役割の判定も表示の整形も、すべて `CalEntryCore` にあります。合成データを流し込めるので、138 個のチェックが Xcode なしで走ります。

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
    show (event id "690F7F9F-..." of calendar "Regular work")
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

## 詰まりどころ 6: Xcode なしで SwiftUI アプリを作る

この Mac には Xcode が入っていますが、**ライセンス同意が済んでおらず、同意には `sudo` が要ります**。使えるのは Command Line Tools だけ、という状態でした。

結論から言うと、**SwiftUI アプリは Command Line Tools だけで作れます。**

### ビルド

SwiftPM が CLT の SDK に対してビルドします。`MenuBarExtra` を含め、型検査は通ります。

```swift
// Package.swift
platforms: [.macOS(.v14)],
products: [
    .executable(name: "EntryLog", targets: ["EntryLogApp"]),
]
```

### `.app` は 4 ファイルとディレクトリ構造

バンドルは手で組めます。**重要なのは Info.plist で、これが権限ダイアログに出る名前を決めます。**

```bash
mkdir -p "${CONTENTS}/MacOS" "${CONTENTS}/Resources"
cp ".build/release/EntryLog" "${CONTENTS}/MacOS/EntryLog"
cat > "${CONTENTS}/Info.plist" <<PLIST
...
  <key>NSCalendarsFullAccessUsageDescription</key>
  <string>...</string>
  <key>NSAppleEventsUsageDescription</key>
  <string>...</string>
...
PLIST
```

**コマンドラインにはバンドルがないので、権限要求は「ターミナル」名義になります。** システム設定のプライバシーで探すときも Terminal を探すことになるので、覚えておかないと混乱します。

### ad-hoc 署名は省略できない

```bash
codesign --force --sign - --identifier "com.takahiromori.entrylog" "${APP}"
```

自分の Mac から出ないツールに署名は要らない気がしますが、**要ります**。署名がないと**ビルドのたびに別アプリ扱いになり、カレンダーの許可が毎回消えます**。identifier を固定して ad-hoc 署名しておけば、許可は引き継がれます。

### XCTest も swift-testing も CLT には入っていない

```
error: no such module 'Testing'
error: no such module 'XCTest'
```

どちらも Command Line Tools には同梱されていません。Xcode 側を `DEVELOPER_DIR` で指そうとしましたが、ここで前述のライセンス同意に阻まれました。

**100 行ほどのハーネスを書いて、テストを `.executableTarget` にしました。**

```swift
// Package.swift
.executableTarget(name: "CoreTests", dependencies: ["CalEntryCore"]),
```

```
swift run core-tests
→ 138 checks passed
```

失敗時に `file:line` を出すところまで作れば、実用上これで足ります。依存ゼロで、どこでも走ります。**書いたあと、わざと 1 行壊して落ちることを確認してください。** 何もテストしていないハーネスは静かに全部 pass します。

## まとめ

同じ道具を 2 つのプラットフォームで作って、詰まりどころが一つも重ならなかったのが面白いところでした。

- **EventKit も作成日時では検索できない。** 開催日時で広く取って後から絞る
- **述語 1 本は 4 年まで。ただし本数に制限はない。** 「4 年しか検索できない」と読むと詰むが、読み替えればループ
- **認可前に作った `EKEventStore` は、認可後も空のまま。** 初回利用時に作り、読む前に `reset()`
- **`creationDate` は read-only。** テスト用の作成日時を作れないので、判断は EventKit のない層に置くしかない
- **EventKit は作成者を記録しない。** `organizer` は招待者がいる予定にしかない
- **Calendar.app へのジャンプは `event id ... of calendar ...`。** `whose uid` は走査で 2 分、直接参照なら 0.27 秒
- **繰り返し予定は全回が同じ UID。** 重複排除では味方、特定の回を開くときは敵
- **Xcode なしで SwiftUI アプリは作れる。** ただし ad-hoc 署名は必須で、テストは自前

一点、これを人に配るなら直すべきところも書いておきます。**いまはメモの冒頭 50 文字をそのまま一覧に出しています。** 自分用には便利ですが、カレンダーのメモは玄関の暗証番号が書かれている可能性が予定のリマインダーと同じくらいあります。**画面に出すということはスクリーンショットに写るということ**なので、公開版では「メモあり」の印だけ出して、クリックで開く形にするつもりです。

---

Google カレンダー版と、その過程で踏んだ落とし穴は別記事にまとめてあります。

- [Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った](https://zenn.dev/takagit/articles/gcal-entry-log-apps-script)
- [Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している](https://zenn.dev/takagit/articles/gas-webapp-viewport-addmetatag)
- [CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない](https://zenn.dev/takagit/articles/css-liquid-glass-backdrop-filter)
