---
title: "APIキーなしで個人ツールにAIを足す — claude CLIは楽だが、無視された引数が静かに嘘をつく"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "python", "llm", "cli", "anthropic"]
published: true
---

自分用の小さなツールに「AI に聞く」を足したくなることがあります。ワインのラベル写真からフィールドを埋める、行ったことのない店を提案してもらう、医学雑誌の新着号を要約する。どれも個人用で、使うのは自分ひとりです。

素直にやるなら Anthropic API を叩くのですが、そのために増えるものが地味に多い。

- `ANTHROPIC_API_KEY` の発行と管理（git に入れない仕組みも要る）
- 従量課金。使うたびに減るので、気軽に試せない
- 画像を渡したければ base64 にしてメッセージに積む実装
- Web 検索させたければ、そのための道具立て

一方、手元の Mac には Claude Code が入っていて、**すでにログイン済み**です。定額も払っている。ならばそれを使えばいい、というだけの話です。

```python
subprocess.run(["claude", "-p", "--", prompt], capture_output=True, text=True)
```

本質はこの 1 行です。ただしこのまま書くと必ず踏む穴がいくつもあり、**そのうち 1 つは、エラーを出さずに間違った答えを返してきます。** この記事はその記録です。

なお、これは個人ツールの話です。自分の端末から自分が使う分にはサブスクリプションの範囲ですが、不特定多数にサービスとして提供するなら話が変わります。そこは API の領分です。

## 何を作ったか

3 つの個人ツールで、同じ形の `ai/` 層に落ち着きました。

| ツール | AI にやらせること |
| --- | --- |
| ワインのテイスティング記録 | ラベル写真からフィールドを埋める / 好みのプロファイルを書く |
| レストラン評価 | 訪問履歴と Web 検索から、まだ行っていない店を提案する |
| 医学雑誌モニタ | 新着号の全論文を英日 2 言語で要約する |

3 つとも `ANTHROPIC_API_KEY` を持っていません。`claude` コマンドが入っていてログイン済みであること、それだけが前提です。

ソースは公開していません。ワインの記録も訪問した店の評価も、そのまま個人データなので、リポジトリごと私物です。この記事にはコードの側だけ抜き出します。

## まず、差し替えられる形にする

`subprocess.run` をアプリのロジックに直接書かないこと。ここだけは最初にやる価値があります。

```python
# ai/providers/base.py
class LLMProvider(ABC):
    name: str
    model_version: str

    @abstractmethod
    def ask(self, prompt: str, allowed_dirs=None, allow_web: bool = False) -> str:
        """プロンプトを1回投げて、テキストの返事を返す。"""
```

```python
# ai/registry.py
PROVIDERS = {
    "dummy": DummyProvider,
    "claude_code": ClaudeCodeProvider,  # ローカルの `claude` CLI。APIキー不要
    "claude_api": ClaudeAPIProvider,    # 直接API。ANTHROPIC_API_KEY が必要
}

DEFAULT_PROVIDER = "claude_code"


def get_provider(name: str) -> LLMProvider:
    if name not in PROVIDERS:
        available = ", ".join(sorted(PROVIDERS))
        raise ValueError(f"Unknown provider {name!r}. Available: {available}")
    return PROVIDERS[name]()
```

呼ぶ側はこれだけです。

```python
provider = get_provider(provider_name)
return provider.ask(prompt, allow_web=True)
```

エンジンを増やすときに触るのは、**`ai/providers/` にファイルを 1 つと、`registry.py` に 1 行**。既存のコードは 1 行も変わりません。既定エンジンの切り替えも `DEFAULT_PROVIDER` の 1 行です。CLI の `--provider` も `choices=sorted(PROVIDERS)` で済むので、選択肢を二重に書く場所ができません。

### ついでに: 各 provider は重い import をメソッドの中でやる

`claude_api.py` の `import anthropic` は、モジュールの先頭ではなく `__init__()` の中に書きます。

```python
class ClaudeAPIProvider(LLMProvider):
    def __init__(self, model: str = None):
        import anthropic          # ← ここ。モジュール先頭ではない
        api_key = os.environ.get("ANTHROPIC_API_KEY")
        ...
```

`registry.py` は全 provider を import するので、先頭に書くと **`anthropic` を入れていない環境では `claude_code` すら使えなくなります**。「どんな provider があるか見る」だけの操作が、使うつもりのない SDK を要求する状態は避けたいところです。

