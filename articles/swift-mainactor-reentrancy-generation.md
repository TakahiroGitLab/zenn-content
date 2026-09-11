---
title: "@MainActorなのに古い結果が新しい結果を上書きする — actorはawaitで割り込まれる"
emoji: "🔀"
type: "tech"
topics: ["swift", "swiftui", "concurrency", "macos", "eventkit"]
published: true
---

カレンダーの予定を「いつ登録したか」で一覧する Mac アプリを作っています。日付の範囲を選ぶと、その範囲を読み直して表示します。

たまに、**選んだのと違う範囲が表示されます。** 今日を選んだのに、さっきまで見ていた昨日が出ている。もう一度押せば直ります。

モデルは `@MainActor` が付いた `@Observable` クラスで、状態を書き換えるのは全部その中です。**データ競合は起きようがない**はずでした。

そして実際、データ競合は起きていませんでした。**起きていたのは順序の問題です。**

## 症状

- 日付を変えると、たまに前の範囲の結果が表示される
- 必ず起きるわけではない。読み込みが遅いときだけ
- リロードボタンを押すと直る
- クラッシュしない。警告も出ない。Thread Sanitizer も何も言わない

## 読み込みを始めるものが3つあった

このアプリで読み込みを要求する経路は3つあります。

```swift
.task(id: model.fetchKey) {
    // 日付ピッカーは操作中に何度も変更を報告するので、少し待つ
    try? await Task.sleep(for: .milliseconds(250))

    guard !Task.isCancelled else { return }

    await model.reload()
}
.task {
    // Calendar 側で追加された予定や、iPhone から届いた変更
    for await _ in NotificationCenter.default.notifications(
        named: EventKitSource.storeChanged
    ) {
        model.reloadSoon()
    }
}
```

これに加えて、リロードボタンです。

```swift
Button {
    Task { await model.reload() }
} label: {
    Image(systemName: "arrow.clockwise")
}
```

**範囲の変更、ストアからの通知、リロードボタン。** そして直列化していたのは、このうち通知だけでした。

```swift
private var pending: Task<Void, Never>?

func reloadSoon() {
    pending?.cancel()

    pending = Task { [weak self] in
        try? await Task.sleep(for: .milliseconds(800))

        guard !Task.isCancelled else { return }

        await self?.reload()
    }
}
```

同期が走ると変更通知はまとめて何発も飛んでくるので、その連打を1回にまとめる仕組みです。**通知どうしは直列化されますが、通知とボタンは別物です。** 3つのうち2つが同時に飛ぶことを、何も止めていませんでした。

## `@MainActor` が保証するのは「安全」であって「順序」ではない

ここが本題です。

`@MainActor` が付いているので、このクラスのメソッドはすべてメインアクター上で動きます。2つの `reload()` が**同時に**走ることはありません。それは正しい。

しかし `reload()` は `async` で、途中で3回 `await` します。**`await` のたびに、そのアクターは他の仕事を実行できます。**

Swift のアクターを定義した [SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md) に、そのまま書かれています。

> When an actor-isolated function suspends, reentrancy allows other work to execute on the actor before the original actor-isolated function resumes, which we refer to as _interleaving_.

そして、何が保証されて何が保証されないかも明記されています。

> Interleaving executions still respect the actor's "single-threaded illusion", i.e., no two functions will ever execute _concurrently_ on any given actor. However they may _interleave_ at suspension points.

**同時には走らない。しかし中断点で交互に進む。** アクターは再入可能（reentrant）です。

同じ提案は、その帰結までこう言っています。

> every suspension point must be carefully inspected if the code _after_ it depends on some invariants that could have changed before it suspended

まさにこれでした。`reload()` は `await` の後で `loaded` や `state` に書き込みます。その `await` の間に別の `reload()` が始まって先に終わっていれば、**遅いほうが後から、古い答えを上書きします。**

「メインアクターにいるから安全」は正しく、そして**問題の半分しか見ていませんでした。**

## 30行で再現する

アプリを持ち出さなくても、これだけで起きます。`swift repro.swift` で走ります。

