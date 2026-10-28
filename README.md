# j — Pokeno Mania invite page

Static GitHub Pages site (`index.html`, served from the repo root as is) that turns an `https://` invite link,
tappable in any messenger, into an app launch, and shows the invited room so a player who has to install the app
first can still join it.

## Invite URL

```
https://pokenomania.github.io/j/?roomId=<room number>&pw=<password>&rid=<room id>
```

The game's "Copy invite link" button builds these links. The parameters:

- `roomId`: the room number. Without it the page opens nothing and shows only the store buttons.
- `pw`: the room password. When it is empty the panel has no Password line.
- `rid`: the server's room id, passed through to the app, which uses it to tell a closed room from a newer one with
  the same number. Left out of the app URL when absent. App versions older than the one that reads `rid` and the Play
  referrer ignore `rid`.
- `noapp=1`: show the panel at once and attempt nothing. Chrome's fallback on Android always lands here when the app
  is not installed.

Any other parameter, and any `#fragment`, is ignored.

## What the page does

The page reads `roomId`, `pw` and `rid` with `URLSearchParams` and rebuilds every URL from them with
`encodeURIComponent`; it never copies the raw query.

1. With `roomId` (and without `noapp=1`) it shows "Opening Pokeno Mania…" and opens the app:
   - **Android**: `intent://join?roomId=…&pw=…&rid=…#Intent;scheme=pokeno;package=com.pokenomania.game;S.browser_fallback_url=…;end`.
     Chrome launches the app by package. When the app is not installed, Chrome goes to `browser_fallback_url`: this
     page with the same three values and `noapp=1`.
   - **iPhone and everything else**: `pokeno://join?roomId=…&pw=…&rid=…`.
2. After 2 s (at once with `noapp=1`) it shows the room panel:
   - the room number and the password, each with a Copy button that reads "Copied" for 2 s ("Copy failed" if the
     browser refuses, with an error in the console). Without the Clipboard API, as on a plain `http://` LAN address
     (`localhost`, `127.0.0.1` and `[::1]` are secure contexts and keep it), the Copy buttons are hidden; on desktop,
     one mouse click still selects the whole value;
   - **Open in Pokeno Mania**: the same app URL as above;
   - **Get it on Google Play**, on every platform, iPhone included:
     `https://play.google.com/store/apps/details?id=com.pokenomania.game&referrer=…`. The `referrer` carries the same
     three values (`roomId=…&pw=…&rid=…`, encoded once more), which the app reads once after a Play install (app
     versions older than the one that reads `rid` and the referrer never read it);
   - **Download on the App Store**, only when `IOS_STORE_URL` is set;
   - the help line: "Just installed the app? Tap your invite link again, or Open in Pokeno Mania above. On Android
     the app may already show this room. You can also open the app, tap Join Room By Number and enter the room
     number and password."

Values from the query are only ever shown with `textContent`, never as HTML.

## The one setting

`IOS_STORE_URL`, at the top of the script, is empty until an App Store listing exists. Setting it shows the
"Download on the App Store" button, linking to that URL, beside the Google Play button (also on the page without an
invite).

## Checking a change

```
python3 -m http.server 8000
```

Open `http://localhost:8000/?roomId=123456&pw=ab%26c%25d&rid=r-1` in a desktop browser, then with an Android and an
iPhone device preset in the browser's developer tools. Expected:

- "Opening Pokeno Mania…", then after 2 s the panel with room number `123456` and password `ab&c%d`; Copy pastes
  exactly those values. Desktop Chrome logs "Failed to launch 'pokeno://join?…'" because no app handles the scheme
  there: that line is the attempt.
- `document.getElementById("play").href` is
  `https://play.google.com/store/apps/details?id=com.pokenomania.game&referrer=roomId%3D123456%26pw%3Dab%2526c%2525d%26rid%3Dr-1`
  on every preset.
- With the Android preset,
  `decodeURIComponent(document.getElementById("open").getAttribute("href").match(/S\.browser_fallback_url=([^;]*)/)[1])`
  is `http://localhost:8000/?roomId=123456&pw=ab%26c%25d&rid=r-1&noapp=1`.
- `?roomId=%3Cb%3Ex%3C%2Fb%3E&pw=1` shows the room number `<b>x</b>` literally; `?roomId=123456&pw=` shows no
  Password line; the bare page shows only "Get it on Google Play" and opens nothing; the first URL above (the one
  with `roomId`) with `&noapp=1` added shows the panel at once and opens nothing.
- The first URL with `localhost` replaced by the machine's LAN address over plain HTTP (on a Mac:
  `echo "http://$(ipconfig getifaddr en0):8000/?roomId=123456&pw=ab%26c%25d&rid=r-1"`); not `127.0.0.1` or `[::1]`,
  which are secure contexts like `localhost`. `navigator.clipboard` is undefined there, so there are no Copy buttons;
  on desktop, one mouse click still selects the whole value.

## App Links and Universal Links

Android App Links and iOS Universal Links (an `assetlinks.json` and an `apple-app-site-association` on a domain of
our own) would open an installed app straight from the `https://` link, without the browser hop. They do nothing for
a player who still has to install the app, which is what the room panel and the Play `referrer` are for.
