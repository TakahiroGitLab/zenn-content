---
title: "「launchdはスリープに強い」は半分だけ正しい — 追いつくのはStartCalendarIntervalだけ"
emoji: "⏰"
type: "tech"
topics: ["macos", "launchd", "cron", "launchagent", "shell"]
published: false
---

Mac で、あるフォルダを毎日 `git pull` したくなりました。調べると cron と launchd の比較表がいくらでも出てきます。そしてどれも同じ結論に着地します。**launchd を使え、cron と違ってスリープ中に逃した回を追いかけてくれるから。**

その一文を手元の man と突き合わせたら、**合っていたのは半分でした。** さらに、追いつき実行を実際に握っている主体を辿ったら、思っていた場所にいませんでした。

以下はすべて macOS 上で実測しています。再現用のスクリプトは最後に置きます。`sudo` は要りません。

## 追いつくのは `StartCalendarInterval` だけで、launchd の性質ではない

`man launchd.plist` の `StartCalendarInterval` は、比較表が言うとおりのことを書いています。

> Unlike cron which skips job invocations when the computer is asleep, launchd will start the job the next time the computer wakes up.

ところが同じ man の、すぐ上にある `StartInterval` にはこう書いてあります。

> If the system is asleep during the time of the next scheduled interval firing, **that interval will be missed** due to shortcomings in kqueue(3).

| キー | スリープで逃した回 |
| --- | --- |
| `StartCalendarInterval`（時刻指定） | 復帰時に実行される |
| `StartInterval`（N 秒ごと） | **実行されない** |

「launchd はスリープに強い」は launchd 全体の性質ではなく、**キー1つの性質**でした。`StartInterval` を選んだ時点で、この利点は手に入りません。しかも man は理由まで書いています。`kqueue(3)` の都合です。

