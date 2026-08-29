---
title: "常駐させたclaude CLIがNot logged inになる — PATHを直しても直らない理由はKeychainにある"
emoji: "🔑"
type: "tech"
topics: ["macos", "launchd", "claudecode", "keychain", "launchagent"]
published: false
---

Mac mini に常駐させた自作の Web アプリに、「AI に聞く」ボタンがあります。中身は `claude -p` を subprocess で叩くだけの、ごく単純なものです。

ターミナルから手で実行すれば動きます。同じ Mac の、同じユーザーで。

ところが launchd に登録して常駐させた途端、ボタンを押すとこれが返ってきます。

```
Not logged in
```

ログインはしています。ターミナルで `claude -p "hello"` と打てば普通に答えます。plist には `UserName` も書きました。それでも常駐プロセスからだけ、ログインしていないことになります。

原因は 2 つあって、**1 つ目を直しても症状が変わらないので、2 つ目に気づくまで時間を溶かします。** 私は PATH を疑い続けました。

## 症状

- ターミナルから `claude -p` → 動く
- launchd 経由の常駐プロセスから `claude -p` → `Not logged in`
- plist の `UserName` は正しいユーザーになっている
- ユーザーを間違えたわけでも、ログインが切れたわけでもない

「デーモンだから環境が違うんだろう」までは誰でも思い当たります。問題は、**その"環境"が 2 種類ある**ことです。

## 詰まりどころ 1: launchd はログインシェルの PATH を継がない

まずこちらに当たります。`claude` は `~/.local/bin` あたりに入っていることが多く、そこに PATH を通しているのは `.zshrc` です。

launchd はログインシェルを経由しないので、`.zshrc` は読まれません。**`claude` コマンドがそもそも見つかりません。**

plist で明示すれば直ります。

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/Users/taka/.local/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
</dict>
```

**これは LaunchAgent でも LaunchDaemon でも同じく必要です。** どちらもログインシェルを通らないので、どちらでも書きます。

ここで問題は、この修正で**症状が変わらない**ことです。`claude` は見つかるようになったのに、今度は見つかったうえで `Not logged in` と言ってきます。エラーメッセージが同じなので、直ったのか直っていないのかも分かりにくい。

私はここで「PATH の書き方が悪いのだろう」と考えて、`which claude` の出力を貼り直したり、フルパスで直接叩いてみたりしました。**全部無駄でした。** PATH の話はここで終わっていて、症状の本体は別のところにあります。

## 詰まりどころ 2: ログイン Keychain は GUI セッションの外から読めない

`claude` の OAuth 認証情報は、macOS の**ログイン Keychain** に入っています。

そしてログイン Keychain は、**アクティブな GUI セキュリティセッション（Aqua / loginwindow）の中からしか読めません。**

LaunchDaemon は `system` ドメインで動きます。**そのセッションの外側です。** `UserName` に正しいアカウントを指定しても、それは「どのユーザー権限で動くか」を決めるだけで、GUI セッションの中に入れてくれるわけではありません。

だから、

- ユーザーは合っている
- PATH も通っている
- `claude` バイナリにも到達している
- **それでも Keychain が読めないので `Not logged in`**

という状態になります。ここが「PATH をどれだけ正しくしても直らない」の中身です。**直す対象は環境変数ではなく、プロセスが所属するドメインでした。**

## 対処: LaunchAgent にして `gui/<uid>` に入れる

置き場所とロード先を変えます。

| | LaunchDaemon | LaunchAgent |
| --- | --- | --- |
| 置き場所 | `/Library/LaunchDaemons/` | `~/Library/LaunchAgents/` |
| ドメイン | `system` | `gui/<uid>` |
| ロードに `sudo` | 要る | 要らない |
| 起動タイミング | OS 起動時 | **GUI ログイン後** |
| ログイン Keychain | **読めない** | 読める |

`claude` を呼ぶプロセスは LaunchAgent 一択です。

```xml
<!-- ~/Library/LaunchAgents/com.taka.winenotes-webapp.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.taka.winenotes-webapp</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/WineNotes/venv/bin/python3</string>
        <string>/path/to/WineNotes/webapp/app.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/path/to/WineNotes</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/path/to/wherever/claude/lives:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/path/to/WineNotes/webapp/logs/webapp.log</string>
    <key>StandardErrorPath</key>
    <string>/path/to/WineNotes/webapp/logs/webapp.err.log</string>
