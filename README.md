# j — Pokeno Mania invite redirect

Static GitHub Pages site that turns an `https://` invite link (tappable in any
messenger) into an app launch.

Invite URL shape:

```
https://pokenomania.github.io/j/?roomId=<number>&pw=<password>
```

`index.html` redirects:

- **Android** → `intent://join?...#Intent;scheme=pokeno;package=com.pokenomania.pokeno;...end`
  (Chrome launches the app by package; `browser_fallback_url` → Play Store if not installed)
- **iOS / other** → `pokeno://join?...` custom scheme

The client builds this URL in `InviteLink.Build()`. The app's deep-link handler
parses `roomId` (room number) + `pw` and pre-fills the Join Room popup.

When `pokenomania.com` DNS is available, migrate to true Android App Links /
iOS Universal Links (host `assetlinks.json` + `apple-app-site-association`) to
drop the browser hop.