```swift
import Foundation

@MainActor
final class Model {
    var value = "(まだ何も読んでいない)"

    func read(_ label: String, delay: Duration) async {
        try? await Task.sleep(for: delay)
        print("  [\(label)] 再開。Task.isCancelled = \(Task.isCancelled)")
        value = label
    }
}

@MainActor
func run() async {
    let m = Model()

    // 遅い読み込みを先に、速い読み込みを後から始める
    async let slow: Void = m.read("遅い読み込み・古い範囲", delay: .milliseconds(300))
    async let fast: Void = m.read("速い読み込み・新しい範囲", delay: .milliseconds(50))

    _ = await (slow, fast)

    print("最終的な表示: \(m.value)")
}

await run()
```

出力です。

```
  [速い読み込み・新しい範囲] 再開。Task.isCancelled = false
  [遅い読み込み・古い範囲] 再開。Task.isCancelled = false
最終的な表示: 遅い読み込み・古い範囲
```

`@MainActor` が付いていて、`Model` の外から触っているものは何もなくて、それでも**後から始めた新しい読み込みの結果が、先に始めた古い読み込みに上書きされます。**

そして**両方の `Task.isCancelled` が `false`** であることも、ここで見えています。次の節の話です。

## キャンセルでは直らない

最初に思いつくのはキャンセルです。新しい読み込みが始まったら古いほうを止めればいい。

実際 `.task(id:)` はそれをやっています。[ドキュメント](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:))にも "it also cancels and recreates the task when a specified value changes" とあり、`id` が変われば前のタスクはキャンセルされます。**範囲の変更どうしなら**古いほうは止まります。

問題は、3つの経路が**互いにキャンセル関係を持っていない**ことです。

- `.task(id:)` がキャンセルするのは、自分の前任だけ
- ボタンの `Task { }` は非構造化タスクで、誰の子でもない
- `pending` がキャンセルするのは、前の `pending` だけ

ボタンを押した読み込みは、範囲変更の読み込みを止めません。**その逆も同じです。** だから両方の `Task.isCancelled` が `false` のまま、2つとも最後まで走りきります。先ほどの再現コードの出力が、まさにそれでした。

「非構造化タスクは誰の子でもない」も、そのまま確かめられます。

```swift
let parent = Task {
    Task {                                  // 中で作った非構造化タスク
        try? await Task.sleep(for: .milliseconds(200))
        print("  中で作った Task { }: Task.isCancelled = \(Task.isCancelled)")
    }

    try? await Task.sleep(for: .milliseconds(500))
    print("  親タスク: Task.isCancelled = \(Task.isCancelled)")
}

try? await Task.sleep(for: .milliseconds(50))
parent.cancel()
```

```
  親タスク: Task.isCancelled = true
  中で作った Task { }: Task.isCancelled = false
```

**親をキャンセルしても、中で作った `Task { }` には届きません。**

`Task.isCancelled` のチェックは書いてありました。役に立っていなかっただけです。

## 世代番号をつける

やったのは単純なことです。**読み込みごとに番号を振り、`await` のたびに「自分がまだ最新か」を確認する。**

```swift
/// Which read is the newest one. Taken on the way in, checked
/// again after every suspension.
private var generation = 0
```

```swift
func reload() async {

    generation += 1
    let mine = generation

    // ... 範囲の計算、state = .loading ...

    do {
        try await reader.requestAccess()
    } catch {
        if mine == generation { state = .needsAccess("\(error)") }
        return
    }

    let offered = await reader.calendars()

    guard mine == generation, !Task.isCancelled else { return }

    calendars = offered

    let reading = await reader.log(createdIn: range, timeZone: timeZone)

    // A read the user has already moved on from -- the range
    // changed under it, or a sync landed -- must not overwrite
    // the one they are waiting for.
    guard mine == generation, !Task.isCancelled else { return }

    loaded = reading.entries
    searched = reading.plan.span
    queries = reading.plan.windows.count
    calendarsSearched = reading.calendarsSearched
    state = .ready
}
```

