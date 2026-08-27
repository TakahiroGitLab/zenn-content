---
title: "XcodeなしでSwiftUIのMacアプリを作る — .appは手で組めるが、ad-hoc署名だけは省略できない"
emoji: "🔨"
type: "tech"
topics: ["swift", "swiftui", "macos", "swiftpm", "xcode"]
published: true
---

Mac に Xcode は入っているのに、**ライセンス同意が済んでいませんでした**。

```
You have not agreed to the Xcode license agreements.
```

同意には `sudo` が要ります。管理者権限を通せない、あるいは通したくない状況で、使えるのは **Command Line Tools だけ**でした。

このとき素直に諦めかけたのですが、結論から言うと **SwiftUI の Mac アプリは Command Line Tools だけで作れます。** 実際、EventKit を使う実用アプリを 1 本この方法で作りました。

**ソース**: https://github.com/TakahiroGitLab/apple-cal-entry-log （MIT）

ただし「作れる」と「何も困らない」は別です。**省略できないもの**と、**代わりを自分で書く必要があるもの**があります。

## できること・できないこと

| | Command Line Tools だけで |
| --- | --- |
| SwiftUI のコンパイル | **できる**（`MenuBarExtra` も含めて） |
| `.app` バンドルの作成 | **できる**（手で組む） |
| コード署名 | **できる**（ad-hoc） |
| XCTest / swift-testing | **できない**（同梱されていない） |
| Interface Builder、プレビュー、Instruments | できない |

つまり、**GUI の支援が一切ないだけで、成果物は普通の `.app` です。**

## まず 10 秒で確かめる

作り始める前に、SwiftUI が本当に通るかだけ確認しておくと安心です。

```bash
echo 'import SwiftUI
@main struct A: App { var body: some Scene { WindowGroup { Text("hi") } } }' \
  > /tmp/a.swift && swiftc -typecheck /tmp/a.swift && echo OK
```

`OK` が出れば SDK は揃っています。私はここで**「Xcode が要る」という自分の思い込みが間違っていた**ことを知りました。

## 1. ビルドは SwiftPM に任せる

`Package.swift` で実行可能ファイルとして宣言するだけです。特別な設定は要りません。

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "AppleCalEntryLog",
    platforms: [.macOS(.v14)],
    products: [
        .executable(name: "EntryLog", targets: ["EntryLogApp"]),
    ],
    targets: [
        .executableTarget(name: "EntryLogApp", dependencies: [...]),
    ]
)
```

```bash
swift build -c release --product EntryLog
```

`.build/release/EntryLog` ができます。**これは単体でも起動しますが、この状態ではまだ「アプリ」ではありません。**

## 2. `.app` はディレクトリと 2 ファイル

バンドルの正体は決まった構造のディレクトリです。手で組めます。

```
EntryLog.app/
└── Contents/
    ├── Info.plist
    ├── MacOS/
    │   └── EntryLog      ← さっきのバイナリ
    └── Resources/
```

```bash
mkdir -p "${CONTENTS}/MacOS" "${CONTENTS}/Resources"
cp ".build/release/EntryLog" "${CONTENTS}/MacOS/EntryLog"
```

### Info.plist が「権限ダイアログの名前」を決める

ここがバンドルを作る一番の理由です。

**バンドルのないコマンドラインツールが権限を要求すると、要求元は「ターミナル」になります。** 許可ダイアログにもシステム設定のプライバシー一覧にも Terminal と出ます。自分のツールの名前で許可を求めたいなら、`.app` にして `Info.plist` を持たせる必要があります。

```xml
<key>CFBundleIdentifier</key>     <string>com.example.entrylog</string>
<key>CFBundleExecutable</key>     <string>EntryLog</string>
<key>CFBundlePackageType</key>    <string>APPL</string>
<key>NSPrincipalClass</key>       <string>NSApplication</string>
<key>LSMinimumSystemVersion</key> <string>14.0</string>

<key>NSCalendarsFullAccessUsageDescription</key>
<string>Entry Log lists the calendar entries you wrote down...</string>
```

`NS...UsageDescription` の文言はそのままダイアログに出ます。**何のために、何をしないかを書くと親切です。** 私は「読むだけで、何も変更しないし、この Mac から出ない」と書きました。

## 3. ad-hoc 署名は省略できない

自分の Mac から出ないツールに署名は不要な気がします。**必要です。**

```bash
codesign --force --sign - --identifier "com.example.entrylog" "${APP}"
```

`--sign -` が ad-hoc 署名で、証明書も Apple Developer Program も要りません。

**これを飛ばすと、ビルドのたびにカレンダーの許可が消えます。**

macOS は許可を「アプリの識別子」に紐づけて覚えます。署名がないと、システムから見て**リビルドのたびに別のアプリ**になります。開発中は 1 日に何十回もビルドするので、そのたびに許可ダイアログが出ます。identifier を固定して ad-hoc 署名しておけば、許可は引き継がれます。

**気づきにくい形で時間を溶かすので、最初から入れてください。**

## 4. XCTest も swift-testing も入っていない

テストを書こうとして止まります。

```
error: no such module 'Testing'
error: no such module 'XCTest'
```

どちらも Xcode 側にあり、Command Line Tools には同梱されていません。`DEVELOPER_DIR` で Xcode を指す手もありますが、**それができるならそもそもこの記事の状況にいません**（冒頭のライセンス同意に戻ります）。

### 100 行のハーネスを書く

テストターゲットではなく、**実行可能ファイル**にします。

```swift
// Package.swift
.executableTarget(name: "CoreTests", dependencies: ["CalEntryCore"]),
```

ハーネスに要るのは、失敗を貯めることと、`file:line` を出すことと、終了コードだけです。

```swift
final class Harness {