同じ理屈で、`claude` の存在確認も `__init__()` でやって、代替案まで書いておきます。

```python
def __init__(self):
    if shutil.which("claude") is None:
        raise RuntimeError(
            "'claude' コマンドが PATH にありません。Claude Code を入れるか、"
            "--provider claude_api / dummy を使ってください。"
        )
```

## CLI 経由だからタダで付いてくるもの

先に利点を書きます。正直、これ目当てで選ぶ価値があります。

**画像**。`claude -p` はファイルを読む道具を持っているので、プロンプトの中で**パスに言及するだけ**で読んでくれます。アップロード処理も base64 も要りません。

```python
# ワインのラベル写真からフィールドを埋める
provider.ask(prompt_text, allowed_dirs=[image_path.parent])
```

プロジェクトの外にあるファイルを指すときだけ、`--add-dir` でそのディレクトリを許可します。上下 3 色の帯を持つ PNG を作って試すと、こうなります。

```console
$ claude -p --add-dir /tmp -- 'Open the image at /tmp/band-test.png. Reply with ONLY the three horizontal colour bands, top to bottom.'
Magenta, yellow, cyan
```

生成したとおりの並びです。**ここで注目したいのは、許可しなかった場合の振る舞いのほうです。** `--add-dir` を外すと、拒否されたことがエンベロープに残り、モデルも「見えていない」と言います。

```console
$ claude -p --output-format json -- 'Open the image at /tmp/band-test.png. ...'
permission_denials: [{"tool_name": "Read", "tool_input": {"file_path": "/tmp/band-test.png"}}]
result: "I can't read that file — the permission request ... was declined,
         so I never saw the image and can't tell you what the bands are."
```

**読めなかったことを、読めなかったと言います。** これは後述の詰まりどころ 4 と正反対です。同じ「画像が読めない」でも、CLI 経由なら黙りませんでした。

**Web**。`allow_web=True` で、その 1 回の呼び出しにだけ `WebSearch` と `WebFetch` を渡します。

```python
cmd += ["--allowedTools", "WebFetch WebSearch"]
```

ツール名の並べ方は、スペース区切りでもカンマ区切りでも通ります。どちらも 1 つの引数として渡した場合です。

```console
$ claude -p --output-format json -- 'Fetch https://example.com and reply with ONLY the page title.'
  拒否されたツール: ['WebFetch']
$ claude -p --allowedTools "WebFetch WebSearch" -- '同上'
  拒否されたツール: (なし) / result: Example Domain
$ claude -p --allowedTools "WebSearch,WebFetch" -- '同上'
  拒否されたツール: (なし) / result: Example Domain
```

これで「実在して、いま営業している店」を提案させられます。学習データの中の店ではなく。店の URL を渡して商品リストを抜き出す、といった用途も同じ仕組みで済みます。

必要な権限をその呼び出しの分だけ渡す形になっているのも都合がよくて、写真を読むときに Web は開きません。

## 詰まりどころ 1: `--` がないとプロンプトが引数として食われる

最初の穴でした。

```python
cmd = ["claude", "-p"]
for directory in allowed_dirs or []:
    cmd += ["--add-dir", str(directory)]
if allow_web:
    cmd += ["--allowedTools", "WebFetch WebSearch"]
# --add-dir も --allowedTools も可変長なので、"--" がないと
# プロンプトを「もう1つのディレクトリ名」として飲み込んでしまう
cmd += ["--", prompt]
```

`--help` を見ると、どちらも可変長だと分かります。

```
--add-dir <directories...>            Additional directories to allow tool
--allowedTools, --allowed-tools <tools...>
```

`<...>` が付いているオプションは、後ろに続く位置引数を値として食べ続けます。`--` で切らないと、プロンプト本体がディレクトリ名やツール名として吸い込まれます。

厄介なのは、**そのとき出るエラーが別のことを言う**点です。実際に試すとこうなります。

```console
$ claude -p --add-dir /tmp ZZZ-definitely-not-a-directory
Error: Input must be provided either through stdin or as a prompt argument when using --print

$ claude -p --allowedTools WebFetch ZZZ-my-prompt
Error: Input must be provided either through stdin or as a prompt argument when using --print
```

「プロンプトを渡せ」と言われます。**渡しているのに、です。** プロンプトが直前のオプションに食われて、位置引数が 1 つも残らなかった結果なのですが、メッセージからはそう読めません。渡し方を疑ってクォートや改行をいじり始めると、しばらく戻ってこられません。

