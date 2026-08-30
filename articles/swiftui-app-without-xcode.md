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
| アプリアイコン | **できる**（`iconutil`） |
| XCTest / swift-testing | **できない**（同梱されていない） |
| Interface Builder、プレビュー、Instruments | できない |

つまり、**GUI の支援が一切ないだけで、成果物は普通の `.app` です。**

## まず 10 秒で確かめる

作り始める前に、SwiftUI が本当に通るかだけ確認しておくと安心です。

```bash
echo 'import SwiftUI
@main struct A: App { var body: some Scene { WindowGroup { Text("hi") } } }' \
  > /tmp/a.swift && swiftc -parse-as-library -typecheck /tmp/a.swift && echo OK
```

`OK` が出れば SDK は揃っています。私はここで**「Xcode が要る」という自分の思い込みが間違っていた**ことを知りました。

**`-parse-as-library` を落とすと、SDK の有無と関係なく落ちます。**

```
error: 'main' attribute cannot be used in a module that contains top-level code
```

`swiftc` に単独のファイルを渡すと、その中身はトップレベルコードとして扱われます。`@main` は「エントリポイントはこの型だ」と宣言するものなので、両立しません。**SwiftUI が使えないという意味ではありません。** ここで諦めると、この記事と正反対の結論に着地します。

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

飛ばすとどうなるか。**バンドルとして壊れます。**

```console
$ codesign --verify --strict Bare.app
Bare.app: code has no resources but signature indicates they must be present
```

Apple Silicon では、リンカが実行ファイルに ad-hoc 署名を自動で付けます。なので「まったくの無署名」にはなりません。ただし付くのはバイナリだけで、**バンドルには `Contents/_CodeSignature/` ができません。** 結果、署名はリソースがあると言っているのにリソースの封印が無い、という上の状態になります。identifier も、`Info.plist` の `CFBundleIdentifier` ではなく**実行ファイル名**のままです。

```console
$ codesign -dv Bare.app | grep Identifier
Identifier=Demo                    # ← バイナリ名。バンドルIDではない

$ codesign --force --sign - --identifier com.example.demoapp Demo.app
$ codesign -dv Demo.app | grep Identifier
Identifier=com.example.demoapp     # ← 意図した識別子になる
```

私の場合、これを入れてからカレンダーの許可ダイアログがビルドのたびに出る状態が収まりました。ただし**その機序を「identifier に紐づくから」と説明するのは正しくないようです。** ad-hoc 署名の designated requirement を見ると、識別子ではなく `cdhash` で書かれていて、`--identifier` を固定していてもコードを変えれば変わります。

```console
$ codesign -d -r- Demo.app
designated => cdhash H"8b98674d9132807854ecb8423c90fa87c875bfb2"
$ # ソースを1行変えて再ビルド・再署名
designated => cdhash H"4ba23ccd79452e2173fc7b83c5fd7062464f33f4"
```

確実に言えるのは、**署名しないとバンドルの検証が通らない**ことと、**識別子が意図したものにならない**ことです。そのうえで許可がどう保存されるかは、TCC のデータベースが Full Disk Access なしには読めないため、この記事では確かめられていません。

いずれにせよ 1 行で済み、飛ばすと気づきにくい形で時間を溶かすので、最初から入れてください。

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

156 checks passed
```

終了コードを返しているので、CI でもそのまま使えます。

### 書いたら、必ず 1 行壊してみる

**自作ハーネスの最大の危険は、何もテストしていないのに全部 pass することです。**

`expect` の中身を書き間違えていても、`suite` を呼び忘れていても、出力は緑のままです。XCTest なら気づける類のミスが、自作だと静かに通ります。

書き終えたら、**わざとロジックを 1 行壊して、赤が出ることを確認してください。** これをやるまでは、そのハーネスは動作未確認です。

## 5. アイコンも Xcode なしで付く

ここまでで動くアプリになりますが、Dock には**白い書類の絵**が並びます。

Xcode なら Asset Catalog に画像をドラッグして終わりです。それが使えなくても、**`.icns` は 2 コマンドで作れます。**

### 必要なのは PNG 10 枚と `iconutil`

`AppIcon.iconset` という名前のディレクトリに、**決まった名前の PNG** を入れます。名前が規約です。

そして、ここが本当に厄介なところです。**1 枚欠けても、綴りを間違えても、`iconutil` は失敗しません。** 何も言わずに成功し、そのサイズだけ入っていない `.icns` を作ります。

```console
$ rm AppIcon.iconset/icon_128x128.png     # 1枚消す
$ iconutil --convert icns --output missing.icns AppIcon.iconset
$ echo $?
0                                          # 何も言わない

$ iconutil --convert iconset --output back.iconset missing.icns
$ ls back.iconset
icon_128x128@2x.png  icon_16x16.png  ...    # 128x128.png だけ無い
```

`exit 0` で、標準出力にも標準エラーにも何も出ません。**ビルドは通り、アイコンも付き、特定のサイズでだけ見た目が崩れます。** 失敗してくれたほうがまだ親切でした。

| ファイル名 | ピクセル |
| --- | --- |
| `icon_16x16.png` / `icon_16x16@2x.png` | 16 / 32 |
| `icon_32x32.png` / `icon_32x32@2x.png` | 32 / 64 |
| `icon_128x128.png` / `icon_128x128@2x.png` | 128 / 256 |
| `icon_256x256.png` / `icon_256x256@2x.png` | 256 / 512 |
| `icon_512x512.png` / `icon_512x512@2x.png` | 512 / 1024 |

あとは変換して、`Info.plist` に名前を書くだけです。

```bash
iconutil --convert icns \
    --output "${CONTENTS}/Resources/AppIcon.icns" \
    "${ROOT}/build/AppIcon.iconset"