    private var checks = 0
    private var failures: [String] = []

    func equal<T: Equatable>(
        _ actual: T, _ expected: T, _ what: String,
        file: StaticString = #fileID, line: UInt = #line
    ) {
        checks += 1
        if actual != expected {
            record("\(what): expected \(expected), got \(actual)",
                   file: file, line: line)
        }
    }

    /// Prints the tally and returns the process exit code.
    func report() -> Int32 {
        guard failures.isEmpty else {
            print("\(failures.count) of \(checks) checks failed\n")
            for failure in failures { print("  - \(failure)") }
            return 1
        }
        print("\(checks) checks passed")
        return 0
    }
}
```

`#fileID` と `#line` を**デフォルト引数**にしておくのがポイントです。呼び出し側は何も書かなくても、失敗した行が出ます。

```swift
// main.swift
let harness = Harness()
roleTests(harness)
dayRangeTests(harness)
exit(harness.report())
```

```
$ swift run core-tests

Role
  ok    The organiser being the user means created
  ok    Someone else's meeting with the user on the list means invited
...

138 checks passed
```

終了コードを返しているので、CI でもそのまま使えます。

### 書いたら、必ず 1 行壊してみる

**自作ハーネスの最大の危険は、何もテストしていないのに全部 pass することです。**

`expect` の中身を書き間違えていても、`suite` を呼び忘れていても、出力は緑のままです。XCTest なら気づける類のミスが、自作だと静かに通ります。

書き終えたら、**わざとロジックを 1 行壊して、赤が出ることを確認してください。** これをやるまでは、そのハーネスは動作未確認です。

## 5. `/Applications` に入れる

最後に、地味に効いた話を一つ。

`build/` の中で動作確認していると、**`/Applications` に置いたコピーが古いまま**になります。昨日のアプリを起動しながら今日のコードを直す、という分かりにくい午後が発生します。

インストールも 1 コマンドにしました。

```bash
"${ROOT}/Scripts/make-app.sh" "$@"

if pgrep -x EntryLog > /dev/null; then
    echo "quitting the running copy"
    pkill -x EntryLog
fi

# Replaced rather than copied over: a file left behind by an older
# build would otherwise survive into the new one.
rm -rf "${INSTALLED}"
cp -R "${BUILT}" "${INSTALLED}"

codesign --verify --strict "${INSTALLED}"
```

**`cp -R` で上書きせず、`rm -rf` してから置く**のがポイントです。上書きだと、前のビルドにしか存在しないファイルが新しいバンドルの中に生き残ります。署名の検証まで入れておくと、壊れたものを起動して悩む時間がなくなります。

## まとめ

- **SwiftUI は Command Line Tools だけでコンパイルできる。** 先に `swiftc -typecheck` で 10 秒確認する
- **`.app` はディレクトリと `Info.plist` と実行ファイル。** 手で組める
- **バンドルがないと、権限要求は「ターミナル」名義になる。** 自分の名前で求めたいなら `.app` にする
- **ad-hoc 署名（`codesign --sign -`）は省略できない。** 識別子が変わると許可が毎回消える
- **XCTest も swift-testing も入っていない。** 100 行のハーネスと `.executableTarget` で足りる
- **自作ハーネスは、1 行壊して赤を見るまで動作未確認**
- インストールは `rm -rf` してから `cp -R`

Xcode が使えないことは、思っていたより小さな制約でした。**むしろ、ビルドが何をしているかが全部シェルスクリプトの上に見えている状態は、把握しやすくて悪くありません。**

---

この方法で作ったのは、Apple Calendar の予定を「**いつ登録したか**」で一覧する Mac アプリです。EventKit 側で踏んだ落とし穴 — 作成日時では検索できない、認可前に作った `EKEventStore` は空のまま、`creationDate` が read-only でテストが書けない — は別記事にまとめました。

[Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った](https://zenn.dev/takagit/articles/apple-calendar-entry-log-eventkit)

コードは MIT で公開しています。

https://github.com/TakahiroGitLab/apple-cal-entry-log
