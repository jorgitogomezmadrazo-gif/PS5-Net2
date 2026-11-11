# Game Hub + PS Proxy (fly.io)

Files (upload all of them, keep them together):
- `public/index.html` - the hub
- `server.js` - small web server
- `psproxy.js` - the PS Proxy
- `psclient.js` - PS Proxy's script that runs inside proxied pages
- `package.json`, `Dockerfile`, `fly.toml`

Deploy: edit `fly.toml` (change the app name), then `fly launch --no-deploy --copy-config` and `fly deploy`.
If a proxy file is missing, the build stops with "MISSING FILE" instead of failing later.

## PS Proxy v2

New in v2:
- **Loop breaker**: a page that reloads itself over and over (frame-busting scripts) is stopped after a few rounds, so the hub no longer freezes.
- **Blank page watchdog + safe mode**: if a page shows nothing, a small "This page looks blank" bar appears with a **Safe mode** button. If the page's scripts crashed, safe mode is tried once automatically. Safe mode is remembered per site (PS tools > reset clears it).
- **Upstream proxy (optional)**: some sites (Reddit, parts of Twitch/YouTube) refuse fly.io's data-center address with "blocked by network security". Code cannot fix that. If you have an HTTP proxy with a different address, run:
  `fly secrets set UPSTREAM_PROXY=http://user:pass@host:port`
  and every request (and WebSocket) goes through it. Unset it to go back to direct.