これは自分に刺さる話でもあります。前に [常駐させた claude CLI が Not logged in になる](https://zenn.dev/takagit/articles/launchagent-claude-cli-keychain) という記事を書いたとき、「定期チェックのジョブは LaunchDaemon のままでいい」という例で `StartInterval` を 3600 で出しました。1 時間おきに外部 API の新着を見るだけなので取りこぼしても次の回で拾えて、そこは今も変わりません。ただ、**あれを「launchd だからスリープに強い」と思って選んだのなら間違いだった**ことになります。

## 「追いつき」ではなく「合流」

`StartCalendarInterval` の引用には続きがあります。

> If multiple intervals transpire before the computer is woken, those events **will be coalesced into one event** upon wake from sleep.

追いつくのは回数ではなく、**1 回に潰れます。** 3 日間フタを閉じていた MacBook は、開いたときに 3 回ではなく 1 回だけ走ります。

毎日 1 回のつもりで組んだジョブが、3 日ぶんの仕事を 1 回で片付けろと言われる。ではジョブの側は、自分がいま合流ぶんとして呼ばれたことを知れるのでしょうか。

## 測定 1: ジョブには何も渡ってこない

`RunAtLoad` でジョブを起こし、`argv` と環境変数を丸ごとダンプしました。

```console
argv: /bin/sh
HOME=/Users/taka
LOGNAME=taka
OSLogRateLimit=64
PATH=/usr/bin:/bin:/usr/sbin:/sbin
PWD=/
SHELL=/bin/zsh
SHLVL=1
SSH_AUTH_SOCK=/var/run/com.apple.launchd.NsMsxhRtel/Listeners
TMPDIR=/var/folders/th/_9lnmzdj5j94n53vhdp2kzfm0000gn/T/
USER=taka
XPC_FLAGS=0x0
XPC_SERVICE_NAME=0
_=/usr/bin/env
```

これで全部です。`argv` は `ProgramArguments` に書いたものだけ、環境変数は 13 行（末尾の `_` は `env` を呼んだシェルが付けるものなので、launchd が渡したのは 12 個）。**予定時刻も、何回ぶんの合流なのかも、渡ってきません。**

つまり合流して起きた 1 回は、普通の 1 回とまったく同じ顔をしています。区別する手段がジョブの側にありません。

ここが cron との本当の違いです。よく言われる対比は「cron は逃す / launchd は追いつく」ですが、実際はこうです。

- cron は逃す。**そして何も言わない。**
- launchd は 1 回に潰す。**そして何も言わない。**

**どちらも黙ります。** 失敗の形が違うだけで、「起動された回数」を数えて仕事の量を決める設計は、どちらでも壊れます。スケジューラの起動回数は契約ではありません。

対処は、どちらを選ぶかとは別の場所にあります。**ジョブが自分で状態を見ること。** 前回どこまで処理したかを自分で記録して、今回の範囲をそこから決める。起動されたという事実には、期間の情報が乗っていないからです。

最初の目的だった毎日の `git pull` は、この点では困りません。3 日ぶんが 1 回に潰れても、`git pull` は結局リモートの現在の状態に追いつきます。**冪等なので回数に意味がない。** 逆に「日次レポートを 1 通送る」「その日ぶんを追記する」といったジョブは、合流した瞬間に 2 日ぶんが消えます。

## 測定 2: 「追いつく」の境界

紛らわしいのが、**過去の時刻を指定して読み込んだとき**です。追いついてくれるなら、さっき過ぎた時刻のぶんも走ってくれそうに見えます。

16:10 に、`00:01` を指定した LaunchAgent を `bootstrap` しました。

```console
現在 16:12:46 / 発火指定 00:01（今日ぶんは過去）
### 1. 過去の時刻を指定して読み込むと、すぐ発火するか
発火していない
	state = not running
	runs = 0
	last exit code = (never exited)
```

走りません。`runs = 0` のままです。

追いつきの対象は、**すでにロードされていたジョブがスリープで取り逃した回**だけです。時刻が過ぎてから読み込んだジョブに、過去は生えてきません。plist を置いた初日は、翌日まで一度も動かないことになります。

## 測定 3: 追いつきを握っているのは Aqua セッション

ではその追いつきは、誰が見張っているのか。`launchctl print` がそのまま答えを持っていました。

```console
### 2. 追いつき実行を握っているのは誰か
			stream = com.apple.launchd.calendarinterval
			monitor = com.apple.UserEventAgent-Aqua
			descriptor = {
				"Minute" => 1
				"Hour" => 0
			}
		"com.apple.launchd.calendarinterval" = {
			port = 0x0
```

`monitor = com.apple.UserEventAgent-**Aqua**`。LaunchAgent の時刻発火を見ているのは、**GUI セッションのユーザーイベントエージェント**でした。

ここで話がつながります。

- 比較表が褒める「スリープを追いかける」挙動は `StartCalendarInterval` のもの
- LaunchAgent でそれを使うと、見張っているのは Aqua セッションのエージェント
- **Aqua セッションは、誰かが GUI にログインして初めて存在する**

うちの Mac mini は FileVault を有効にしているので、再起動後は**人がパスワードを打つまでログインセッションがありません。** ヘッドレスで堅牢に回したくて launchd を選んだはずが、人間がキーボードの前に座ることが前提条件になっていた、という順序になります。

比較表の「スリープ復帰時に追いつく」の行には、この代償が書かれていません。

## 測定 4: daemon と agent を分けているのは uid ではなく `asid`

もうひとつ、同じ `launchctl print` で見える数字があります。

```console
### 4. ドメインの違いは uid ではなく asid に出る
gui/501 : type = login uid = 501 asid = 100015
system  : type = system uid unset asid = 0
```

`asid` は audit session ID です。`gui/501` は `100015` を持ち、`system` は **`0`**、つまりセッションに属していません。

そして `man launchd.plist` の `UserName` は、こう書かれています。

> This key is only applicable for services that are loaded into the privileged system domain.

別の箇所には、こうもあります。

> Note that for agents, the UserName key is ignored.

LaunchDaemon に `UserName` を書けば uid は動きます。**しかし `asid` は動きません。** 「ユーザーとして実行する」ことと「ユーザーのセッションの中で実行する」ことが別物であることが、この 2 つの数字で見えます。

この違いが何を壊すかは[前の記事](https://zenn.dev/takagit/articles/launchagent-claude-cli-keychain)に書きました。あのときは「直す対象は環境変数ではなくドメインだった」で終わりましたが、ドメインが決めているのは Keychain だけではなく、**時刻発火を誰が見張るかもそうだった**わけです。

## ついでに、比較表の 3 つの記述

同じ比較表には、man と突き合わせると通らない記述がいくつかありました。

### 「cron は Apple が非推奨にした」— そうは書いていない

macOS の `man crontab` にある Darwin note の全文です。

> (Darwin note: Although cron(8) and crontab(5) are **officially supported** under Darwin, their functionality has been absorbed into launchd(8), which provides a more flexible way of automatically executing commands. See launchctl(1) for more information.)

「公式にサポートされている」と書いてあります。deprecated の語はありません。`man launchd.plist` 全文を検索しても `deprecat` は 1 件も出ず、cron への言及は 2 箇所だけで、どちらも非推奨の話ではありません。

launchd のほうが柔軟だ、というのは man の言うとおりです。それは「cron は捨てられた」とは別のことです。

### 「WatchPaths でファイル変更をトリガーにできる」— Apple は使うなと書いている

比較表はこれを launchd の長所（多彩なトリガー）として挙げます。当該項の全文はこれだけです。

> **IMPORTANT: Use of this key is highly discouraged**, as filesystem event monitoring is highly race-prone, and it is entirely possible for modifications to be missed. When modifications are caught, there is no guarantee that the file will be in a consistent state when the job is launched.

長所として挙がっていたものに、Apple 自身が「強く非推奨」と書いています。しかも理由が二段構えで、**取りこぼす**うえに、**拾えたときもファイルが中途半端な状態かもしれない**。

ちなみに `kqueue` の名前が man に出てくるのは `WatchPaths` の項ではなく、さきほどの `StartInterval` の項で、しかも欠点の理由としてです。

### 「PATH はほぼ空」— 空ではなく、名前のついた定数

実測した PATH をもう一度置きます。

```
PATH=/usr/bin:/bin:/usr/sbin:/sbin
```

これは思いつきの既定値ではありません。`paths.h` にある定数そのものです。

```console
$ grep _PATH_STDPATH "$(xcrun --show-sdk-path)/usr/include/paths.h"
#define	_PATH_STDPATH	"/usr/bin:/bin:/usr/sbin:/sbin"
```

`man launchd.plist` も、この定数の名前で参照しています。

> In the absence of the Program key, the first element of the ProgramArguments array may be either an absolute path, or a relative path which is resolved using _PATH_STDPATH.

「ほぼ空」ではなく、**`_PATH_STDPATH` で固定**です。空だと思っていると「何も入っていないなら仕方ない」で終わりますが、実際には「4 つ入っているのに Homebrew と `~/.local/bin` だけが無い」状態です。だから `/usr/bin/git` は動くのに `/opt/homebrew/bin/gh` は動かない、という選択的な壊れ方をします。

## 結局、毎日の `git pull` はどう組むか

- `StartCalendarInterval` を使う。`StartInterval` では、その時刻に眠っていた日は落ちる
- 合流は気にしなくていい。`git pull` は冪等で、回数に意味がないので
- ただし plist を置いた初日は走らない。過去の時刻は追いつきの対象外
- PATH は `EnvironmentVariables` で明示する。既定は `_PATH_STDPATH` で、Homebrew も `~/.local/bin` も入っていない
- そして、**GUI にログインするまで動かない**ことを受け入れる

最後の 1 行が、比較表を読んでいるときには見えていなかった条件でした。

なお、このスクリプトが観察しているのは LaunchAgent の場合です。`system` ドメインに置いた場合に時刻発火を誰が見張るかは、`/Library/LaunchDaemons` への設置に `sudo` が要るため、ここでは扱っていません。

## 再現用スクリプト

`sudo` は不要です。最後に `bootout` して撤去します。

```sh
#!/bin/sh
set -u
LABEL=probe.calendar
DIR=$(mktemp -d)
PLIST=$DIR/$LABEL.plist
LOG=$DIR/fire.log

cat > "$PLIST" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>$LABEL</string>
  <key>ProgramArguments</key><array>
    <string>/bin/sh</string><string>-c</string>
    <string>{ echo "argv: \$0 \$@"; env | sort; } > $LOG</string>
  </array>
  <key>StartCalendarInterval</key><dict>
    <key>Hour</key><integer>0</integer><key>Minute</key><integer>1</integer>
  </dict>
</dict></plist>
EOF

echo "現在 $(date '+%H:%M:%S') / 発火指定 00:01（今日ぶんは過去）"
launchctl bootstrap gui/$(id -u) "$PLIST"
sleep 5

echo "### 1. 過去の時刻を指定して読み込むと、すぐ発火するか"
[ -s "$LOG" ] && echo "発火した" || echo "発火していない"
launchctl print gui/$(id -u)/$LABEL | grep -E "^\s+(state|runs|last exit code) "

echo "### 2. 追いつき実行を握っているのは誰か"
launchctl print gui/$(id -u)/$LABEL | sed -n '/com.apple.launchd.calendarinterval/,/}/p' | head -8

echo "### 3. ジョブが受け取る環境（発火時刻は渡ってくるか）"
launchctl kickstart gui/$(id -u)/$LABEL >/dev/null 2>&1
sleep 2
cat "$LOG"

echo "### 4. ドメインの違いは uid ではなく asid に出る"
printf 'gui/%s : ' "$(id -u)"; launchctl print gui/$(id -u) | grep -E "^\s+(type|uid|asid) " | tr -d '\t' | tr '\n' ' '; echo
printf 'system  : '; launchctl print system | grep -E "^\s+(type|uid|asid) " | tr -d '\t' | tr '\n' ' '; echo

launchctl bootout gui/$(id -u)/$LABEL 2>/dev/null
rm -rf "$DIR"
```

## まとめ

- 「launchd はスリープに強い」は `StartCalendarInterval` 限定。`StartInterval` は man が明示的に「逃す」と書いている
- 追いつきは 1 回に**合流**し、ジョブ側にそれを知る手段はない。`argv` にも環境変数にも予定時刻は来ない
- cron は逃して黙り、launchd は潰して黙る。起動回数を仕事量の根拠にしない
- 過去の時刻を指定して読み込んでも発火しない。追いつくのは、ロード済みで取り逃した回だけ
- その追いつきを見張っているのは `com.apple.UserEventAgent-Aqua`。**GUI ログインが前提条件**
- daemon と agent を分けているのは uid ではなく `asid`。`UserName` は uid しか動かさない
- cron は man 上「officially supported」で、非推奨とは書かれていない
- `WatchPaths` は Apple 自身が「highly discouraged」と書いている
- 既定 PATH は空ではなく `_PATH_STDPATH`。4 つ入っていて、Homebrew だけが無い

検証環境: macOS 26.5.2（Darwin 25.6.0、FileVault On）、uid 501、2026-09-13 実測。man の引用はすべて同マシンの `man crontab` / `man launchd.plist` から。
