# zenn-content

Articles published to [Zenn](https://zenn.dev/takagit), deployed from
this repository by Zenn Connect.

Each file in `articles/` is one article. The filename is the URL slug,
and the `published` field in its frontmatter decides whether it is live
or a draft.

## Working on an article

```bash
npm install          # first time only
npx zenn preview     # http://localhost:8000
```

Pushing to `main` deploys, and `published: true` in the frontmatter is
what decides. The dashboard's publish button works too, but this
repository is the source of truth: whatever the dashboard sets, the
next push overwrites. Set the field.

Zenn rate-limits how many articles may be posted in a window. Over it,
the deploy log says the article "was not deployed", and Zenn does not
retry on its own — the article sits there answering 403, which means
Zenn has the article and is treating it as unpublished. (404 would mean
it never synced at all.) Either push again or press publish in the
dashboard once the limit clears; the third article below spent a day
like that before going live.

A 403 only means "unpublished" for an article Zenn already holds. An
article that is `published: false` by design has no public page either,
so the same 404 covers both "synced as a draft" and "never arrived" —
the deploy log is the only place that separates them.

**A title may be at most 70 characters.** Over it the deploy log reads
`デプロイ中断` with `保存に失敗しました（Titleには最大70文字まで使用でき
ます）` naming the file — the run is aborted, not partially applied, so
one long title holds up whatever else was in the push. The count is
characters, not bytes: Japanese, the em-dash and spaces are one each.
Existing titles run 42-62, so that band is the safe place to sit.

## Articles

Ordered by publication. `BACKLOG.md` holds what is not written yet.

| Slug | Title | State |
| --- | --- | --- |
| `gcal-entry-log-apps-script` | Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った | published |
| `gas-webapp-viewport-addmetatag` | Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している | published |
| `css-liquid-glass-backdrop-filter` | CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない | published |
| `apple-calendar-entry-log-eventkit` | Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った | published |
| `swiftui-app-without-xcode` | XcodeなしでSwiftUIのMacアプリを作る — .appは手で組めるが、ad-hoc署名だけは省略できない | published |
| `claude-cli-llm-provider` | APIキーなしで個人ツールにAIを足す — claude CLIは楽だが、無視された引数が静かに嘘をつく | published |
| `launchagent-claude-cli-keychain` | 常駐させたclaude CLIがNot logged inになる — PATHを直しても直らない理由はKeychainにある | published |
| `gas-chat-api-gcp-disabled` | Apps ScriptのChatは既定Cloudプロジェクトのまま読めた — ドキュメントは標準が要ると書いている | published |
| `gmail-plain-text-proportional-font` | 空白で桁を揃えたメールは崩れる — Gmailはプレーンテキストを等幅で表示しない | **draft** |