## 詰まりどころ 2: `check=True` の例外メッセージには、失敗の理由が入らない

`subprocess.run(..., check=True)` は一見きれいです。しかし失敗時に投げる `CalledProcessError` を `str()` すると、こうなります。

```
Command '['/path/to/python3', 'failing.py']' returned non-zero exit status 1.
```

**stderr の中身がありません。** `claude` が落ちる理由はだいたい「ログインしていない」「レート制限」といった、ユーザーに伝えるべきものです。それが消えると、Web UI のエラーバナーには `str(exception)` 経由で「exit status 1」としか出ません。原因が分からない画面ができあがります。

正確に言うと、**捨てられているわけではありません。** `capture_output=True` なら例外オブジェクトの `exc.stderr` に中身は残っています。

```python
except subprocess.CalledProcessError as exc:
    print(str(exc))     # Command '[...]' returned non-zero exit status 1.
    print(exc.stderr)   # Not logged in
```

つまり `check=True` 自体が悪いのではなく、**`str(exception)` をそのまま画面に出す書き方と組み合わさると理由が消える**、という話です。捕まえて `exc.stderr` を読むなら `check=True` のままでも構いません。私は分岐を 1 つ減らしたかったので、自分で見る形にしました。

```python
result = subprocess.run(cmd, capture_output=True, text=True)
if result.returncode != 0:
    detail = result.stderr.strip() or result.stdout.strip()
    raise RuntimeError(f"claude -p failed (exit {result.returncode}): {detail}")
return result.stdout.strip()
```

## 詰まりどころ 3: timeout がないと、そのリクエストは永遠に返らない

`claude` が Web 検索で詰まると、`subprocess.run` は待ち続けます。timeout を書いていなければ、**そのリクエストは永遠に返りません。** ブラウザのタブは回り続け、スレッドも解放されません。

```python
try:
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=180)
except subprocess.TimeoutExpired:
    raise RuntimeError("AIの応答が180秒を超えました。条件を絞って再試行してください。")
```

ここで、私が長らく誤解していたことを書いておきます。**「Flask の開発サーバーは 1 リクエストずつしか捌かないので、遅い AI 呼び出しが全ページを止める」と思っていました。違いました。**

`app.run()` は **Flask 1.0 以降 `threaded=True` が既定**です。手元の Flask 3.1.3 の `Flask.run` にも、そのまま入っています。

```python
options.setdefault("threaded", True)
```

2 秒眠るだけのエンドポイントを立てて、同時に 2 本投げると差が出ます。

| `app.run()` の指定 | 2 リクエスト同時発行の総時間 |
| --- | --- |
| 既定のまま | **2.04 秒**（並行） |
| `threaded=False` を明示 | **4.04 秒**（直列） |

リクエストごとにスレッドが割り当てられるので、止まるのは「サーバー全体」ではなく「そのリクエスト」でした。明示的に `threaded=True` と書いても、既定の再確認にしかなりません。

```python
app.run(host="0.0.0.0", port=port, debug=debug, threaded=True)
```

とはいえ、**返らないリクエストが溜まっていくこと自体は困ります**し、timeout を入れない理由にはなりません。

そして threaded であることには別の請求書が付いてきます。**共有状態にロックが要る**ことです。AI の提案は 1 分近くかかることがあり、スマホでブラウザをバックグラウンドに送ると、AI が答え終わっていてもタブが白くなって結果が消えます。そこで直近の結果をメモリに持たせたのですが、この変数はスレッド間で共有されるので、ロックで守る必要がありました。

```python
_last_suggestion_lock = threading.Lock()
_last_suggestion = None
```

## 詰まりどころ 4: 「無視してよい引数」は、エラーを出さずに嘘をつく

これが一番痛い失敗です。

インターフェースにこう書いていました。

> `allowed_dirs` / `allow_web`: これらに対応できない provider（素のテキスト API 呼び出しなど）は**無視してよい**。

一見きれいな設計です。実際に何が起きたか。

`claude_api` は画像を読めません。`allowed_dirs` を受け取っても無視します。そこに `--image` 付きで `--provider claude_api` を指定すると、**画像パスがただの文字列としてモデルに渡り、モデルは開いてもいないファイルについて、自信たっぷりに答えます。**

例外は出ません。ログにも何も出ません。ただ、まったく違うワインの説明が返ってきます。**動いているように見えるまま、内容だけが嘘**という壊れ方でした。

