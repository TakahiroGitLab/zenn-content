# 検証記録

記事の記述を実際に動かして確かめた記録。**Zenn が公開するのは `articles/` `books/` `images/` だけ**なので、
このファイルは記事には出ない。ただし `zenn-content` リポジトリ自体は public なので、GitHub 上では読める。

最終実施: 2026-08-30 / macOS 26.5.2, uid 501, FileVault On

記事本文には検証環境の 1 行だけを残し、「何を再測定していないか」の棚卸しはここに移した。
読者は書いてあることを著者が確かめたものとして読むので、記事側で自己申告すると
書いていない箇所まで疑わせることになるため。

---

## claude-cli-llm-provider

環境: Claude Code 2.1.251 / Python 3.9.6 / Flask 3.1.3

| 主張 | 方法 | 結果 |
| --- | --- | --- |
| `--` を省くとプロンプトが引数に食われる | `claude -p --add-dir /tmp ZZZ` / `--allowedTools WebFetch ZZZ` を各2回 | 4回とも `Error: Input must be provided either through stdin or as a prompt argument when using --print` |
| `--` ありなら通る | `claude -p --add-dir /tmp -- 'Reply with exactly: OK-42'` | `OK-42` |
| `--add-dir` / `--allowedTools` は可変長 | `claude --help` | `<directories...>` `<tools...>` |
| `--output-format json` のエンベロープ | 実行して JSON のキーを列挙 | `result`(文字列) と `is_error` を確認。`claude-haiku-4-5` は `claude-haiku-4-5-20251001` に解決 |
| 画像をパス参照で読む | 上からマゼンタ/黄/シアンの帯を持つ PNG を生成し `--add-dir /tmp` で質問 | `Magenta, yellow, cyan`（生成どおり） |
| `--add-dir` なしだと拒否される | 同じ質問を `--add-dir` なしで | `permission_denials` に Read が入り、モデルは「見ていない」と回答 |
| `--allowedTools` の表記 | スペース区切りとカンマ区切りの両方で example.com を取得 | どちらも許可され `Example Domain`。未許可だと WebFetch が拒否 |
| `_extract_json_array` | 素の JSON / 複数行フェンス / 前後に一言 / 同一行フェンスの4例 | 全て復旧。`and "\n" in text` を外すと同一行フェンスだけ空になることも再現 |
| 長さ・型の検証 | 件数不足・非 dict・数値・null | すべて弾く |
| バッチ分割と再開 | 50件を8件ずつ、3バッチ目で例外 | 24件残り、再開して50件 |
| Flask の並行性 | 2秒スリープのエンドポイントに同時2本 | 既定 2.04秒（並行）/ `threaded=False` 4.04秒（直列）。`Flask.run` に `options.setdefault("threaded", True)` |

### 記事を直した点
- `check=True` は stderr を「捨てる」ではない。`str(exc)` に入らないだけで `exc.stderr` には残る（実測）。見出しと本文を修正。
- 「単一プロセスなのでサーバー全体が止まる」は誤り。Flask は既定で threaded。節を書き直した。
- 「`--` を省くとエラーにならない」は誤り。エラーになるが、内容が「入力がない」と誤誘導する。

---

## launchagent-claude-cli-keychain

| 主張 | 方法 | 結果 |
| --- | --- | --- |
| launchd はログインシェルの PATH を継がない | `EnvironmentVariables` なしの LaunchAgent を `gui/501` に登録 | `PATH=/usr/bin:/bin:/usr/sbin:/sbin`、`which claude: claude not found` |
| PATH を書けば見つかる | 同じジョブに `EnvironmentVariables` を追加 | `/Users/taka/.local/bin/claude` |
| LaunchAgent なら Keychain が読める | 上のジョブから `claude -p -- 'Reply with exactly: AGENT-OK'` | `AGENT-OK` |
| 認証情報はログイン Keychain | `security find-generic-password -s "Claude Code-credentials" login.keychain-db`（`-w` なし、シークレットは出さない） | `svce="Claude Code-credentials"`, `acct="taka"` |
| bootstrap に sudo は不要 | `launchctl bootstrap gui/501 <plist>` | 成功 |
| load より bootstrap が現行 | `launchctl help` | `load: Recommended alternatives: bootstrap \| enable` |
| PaperBrief の分割 | `/Library/LaunchDaemons/` と `~/Library/LaunchAgents/` を突き合わせ | checker=Daemon, statusserver=Daemon, summarizeserver=Agent |
| 稼働中の Agent | `launchctl print gui/501/<label>` | winenotes / restaurantrating / paperbrief-summarize すべて `state = running` |
| FileVault | `fdesetup status` | `FileVault is On.` |

検証用に作った `com.zenn-verify.*` は `bootout` で削除済み。

### 未検証
- **system ドメインから Keychain が読めないこと。** `/Library/LaunchDaemons/` への設置に sudo が要るため実施せず。
  根拠は WineNotes / RestaurantRating / PaperBrief で実際に踏み、LaunchAgent へ移して直った記録。
  記事には「自分の環境で確かめるなら」という形で手順を残した。