入口で `generation` を増やして、自分の番号を `mine` に控える。**新しい読み込みが始まれば `generation` だけが進むので、`mine == generation` が偽になった時点で自分は用済み**だと分かります。

先ほどの再現コードに世代番号を足すと、こうなります。

```
  [速い読み込み・新しい範囲] 最新なので反映 (mine=2)
  [遅い読み込み・古い範囲] 追い越されたので結果を破棄 (mine=1, generation=2)
最終的な表示: 速い読み込み・新しい範囲
```

遅いほうは再開して、自分の番号が古いことに気づいて、**何も書かずに戻ります。**

`generation` の読み書きはどちらもメインアクター上なので、ここにロックは要りません。**アクターが保証してくれるのは、まさにこの部分です。** 足りなかったのは、アクターが保証しない側だけでした。

`Task.isCancelled` も残してあります。世代番号と役割が違うからです — 世代は「追い越されたか」、キャンセルは「もう要らなくなったか」。`.task(id:)` の経路では後者も実際に起きます。

## 途中の結果を、最後まで持ち越さない

`calendars` への代入は、読み込みの**途中**にあります。最後にまとめて書いてはいません。

```swift
let offered = await reader.calendars()

guard mine == generation, !Task.isCancelled else { return }

calendars = offered      // ← ここで代入。まだ本体の読み込み前
```

すべてを最後まで持ち越して一気に反映するほうが「原子的」に見えますが、**そうするとカレンダーの絞り込みフィルタが、予定を全部読み終わるまで画面に出てきません。** 数秒かかる操作で、それは体感としてはっきり遅くなります。

世代のチェックさえ通っていれば、途中で反映しても古い読み込みの結果が混ざることはありません。**守りたいのは「古い答えが新しい答えを上書きしないこと」であって、「全部を同時に差し替えること」ではない**、という整理です。

## おまけ: 2つの読み込みが同じものを読んでいない場合

このアプリにはデモモードがあります。スクリーンショットを撮るための、架空のカレンダーです。

```swift
if isDemo {
    showDemo(in: range)
    return
}
```

デモに切り替えると `fetchKey` が変わって読み込みが走り、架空のデータが入ります。ところが**本物の読み込みが飛んでいる最中にデモへ切り替えると、後から着地した本物が架空のカレンダーを上書きしていました。**

これは「古い結果が新しい結果を上書きする」の一種ですが、**2つの読み込みが同じデータソースですらない**ぶん、症状としては派手です。デモを出したのに実物が出る。

世代番号を入れたら、これも直りました。**同じ原因だったからです。**

## まとめ

- **`@MainActor` はデータ競合を防ぐ。順序は守らない。** SE-0306 の言葉では、アクターは "single-threaded illusion" を保つが、中断点で "interleave" する
- `async` 関数の `await` は、**すべて「他の処理が割り込みうる場所」**。再開後のコードが、中断前の前提に依存していないか毎回確認する必要がある
- **キャンセルは万能ではない。** 互いにキャンセル関係を持たない複数の経路があると、`Task.isCancelled` は全部 `false` のまま走りきる。`.task(id:)` が止めるのは自分の前任だけで、非構造化の `Task { }` は誰の子でもない
- 対処は**世代番号**。入口で番号を取り、`await` のたびに自分が最新かを確認して、違えば結果を捨てる
- 世代番号そのものの読み書きにロックは要らない。**アクターが守ってくれるのは、まさにそこ**
- 途中の結果を途中で反映してよい。守りたいのは「古い答えが勝たないこと」であって、原子的な差し替えではない

読み込みを始める経路が2つ以上あるなら、`@MainActor` を付けた時点では**まだ何も解決していません。**

---

このアプリ自体についても書いています。

[Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った](https://zenn.dev/takagit/articles/apple-calendar-entry-log-eventkit)

[XcodeなしでSwiftUIのMacアプリを作る — .appは手で組めるが、ad-hoc署名だけは省略できない](https://zenn.dev/takagit/articles/swiftui-app-without-xcode)

検証環境: Swift 6.3.3（swiftlang-6.3.3.1.3）、macOS 26、arm64。