上で見たとおり、`claude` CLI に画像を渡し忘れたときは「見えていない」と言ってくれます。**黙って嘘をついたのは CLI ではなく、こちらが書いた抽象化のほうでした。**

対処は、呼び出し側で明示的に弾くこと。

```python
if args.image and args.provider == "claude_api":
    parser.error("--image は claude_api では使えません（テキスト専用）。claude_code か dummy を使ってください")
```

3 つのスクリプトのうち 2 つは最初からこれを書いていて、写真検索の 1 つだけ書き忘れていました。裏を返すと、**「無視してよい」を許すインターフェースは、能力の有無を呼び出し側が全部知っている前提**です。抽象化したつもりが、能力の差の分だけ穴として残っていました。

いま設計し直すなら、`LLMProvider` に `supports_images: bool` のような能力フラグを持たせて、基底クラス側で弾きます。呼び出し側に覚えさせる形にした時点で、いつか 1 箇所書き忘れます。

## 詰まりどころ 5: JSON を頼んでも JSON では返ってこない

構造化して受け取りたい場合、`--output-format json` を付けます。返ってくるのは**エンベロープ**で、中身は `result` フィールドに入っています。

```python
completed = subprocess.run(
    ["claude", "-p", prompt, "--model", "claude-haiku-4-5", "--output-format", "json"],
    capture_output=True, text=True, timeout=timeout,
)
envelope = json.loads(completed.stdout)
if envelope.get("is_error"):
    raise SummarizeError(f"claude CLI reported an error: {envelope.get('result')}")
summaries = json.loads(envelope["result"])   # ← ここが素直にいかない
```

終了コードとエンベロープの `is_error` は別物なので、両方見ています。

そして `result` の中身です。プロンプトで「マークダウンのフェンスなし、JSON 配列だけ」と明示しても、Haiku はしばしば ```` ```json ... ``` ```` で包んできます。前後に一言添えてくることもある。

書式でモデルと殴り合っても勝てないので、**寛容に取り出す**方に倒しました。

```python
def _extract_json_array(text: str) -> str:
    text = text.strip()
    if text.startswith("```") and "\n" in text:
        text = text.split("\n", 1)[1]
        if text.rstrip().endswith("```"):
            text = text.rstrip()[: -len("```")]
        text = text.strip()
    # フェンスを剥がせなくても、最も外側の [ ... ] を切り出す
    start, end = text.find("["), text.rfind("]")
    if start != -1 and end != -1 and end > start:
        return text[start : end + 1]
    return text
```

`and "\n" in text` は後から足した条件です。これがないと ```` ```json[{...}]``` ```` のように**フェンスと配列が同じ行にある**場合にフェンス剥がしが空文字列を返す分岐へ落ち、フォールバックである `[...]` の切り出し（この関数の存在理由そのもの）が永久に走りません。生成し終わった出力を丸ごと捨てて「要求した JSON 配列ではありません」と言う、一番もったいない壊れ方をしていました。

### パースできた後も、長さと型を見る

```python
if not isinstance(summaries, list) or len(summaries) != len(articles):
    raise SummarizeError(f"expected {len(articles)} summaries, got ...")

for article, item in zip(articles, summaries):
    english = item.get("english") if isinstance(item, dict) else None
    japanese = item.get("japanese") if isinstance(item, dict) else None
    if not isinstance(english, str) or not isinstance(japanese, str):
        raise SummarizeError("'english'/'japanese' が無いか文字列ではありません")
```

型まで見ているのは実害があったからです。文字列でない値がそのままキャッシュに保存され、ずっと後の HTML エスケープ処理で初めて落ちました。しかもその時点でディスクに載っているので、**再起動のたびに同じ場所で落ち続けます**。キャッシュファイルを手で消すまで直りませんでした。境界で弾いていれば、被害はそのバッチ 1 回で済んだ話です。

## 詰まりどころ 6: 1 回で全部やろうとすると、timeout が出力ごと捨てる

雑誌 1 号ぶん（約 50 本）の要約を 1 回の `claude -p` に投げたら、360 秒でも足りませんでした。そして `subprocess` の timeout は**プロセスを殺す**ので、そこまでに生成されていた分は全部消えます。定額とはいえ使用量は消費しているので、丸損です。

```python
BATCH_SIZE = 8
_TIMEOUT_SECONDS = 240