- Stale keep-alive connections are retried once, abandoned downloads are cancelled, `/ps/_status` reports `version: "2.0"`.
- fly.toml: higher connection limit and 1 GB memory (see comments in the file).
- **Debug page**: after a page fails, open `/ps/_debug` on your site. It lists what each site answered (403 from Cloudflare = the site blocks this server's address) and which scripts crashed. Screenshot it when asking for help.

## v3 changes
- Fixed freeze: link fixer no longer loops on app links (intent://, android-app://); app links are ignored instead of blanking the page.
- Faster storage/cookies, service-worker stand-in, frame-buster checks neutralized.
- Video: HLS (.m3u8) and DASH (.mpd) manifests are rewritten through the proxy; Range requests pass through; iframe allows autoplay/fullscreen/PiP.
- Sites that block datacenter IPs (e.g. Reddit): set UPSTREAM_PROXY. Debug log: /ps/_debug. DRM video is not supported.

## v3.1 – new Music app (PS Music)
- Rebuilt Music app: 3-panel layout (library, artist/album pages, now-playing with synced lyrics from lrclib.net).
- Import (top right): Spotify track / album / playlist / artist links or direct audio URLs, plus files from your device.
- Spotify imports play 30s previews in the app; the Spotify button under the lyrics swaps in Spotify's own player for the full song (needs a Spotify login in the browser).
- Reading album/playlist track lists uses a new server endpoint /ps/_raw (allow-listed to Spotify + lyrics hosts), so psproxy.js must be uploaded too.
- Lyrics: matched by artist + cleaned title + song length (wrong-length versions are rejected), 60fps sync, per-song timing nudge (− / +), tap-a-line sync for 30s previews, exact sync through the Spotify player (iFrame API).
- Spotify imports no longer use 30s previews: they play in full through Spotify's own player (hidden behind the app's controls; needs a Spotify login in the browser). Lyrics sync to its exact position.
- Optional "Sign in with Spotify" (Settings in the Music app): Web Playback SDK, Premium required. Create an app at developer.spotify.com/dashboard, add the Redirect URI shown in Settings (https://YOUR-APP.fly.dev/spotify-callback.html), paste the Client ID. public/spotify-callback.html must be uploaded.

## PS Proxy v3
- Video: blob web workers (used by Twitch, YouTube and most players for their stream code) are now wrapped so their playlist and segment downloads go through the proxy; Twitch's `#EXT-X-TWITCH-PREFETCH` lines and unquoted playlist URIs are rewritten.
- Speed: connection pool 96 -> 512 sockets per site, DNS answers cached 60s, TCP no-delay, scripts/styles/images/fonts the site marks cacheable are kept in a 96MB memory cache (and cached in the browser), everything the proxy rewrites is sent compressed (brotli/gzip).
- /ps/_status shows version 3.0 and cache stats; /ps/_debug is unchanged.
- v3.1 reliability: video requests (playlists, segments) are retried automatically (server side up to 4 tries on resets, timeouts and 5xx; browser/worker side 2 more), stalled video downloads are cut after 25s so the player re-requests, and Twitch video servers get twitch.tv Origin/Referer headers.

## Using a Wisp server (optional)
Set the secret/env var `WISP_URL` to your Wisp server's address, e.g. `fly secrets set WISP_URL=wss://your-wisp-server.example/wisp/`.
All upstream connections then go through it (a Wisp server relays TCP over one WebSocket), so websites see the Wisp server's address instead of Fly's.
TLS still ends inside PS Proxy, so the relay sees only host:port. If both are set, `UPSTREAM_PROXY` wins. /ps/_status shows `"wisp": true` when active.
Only use a Wisp server you run yourself or that its owner lets you use.
- Control Center > Proxy (globe icon): turn PS Proxy on/off and enter your own Wisp server (wss://.../wisp/). It is tested when you save, remembered by the hub and re-applied to every new session. Per-session on the server, so it never affects other visitors. Private/local addresses are refused unless the env var ALLOW_PRIVATE_WISP=1 is set.
- Twitch "Error #2000": the proxy now presses Twitch's "reload the player" button for you (max 4 times a minute). fly.toml no longer lets the machine sleep (auto_stop_machines off, min_machines_running 1). If you have two machines, run `fly scale count 1` so Twitch's stream tokens always see the same server/IP.

## Console chooser (v3.4)
On start the hub asks "Choose a console UI to load in!": PS5 (the normal hub, boot animation and all) or Nintendo Switch 2 (a light Switch-style home screen with the same games, Games/Media tabs, search, music, control center, add game, and a "change console" button). Pick with ← → + Enter, a controller, or a tap. The last choice is pre-selected next time. The Switch screen uses the same game list and launcher, so everything you add shows up in both.

## Switch 2 menu (v3.6)
- The Switch 2 home screen is now the dark Switch 2 style: big game tiles with a glowing selection frame and the game name above, a round-icon dock (Search, Music, Media/Games, Add game, Settings, Change console) with a gradient ring and coloured label, clock / Wi-Fi / battery, and the A OK hint bar.
- Sounds live in `public/sounds/switch2/` and are matched to actions: move = navigation, A = default_activation, start a game = launch_game, end of row = bumper_end, search / add game open and close = show_modal / hide_modal, settings panel = side_menu_fly_in / fly_out, Games<->Media = tab_transition, notifications = message_toast / achievement_toast, typing = typing.
- Choosing the PS5 UI is unchanged and keeps its own sounds.
- v3.6 adds the full Switch 2 system: the dock has the same ten icons as the console (User Page, Chat, News, Game Library, Album, GameShare, Controllers, Sleep Mode, System Settings, Power) and each opens a real screen backed by hub data.
  - System Settings: Accessibility, Data Management, Users, Controllers & Accessories, Audio, Display, Themes (light / dark), Notifications, Sleep Mode, Internet, System.
  - Album = all games as a grid (filter / sort), News = hub notifications, Game Library = browse / search / add games, Controllers = connected gamepads + help, User Page = profile + play activity.
  - Back = B / Escape / Backspace or the Back button; OK = A / Enter / tap.

## Nintendo eShop (v3.7)
The red **Nintendo eShop** dock icon is now the way to install games, laid out like the Switch 2 eShop (red header, left menu, source tabs, card grid, Back / Confirm bar).
- Menu: Featured, Search/Browse, For You, Wish List, Great Deals, Best Sellers, Recent Releases, Coming Soon, Downloaded Software, Enter Code.
- Tabs (L / R, `[` `]` or the controller bumpers) switch between the gn-math and Selenite libraries; "Hide Installed Games" filters what you already have.
- A on a card opens its page: Download (confirm dialog, progress toast, "Software download complete"), then Start. Heart adds it to the Wish List.
- Games install through the same code as the old library dialog, so they land on your Home screen like any other game. Everything is free.
- "Best Sellers" / "Recent Releases" follow library order (there are no sales numbers), "Great Deals" is a daily shuffle.

## Starting a game (v3.8, Switch 2 UI)
- First game you start after opening the hub asks **Who's using the software?** (pick a user, or + to add one). It does not appear again until the hub tab/app is closed and reopened (a reload keeps your choice).
- Then the Switch 2 loading screen plays (fade to black, Nintendo mark, Switch 2 mark, progress line) and fades into the game once it has opened.

(v4.0) The loading screen now replays frames cut from the Switch 2 start-up video (the mark animation sprite is embedded in index.html as `MARKIMG`) on the video's own timeline: black, Nintendo mark, joy-cons join into the "2", split, rejoin, then fade into the game once it has opened (about 8 s minimum; change `MK.min` to shorten it). Sizes and corners adapt for PC and phone.

## Switch 2 home menu music (v4.1)
`public/sounds/switch2/home_menu.mp3` plays (looping) while the Switch 2 UI is on the Home screen, fades out when a game starts and back in afterwards. Turn it on/off in System Settings > Audio > Home Menu Music (or the 🔊 button). The PS5 layout keeps its own theme.
The 21 `deck_ui_*.wav` sounds from your latest zip are byte-identical to the ones already installed, so each button/screen/dialog already uses its matching sound.

(v4.2) The Switch 2 sounds and the home menu song are embedded inside `public/index.html` (`swSnd`, `swMusic`), so only index.html is needed; no sound files have to be deployed.

## VanilliaPXY option
Control Center > Proxy > "Engine" switches between PS Proxy and VanilliaPXY
(https://github.com/Portal-NetworkDev/VanilliaPXY, beta, no license stated - credit the authors).
VanilliaPXY is built into the server now: package.json depends on `vanilliapxy`, the Dockerfile uses Node 24 and runs `npm install`,
and server.js mounts `/vanillia?url=<address>` (plus /api/icon, /service-worker.js, and /ws + /wisp/ upgrades) next to PS Proxy.
Leave the server field empty to use the hub's own; enter another address to use a separate deployment.
If `npm install` fails or Node is older than 24, the hub still starts and only VanilliaPXY is off (check the logs for "VanilliaPXY loaded").

### VanilliaPXY black screen / troubleshooting
VanilliaPXY's errors are plain text and the player frame is black, so they used to look like a black screen. The hub now shows the real
message ("Target is not allowed", "Unable to reach upstream", ...) with "Open with PS Proxy" / "Reload" buttons, and the frame is white for it.
`/vx/_status` on your server tells you if VanilliaPXY loaded (needs Node 24 + `npm install`).

## Daknux games
Game library (+ tile / Library) has a third tab, "Daknux games": the catalog from daknux/assets zones.json, loaded live when you open the tab
(covers from daknux/covers, pages from daknux/html on jsDelivr). Added games are saved in your browser like the gn-math ones.

## CloudMoon (Media tab)
The CloudMoon page you sent is built in as a Media tile. It always opens inside the hub frame so the top bar has a **Proxy: OFF/ON** button.
OFF = the page talks to CloudMoon's servers directly. ON = its cross-origin fetch/XHR calls go through the selected engine (PS Proxy `/ps/…`, or VanilliaPXY if that engine is chosen).
Video/WebRTC streams and WebSockets are not proxied. The page loads Tailwind/lucide from public CDNs, so it needs a connection.

### CloudMoon popups / "invalid session"
Popups from the CloudMoon page (window.open and target=_blank links) now open as an in-hub window (Close / Open in new tab bar). jsDelivr pages are fetched and run
as part of the hub, so they share the CloudMoon page's saved session (a normal popup is a different origin and sees none of it, which gives "invalid session").

## Unity / engine games that freeze at the end of loading
The game frame now (1) falls back to plain WebAssembly instantiation when a CDN serves .wasm with the wrong content-type (the usual Unity/Emscripten stall),
(2) unzips .gz/.unityweb files a CDN sent without Content-Encoding (Brotli .br can't be fixed this way), and (3) gives the frame keyboard focus so engines don't pause themselves.
The "Game not showing up?" bar also shows the last error the game reported. "Run without tweaks" and "Load it the other way" are still there.
