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

## Articles

| Slug | Title | State |
| --- | --- | --- |
| `gcal-entry-log-apps-script` | Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った | published |
| `gas-webapp-viewport-addmetatag` | Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している | published |
| `css-liquid-glass-backdrop-filter` | CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない | published |
| `apple-calendar-entry-log-eventkit` | Apple Calendarの「いつ登録したか」を見るMacアプリをEventKitで作った | draft |