</dict>
</plist>
```

登録します。

```bash
cp com.taka.winenotes-webapp.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.taka.winenotes-webapp.plist
```

**`sudo` は要りません。** ユーザー単位のエージェントであって、システムのデーモンではないからです。ここで `sudo` を打ちたくなったら、たぶん間違った方に進んでいます。

## 代償: GUI にログインしていないと動かない

いいことばかりではありません。**LaunchAgent は、誰かが実際に GUI にログインするまで起動しません。** OS が起動しただけでは動きません。

FileVault を有効にしていると、これは「**起動時に誰かがパスワードを打つ必要がある**」という意味になります。完全なヘッドレス自動ログインはできません。電源を入れて SSH で入るだけ、という運用だと Web UI は永遠に立ち上がりません。

私の場合は Mac mini の GUI セッションを常時ログインしたままにしてあるので、実害はありませんでした。**元々そういう置き方の機械だった**ので、この制約とは相性がよかった、というだけの話でもあります。再起動した後だけ、一度ログインを通してやる必要があります。

この代償が飲めないなら、次の分割を検討することになります。

## 設計: `claude` を使う部分だけ切り出す

医学雑誌の新着をモニタするツールでは、常駐プロセスを 2 つに割りました。

| プロセス | 種別 | 理由 |
| --- | --- | --- |
| ファイル配信サーバー | **LaunchDaemon** | `claude` を触らない。ヘッドレス再起動でも生き残ってほしい |
| 要約サーバー | **LaunchAgent** | `claude -p` を呼ぶ。Keychain が要る |

plist にもその理由をコメントで書いてあります。

```xml
<!-- LaunchAgent (~/Library/LaunchAgents/, gui/<uid>), NOT a LaunchDaemon:
     this process shells out to `claude -p`, whose OAuth login lives in
     the login Keychain, only reachable from an active GUI session. -->
```

分けておく利点は、**常時動いてほしい部分が `claude` まわりの都合に巻き込まれない**ことです。GUI にログインしていない間、要約ボタンは効きませんが、レポートの閲覧はできます。片方の制約がもう片方に伝染しません。

逆に言うと、**全部 LaunchAgent にしてしまうと、`claude` と何の関係もない機能まで「GUI ログイン待ち」になります。** 最初は 1 つのアプリ全体を LaunchDaemon で組んでいて、AI 機能のために丸ごと LaunchAgent へ移しました。動くようにはなりましたが、その時点で「再起動後、ログインするまで何も見られない」性質を全体に広げていたことになります。分けるなら最初から分けたほうがいいです。

### ついでに: `claude` を使わないジョブは LaunchDaemon のままでいい

上の表の配信サーバーと同じく、定期チェックのジョブも LaunchDaemon です。1 時間おきに外部 API を叩いて新着を確認するだけなので、Keychain も GUI も要りません。

```xml
<key>StartInterval</key>
<integer>3600</integer>
<key>RunAtLoad</key>
<true/>
```

「`claude` を呼ぶか」だけが判断基準です。それ以外は、ヘッドレスでも動く LaunchDaemon のほうが素直です。

## 運用で使うコマンド

`sudo` なしで完結します。`gui/$(id -u)` の `<uid>` は普通 `501` です。

```bash
# コードを更新した後の再起動
launchctl kickstart -k gui/$(id -u)/com.takahiro.restaurantrating.webapp

# 状態確認
launchctl print gui/$(id -u)/com.takahiro.restaurantrating.webapp

# 停止・登録解除
launchctl bootout gui/$(id -u)/com.takahiro.restaurantrating.webapp

# ログ
tail -f ~/Library/Logs/RestaurantRating/webapp.log
tail -f ~/Library/Logs/RestaurantRating/webapp.err.log
```

`launchctl print` を登録直後に叩くと、`state = xpcproxy` と出ることがあります。**これは起動途中であって、ハングではありません。** 少し待つと `running` に変わります。ここで慌てて `bootout` して registered/unregistered を往復すると、何が起きているか分からなくなります。

なお、古い記事でよく見る `launchctl load -w` も動きますが、いまは `bootstrap` / `bootout` のドメイン指定の形が正です。`launchctl help` にもそう書いてあります。

```
load     Recommended alternatives: bootstrap | enable.
unload   Recommended alternatives: bootout | disable.
```

## まとめ

- 常駐プロセスから `claude -p` が **`Not logged in`** になる原因は 2 つあって、直す順番を間違えると詰まる
- **PATH**: launchd はログインシェルの PATH を継がない。plist の `EnvironmentVariables` で明示する。**Agent でも Daemon でも必要**
- **Keychain**: `claude` の OAuth 認証情報はログイン Keychain にあり、**アクティブな GUI セッションの中からしか読めない**
- **LaunchDaemon は `system` ドメイン、つまりそのセッションの外。** `UserName` を正しく書いても入れない。**PATH をどれだけ直しても直らない**
- 対処は `~/Library/LaunchAgents/` に置いて **`launchctl bootstrap gui/$(id -u)`**。`sudo` は要らない
- 代償: **GUI にログインするまで起動しない。** FileVault があるとヘッドレス自動ログインはできない
- **`claude` を呼ぶプロセスだけ LaunchAgent に切り出す。** 全部 Agent にすると、無関係な機能まで GUI ログイン待ちになる
- `launchctl print` の `state = xpcproxy` は起動途中。ハングではない

---

この `claude` CLI を個人ツールから叩く構成そのもの（provider 層の作り方、`--` を忘れるとプロンプトが引数として食われる件、対応しない引数が黙って無視されて嘘の答えが返る件）については、別記事に分けて書きました。

[APIキーなしで個人ツールにAIを足す — claude CLIは楽だが、無視された引数が静かに嘘をつく](https://zenn.dev/takagit/articles/claude-cli-llm-provider)