```

```xml
<key>CFBundleIconFile</key> <string>AppIcon</string>
<key>CFBundleIconName</key> <string>AppIcon</string>
```

`.icns` は `Contents/Resources/` に置きます。拡張子を除いた名前を `CFBundleIconFile` に書く、という古い作法です。

### PNG を 10 枚管理せず、コードで描く

ここからは好みの話ですが、**画像ファイルをリポジトリに置かないことにしました。** 代わりに 200 行ほどの Swift スクリプトが、ビルドのたびに 10 枚を描き出します。

```swift
// Scripts/make-icon.swift
let rect = CGRect(x: S * 0.055, y: S * 0.055,
                  width: S * 0.89, height: S * 0.89)
```

**すべての寸法をキャンバス `S` に対する比率で書くのがポイントです。** こうすると 16px でも 1024px でも同じコードが成立するので、サイズごとの書き出しはループで済みます。

```swift
for (points, scale) in wanted {
    let suffix = scale == 1 ? "" : "@2x"
    guard let png = icon(size: CGFloat(points * scale)) else { exit(1) }
    try png.write(to: directory
        .appendingPathComponent("icon_\(points)x\(points)\(suffix).png"))
}
```

利点は**色を変えたくなったときに分かります**。数値 1 つを直して再ビルドすれば 10 枚すべてが追従します。画像編集ソフトで 10 回書き出す作業が消えます。

### 小さいサイズは「同じ絵の縮小」にしてはいけない

これは実際に失敗して気づきました。

作ったアイコンは、カレンダーの上に虫眼鏡が乗っている絵です。512px では意図どおりでした。**32px で書き出したら、ただの団子になりました。**

当然で、32px の中で実際に絵に使えるのは 28px 程度です。そこにカレンダーのリング、罫線、8 個の日付、虫眼鏡を詰め込めば、**細部は細部にならず汚れになります。**

そこで、しきい値を設けて絵そのものを変えました。

```swift
/// 16 点は絵に使える範囲が 11px しかない。リングも罫線も
/// 8 個の日付も、そこでは 4 つの染みになる。染みは無いより悪い。
let detailed: (CGFloat) -> Bool = { $0 >= 96 }
```

96px 未満では、リングと罫線と 7 個の日付を**描きません**。残すのはページの輪郭、虫眼鏡、そしてその中の点 1 つだけ。線幅も 1.5 倍にします。**同じアイコンの縮小版ではなく、同じ主題の別の絵**です。

Apple 自身も昔から同じことをしています。Finder のアイコンを 16px と 512px で見比べると、描かれているものが違います。**コードで描いていると、この作り分けが `if` 一つで済みます。**

### `make-app.sh` に組み込む

```bash
swift "${ROOT}/Scripts/make-icon.swift" "${ROOT}/build/AppIcon.iconset"
iconutil --convert icns \
    --output "${CONTENTS}/Resources/AppIcon.icns" \
    "${ROOT}/build/AppIcon.iconset"
```

`swift` はスクリプトをそのまま実行できるので、コンパイル手順は要りません。ビルドが 2 秒ほど伸びますが、**アイコンだけ古い状態が発生しない**ことのほうが価値があります。

なお **Dock はアイコンをキャッシュします。** 差し替えたのに変わらないときは、`.app` を `touch` して `killall Dock` してください。アイコンが悪いのかキャッシュなのか分からずに悩む、というのは避けられます。

## 6. `/Applications` に入れる

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

- **SwiftUI は Command Line Tools だけでコンパイルできる。** 先に `swiftc -parse-as-library -typecheck` で 10 秒確認する（このフラグを落とすと `@main` が通らない）
- **`.app` はディレクトリと `Info.plist` と実行ファイル。** 手で組める
- **バンドルがないと、権限要求は「ターミナル」名義になる。** 自分の名前で求めたいなら `.app` にする
- **ad-hoc 署名（`codesign --sign -`）は省略できない。** 飛ばすとバンドルの `--verify --strict` が通らず、識別子も実行ファイル名のままになる
- **XCTest も swift-testing も入っていない。** 100 行のハーネスと `.executableTarget` で足りる
- **自作ハーネスは、1 行壊して赤を見るまで動作未確認**
- **アイコンは `.iconset` の PNG 10 枚 + `iconutil`。** 1 枚欠けても `iconutil` は黙って成功し、そのサイズだけ欠けた `.icns` ができる。コードで描けば色変更が 1 数値で済み、小サイズだけ絵を削るのも `if` 一つ
- インストールは `rm -rf` してから `cp -R`

Xcode が使えないことは、思っていたより小さな制約でした。**むしろ、ビルドが何をしているかが全部シェルスクリプトの上に見えている状態は、把握しやすくて悪くありません。**

---

この方法で作ったのは、Apple Calendar の予定を「**いつ登録したか**」で一覧する Mac アプリです。EventKit 側で踏んだ落とし穴 — 作成日時では検索できない、認可前に作った `EKEventStore` は空のまま、`creationDate` が read-only でテストが書けない — は別記事にまとめました。

[Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った](https://zenn.dev/takagit/articles/apple-calendar-entry-log-eventkit)

コードは MIT で公開しています。

https://github.com/TakahiroGitLab/apple-cal-entry-log

---

この記事のコマンドは、Command Line Tools だけの環境（`xcode-select -p` が `/Library/Developer/CommandLineTools`、Swift 6.3.2、macOS 26.5.2）で実際に実行して確かめました。SwiftUI の 10 秒チェック、`swift build`、`.app` の組み立て、ad-hoc 署名と `--verify --strict`、`import XCTest` / `import Testing` の失敗、`.iconset` から `.icns` への変換まで、出力は記事のとおりです。
