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

Pushing to `main` deploys. Set `published: true` to publish.

## Articles

| Slug | Title | State |
| --- | --- | --- |
| `gcal-entry-log-apps-script` | Googleカレンダーの「いつ登録したか」を見る画面をApps Scriptで作った | published |
| `gas-webapp-viewport-addmetatag` | Apps Scriptのウェブアプリでviewportが効かない — HtmlServiceはmetaタグを消している | published |
| `css-liquid-glass-backdrop-filter` | CSSでliquid glassを作る — backdrop-filterだけでは灰色の板にしかならない | draft |