for i in range(0, len(remaining), BATCH_SIZE):
    batch = remaining[i : i + BATCH_SIZE]
    cache.update(summarize_issue(batch))
    save_partial_cache(partial_path, cache)   # ← 次のバッチに進む前に必ず保存
```

- 1 回の呼び出しを、確実に終わる長さに切る
- **終わったバッチは、次を始める前にディスクへ**書く
- 途中で落ちても、再実行はキャッシュの続きから走る。失うのは落ちたバッチの分だけ

ただ、実際にタイムアウトを解消したのはバッチ分割そのものより、**モデルを Haiku に落としたこと**でした。「何の話か + 何が新しいか」を数文書くだけなら十分ですし、生成が目に見えて速い。上位モデルの使用量を使う理由がありませんでした。`--model` は引数 1 つなので、試すコストもほぼゼロです。

## 常駐させるなら LaunchAgent にする（別記事）

Mac で launchd に登録して常駐させる場合、**LaunchDaemon にすると `claude -p` は動きません。** PATH がないので `claude` が見つからず、PATH を直しても今度は `Not logged in` になります。`claude` の OAuth 認証情報がログイン Keychain にあり、これはアクティブな GUI セッションの中からしか読めないためです。

対処は `~/Library/LaunchAgents/` に置いて `launchctl bootstrap gui/$(id -u)` すること、代償は GUI にログインするまで起動しないことです。launchd の話であってこの記事の主題から外れるので、別記事に分けました。

[常駐させたclaude CLIがNot logged inになる — PATHを直しても直らない理由はKeychainにある](https://zenn.dev/takagit/articles/launchagent-claude-cli-keychain)

## 効いた小さな仕掛け 2 つ

どちらも実装は数行なのに、効き目が大きかったものです。

**dummy provider** は、受け取ったプロンプトをそのまま返します。

```python
class DummyProvider(LLMProvider):
    name = "dummy"
    model_version = "passthrough-1"

    def ask(self, prompt, allowed_dirs=None, allow_web: bool = False) -> str:
        return prompt
```

registry → CLI → Web フォームという配管全体を、AI を 1 回も呼ばずにテストできます。テストがネットワークにもサブスクにも依存しません。

**`--show-prompt`** は、AI を呼ばずにプロンプト文だけを出します。

```python
if args.show_prompt:
    print(build_suggestion_prompt(storage, config, area=args.area, ...))
    return
```

用途が 2 つあって、1 つはプロンプトのデバッグ。もう 1 つは、**別の AI に貼って第二の意見をもらう**ためです。他社のチャットに貼るだけなので、そのための連携実装は要りません。Web UI にも「別の AI 用にプロンプトをコピー」ボタンとして置いてあります。

プロンプト生成と AI 呼び出しを分けていれば自然に手に入るので、分けておく理由のひとつとして数えていいと思います。

## まとめ

- 個人ツールの AI 機能は、`claude -p` を subprocess で叩けば **API キーなし・追加課金なし**で作れる
- `subprocess.run` は provider インターフェースの裏に隠す。**エンジン追加が「1 ファイル + 1 行」**で済む
- **`--` を忘れない。** `--add-dir` や `--allowedTools` は可変長でプロンプトを飲み込み、**エラーは「入力がない」と嘘の方向を指す**
- **`check=True` は使わない。** stderr が消えて「ログインしていない」が伝わらなくなる
- **timeout は必須。** ないとそのリクエストは永遠に返らない。ただし Flask の `app.run()` は既定で threaded なので、止まるのはサーバー全体ではない
- **画像も Web も、パスと `--allowedTools` を渡すだけ。** API ならそれぞれ実装が要る
- **「対応しない provider は無視してよい引数」は、エラーなく嘘を返す。** 能力フラグを持たせて基底クラスで弾く
- **JSON を頼んでも JSON では返ってこない。** フェンスを剥がし、長さと型まで検証してからキャッシュに入れる
- **長い仕事はバッチに割り、終わった分は次を始める前にディスクへ。** timeout はプロセスごと出力を捨てる
- **モデルは落とせるなら落とす。** タイムアウトが消えたのは、バッチ分割よりモデル変更の効果が大きかった
- **常駐させるなら LaunchAgent。** Keychain は GUI セッションの外から読めないので、PATH をどれだけ直しても `Not logged in` は消えない（[別記事](https://zenn.dev/takagit/articles/launchagent-claude-cli-keychain)）

---

検証環境: Claude Code 2.1.251、macOS 26.5.2、Python 3.9.6、Flask 3.1.3。