- `state = xpcproxy` は起動途中の一過性で、任意に再現できていない。

---

## swiftui-app-without-xcode

環境: Command Line Tools のみ、Swift 6.3.2

| 主張 | 結果 |
| --- | --- |
| `swiftc -parse-as-library -typecheck` | `OK` |
| フラグを落とすと | `error: 'main' attribute cannot be used in a module that contains top-level code` |
| `swift build -c release --product` | 成功 |
| `.app` を手で組む → ad-hoc 署名 → 検証 | `Signature=adhoc`, `Identifier=com.example.demoapp`, `--verify --strict` 通過 |
| 署名しない場合 | `_CodeSignature` なし。`code has no resources but signature indicates they must be present`。`Identifier=Demo`（実行ファイル名） |
| `import XCTest` / `import Testing` | どちらも `no such module` |
| `swift script.swift args` | 直接実行できる |
| `.iconset` 10枚 → `iconutil` | `exit 0`、31673 bytes の `.icns` |

### 記事を直した点
- **`iconutil` は「黙って失敗」しない。黙って成功する。** 1枚消しても `exit 0` で無出力、`.icns` はできる。
  展開し直すと `icon_128x128.png` だけ無い。綴り違い（`icon_257x257.png`）でも同様に成功。
- **ad-hoc 署名は identifier に紐づかない。** designated requirement は `cdhash`。
  `--identifier` を固定してもコードが変われば変わる（`8b98674d…` → `4ba23ccd…`）。
  結論（省略できない）は維持し、説明を測定できる範囲に置き換えた。

### 未検証
- TCC が許可をどう保存するか。`~/Library/Application Support/com.apple.TCC/TCC.db` は
  Full Disk Access なしでは開けない（`authorization denied`）。

---

## apple-calendar-entry-log-eventkit

| 主張 | 方法 | 結果 |
| --- | --- | --- |
| 述語1本は4年まで | `EKEventStore.h` | "will only return events within a four year timespan. If the date range … is greater than four years, then it will be shortened to the first four years."（エラーではなく切り詰め。記事本文に引用として追加した） |
| `creationDate` は read-only | ヘッダ + 代入してコンパイル | `@property(readonly)` / `error: cannot assign to property: 'creationDate' is a get-only property` |
| `organizer` は無いことがある | ヘッダ | `readonly, nullable` "The organizer of this event, or nil" |

### 未検証
- AppleScript の所要時間（`whose uid` 2分 vs 直接参照 0.27秒）。Calendar.app の自動化許可を要求するため再測定せず。
- `calendars: []` が `nil` と同じ扱いになること。ヘッダは nil についてのみ言及。カレンダーアクセス許可が必要。
- 認可前に作った `EKEventStore` が空のままになること。同上。
- 選択可能テキストの内容と高さを同時に変えるクラッシュ。GUI が要る。

---

## css-liquid-glass-backdrop-filter

環境: Google Chrome 151（ヘッドレス）

| 主張 | 方法 | 結果 |
| --- | --- | --- |
| 背景がないと灰色の板 | 単色背景とグラデーション背景で同じカードを描画 | 単色では平板、グラデーション上でのみガラスに見える |
| `mask-composite` なしだと白い板 | 疑似要素からマスク指定だけ外して描画 | カード全体が白いグラデーションで覆われ、文字のコントラストが落ちる |
| `rgb(var(--x) / .88)` | computed style | `rgba(124, 58, 237, 0.88)` |
| `@supports` 条件 | `CSS.supports()` | `mask-composite: exclude` true / `-webkit-mask-composite: xor` true / 記事の not 条件は正しく false |
| `prefers-reduced-transparency` | `matchMedia` | 認識される |
| `-webkit-backdrop-filter` | `CSS.supports()` | Chrome では **false**（無印のみ）。記事の「Safari には prefix が必要」を、Safari 18 以降は無印可・両方書くのが安全、に修正 |

### 未検証
- iOS Safari の実機挙動（入力欄 16px 未満での自動ズーム、44px のタップ領域）。

---

## gcal-entry-log-apps-script / gas-webapp-viewport-addmetatag

公式ドキュメントで確認。

| 主張 | 出典 |
| --- | --- |
| HTML ファイル内の `<meta>` は無視される | HtmlOutput リファレンス "Meta tags included directly in an Apps Script HTML file are ignored." |
| `addMetaTag()` は4種類のみ | 同 "Only the following meta tags are allowed:" → viewport / apple-mobile-web-app-capable / mobile-web-app-capable / google-site-verification |
| `Events: list` に作成日時フィルタは無い | Calendar API リファレンス。時刻系は `timeMin` / `timeMax` / `updatedMin` のみ |
| 実行時間上限 6分 | Apps Script クォータ "6 min / execution" |
| `executeAs` の意味 | マニフェスト リファレンス。`USER_ACCESSING` = アクセスした人として実行 / `USER_DEPLOYING` = デプロイした人として実行。`access` は誰が実行できるかのみ |
| clasp のサブコマンド | clasp 3.3.0 の `--help`。`list-deployments` / `create-deployment -i <id>`（"The deployment ID to redeploy"）/ `push -f` |

### 未検証
- iOS 実機での 16px 自動ズームと 44px タップ領域（上と同じ）。

---

## gas-chat-api-gcp-disabled

環境: Google Apps Script（V8）、clasp 3.3.0、Google Workspace（GCP はオフのまま）。

**2026-09-08、記事の結論が実測でひっくり返り、09-09 に実運用まで確認できた。** 初稿は「Chat の Advanced Service には標準
Cloud プロジェクトが要り、管理者が GCP を止めているので読めない」という内容で、根拠は前提条件の
ドキュメントとプロジェクト作成の拒否だけだった。前提条件を満たさないまま `Chat.Spaces.list()` を
呼んだところ通ったため、本文を書き直した。**諦める前に呼んでいなかったことが、この記事の元の誤り。**

| 主張 | 方法 | 結果 |
| --- | --- | --- |
| 標準 Cloud プロジェクトを作れない | Cloud コンソールで作成を試行（2026-09-08、`Chat.Spaces.list()` 成功の約2時間後に再確認） | `Google Cloud Platform service has been disabled. Please contact your administrator to turn the service on in the Google Workspace Admin console.` |
| 既定プロジェクトのまま Chat を読める | `probeSpaces()` をエディタから実行 | 参加スペース約100件を列挙。エラーなし。標準プロジェクトも Chat API 構成ページの Chat アプリも無い状態 |
| メッセージ本文も読める | `probePreopMessages()` を実行（2026-09-08、開発セッション側） | 対象スペースから **498 件の投稿**を取得。`spaces.list` だけでなく `spaces.messages.list` も通る |
| 照合機能が実運用に入った | `CHECK_PREOP_SPACE: true`、週次メールに「未提示 / 提示済み」が出ている | 前提条件を満たさない経路のまま稼働中 |
| Advanced Service の例外にステータスコードが無い | `src/Api.js` の実装判断（開発セッション） | 読めるのはメッセージ文字列のみ。`worthRetrying()` は恒久的条件を名指すものだけ除外する反転判定 |
| chat.* スコープが既定プロジェクトで認可される | 上と同じ実行（認可を経て成功） | `chat.spaces.readonly` `chat.messages.readonly` を宣言したまま実行できた |
| Chat を宣言したマニフェストが受理される | scratchpad に `.clasp.json` だけ置いて `clasp pull`（読み取りのみ） | リモートの `appsscript.json` に `serviceId: chat / v1` と両スコープ。ローカル `src/` と完全一致 |
| Advanced Service は既定プロジェクトで足りる | Apps Script「拡張サービス」ドキュメント | "If using a default Google Cloud project (created automatically by Apps Script), skip this step. The API is enabled automatically when you add the service in Step 1." |
| Chat のページは標準プロジェクトを要求している | Chat Advanced Service ページの Prerequisites | "The app's Apps Script project must use a standard Google Cloud project instead of the default one created automatically for Apps Script projects." **実測と食い違う。記事はこの食い違い自体が主題** |
| 構成ページで埋める項目 | Chat / Apps Script クイックスタート | App name / Avatar URL / Description / Connection settings / Deployment ID。ボットを作る手続きであることの根拠として引用 |
| ユーザー認証側に標準プロジェクト要求は無い | Chat API 認証ガイド | ユーザー認証とサービスアカウント認証を分けて説明しており、前者に標準プロジェクトの要求は書かれていない |
| 管理コンソールで GCP をオフにできる | Workspace 管理者ヘルプ「その他の Google サービスを有効または無効にする」 | メニュー → アプリ → その他の Google サービス → サービスのステータス。組織部門ごとに設定可 |

### 未検証
- **書き込み系。** 読み取り 2 種は確認したが、メッセージ投稿などは試していない。記事も
  読み取りの話に限定してある。
- **この挙動が保証されているか。** ドキュメントが要ると書いているものを満たさずに通っている以上、
  Google 側の実装都合で塞がりうる。記事は「2026-09-08 に動かした記録」として書いてある。
- **管理者が設定を変えていないこと。** GCP は同日に無効のままであることを確認したが、
  Apps Script のプロジェクト設定が既定プロジェクトのままかは画面で見ていない
  （標準プロジェクトを作れない以上、紐付けようがないという推定に留まる）。

### 取り下げた記述
- 「Workspace 系 API の呼び出しに課金アカウントの紐付けは要らない」— 根拠がなく撤回。
  Calendar API のクォータページは "All standard use of the Google Calendar API is available at
  no additional cost." としつつ、上限超過分の課金を 2026 年後半に予定と書いている。
- 「OAuth 同意画面を内部にすれば審査不要」— 条件が不足。免除条件は
  "The project must be owned by the organization" も含む。書き直しでこの節ごと不要になった。
