# Nuvio iOS Bridge

A small self-hosted web app that signs into your Nuvio account and hands stream
URLs to VLC on iOS.

Nuvio has no App Store build — the iOS client must be compiled from source,
which means a Mac, Xcode, and re-signing every seven days on a free developer
account. This container is the way around that. It runs on a server you already
have, reads the addons, library and collections that are already on your Nuvio
account, and opens the stream you pick in VLC via its `vlc-x-callback://` URL
scheme. You add it to your home screen as a PWA. Nothing to sideload, nothing
to re-sign.

It is a **hand-off tool, not a player**. It finds the stream and passes it to
VLC; VLC does the playing.

**Claude coded using https://github.com/NuvioMedia/NuvioMobile as a reference!**

---

## Requirements

- Docker and Docker Compose (or Node.js 20+ to run it directly)
- An existing Nuvio account with addons installed
- VLC for iOS on the phone

---

## Quick start

```bash
cp .env.example .env          # set PUBLIC_URL to the address your phone uses
docker compose up -d --build
```

Then on the phone:

1. Open `http://<your-server>:8080`
2. Sign in with your Nuvio email and password
3. Pick a profile
4. **Share → Add to Home Screen** — it runs standalone without Safari's chrome

`PUBLIC_URL` must be the address the *phone* reaches, not `localhost`. VLC uses
it to bounce back to the app when playback ends, so a wrong value means
playback works but the return trip doesn't.

### Without Docker

```bash
npm install
PUBLIC_URL=http://192.168.1.50:8080 STATE_DIR=./data npm start
```

`npm run dev` restarts on file changes. `npm run lint` runs ESLint.

---

## Configuration

All configuration is environment variables. Only `PUBLIC_URL` really needs
setting.

### Core

| Variable | Default | What it does |
| --- | --- | --- |
| `PUBLIC_URL` | — | The address phones use to reach the container. VLC returns here after playback. |
| `PORT` | `8080` | Listen port. |
| `STATE_DIR` | `/data` | Where encrypted session and playlist records are written. |
| `SESSION_SECRET` | generated | AES-256-GCM key for stored tokens. Generate with `openssl rand -hex 32`. If unset, one is written into the data volume on first boot — set it explicitly if you want sessions to survive the volume being recreated. |
| `SESSION_IDLE_DAYS` | `60` | Idle device records are dropped after this. |
| `ALLOW_SIGNUP` | `false` | `true` offers account creation on the sign-in screen. |

### Backend

Nuvio's publishable key is built into `src/nuvio.js`, exactly as it is in the
official clients, so **none of these need setting** for the hosted backend.

| Variable | Default | What it does |
| --- | --- | --- |
| `NUVIO_BACKEND_URL` | `https://api.nuvio.tv` | Backend origin. |
| `NUVIO_ANON_KEY` | built in | Only to override the key for a different backend. |
| `NUVIO_SERVER` | — | Hostname of a self-hosted Nuvio server. The container reads the backend URL and key from `/.well-known/nuvio` at startup. |

### Playback and binge

| Variable | Default | What it does |
| --- | --- | --- |
| `BINGE_EPISODE_LIMIT` | `60` | Max episodes queued into a binge playlist. |
| `PLAYLIST_TTL_HOURS` | `24` | How long a playlist URL stays valid. |
| `STREAM_PROBE` | `head` | `head`, `off`, or `full`. See below. |
| `STREAM_PROBE_LIMIT` | `3` | Candidates checked per episode. |
| `STREAM_PROBE_TIMEOUT_MS` | `6000` | Per-probe timeout. |

> **Don't use `STREAM_PROBE=full` with debrid providers that issue single-use
> links.** `full` adds a one-byte range GET when HEAD is refused, and that
> request can consume the link before VLC ever plays it.

### Tuning

| Variable | Default | What it does |
| --- | --- | --- |
| `HOME_ROW_LIMIT` | `10` | Catalog rows fetched for the home screen. |
| `HTTP_TIMEOUT_MS` | `15000` | Timeout for addon and backend calls. |
| `CACHE_TTL_MS` | `300000` | Addon manifest cache lifetime. |

---

## How it works

### Sign-in and sessions

Nuvio's backend is Supabase. Sign-in goes through GoTrue
(`POST /auth/v1/token?grant_type=password`); addons come from a Postgres table;
library, collections and home-row settings come from RPCs like
`sync_pull_library`.

The store separates two things:

- **Devices** — one per signed-in browser, holding that browser's Supabase
  tokens, addressed by an opaque cookie id. A phone, a laptop and an iPad are
  three devices on one account, all valid at once; refreshing one doesn't
  disturb the others.
- **Accounts** — one per Nuvio user, holding what should follow you around:
  profile selection, theme, playback preferences, shortcut token.

Sign-out is scoped `?scope=local` and ends that device only.
`POST /api/logout` with `{"everywhere": true}` ends all of them.

Records are encrypted at rest with AES-256-GCM in `STATE_DIR`. Access tokens
refresh a minute before expiry and no token is ever sent to the browser. Failed
sign-ins are rate limited to ten per IP per fifteen minutes.

### Playback preferences

The gear icon on any title opens a Playback sheet, stored per user:

- **Auto-play best stream** — Play hands the top-ranked playable result
  straight to VLC instead of opening the picker.
- **Binge next episode** — VLC gets the whole season as an M3U playlist, so it
  has a real queue with working next/previous.
- **Preferred quality** — Any, 4K, 1080p or 720p. Falls back to the top-ranked
  stream when nothing matches.
- **Audio / Subtitle language** — floats matching release names up the list
  (a hint, never a filter), emits `#EXTVLCOPT:audio-language` /
  `sub-language` in ISO 639-2/B, and attaches a matching subtitle file with
  `input-slave` if a subtitle addon is installed.

Setting a language preference routes even a single movie through a one-entry
playlist, since `vlc-x-callback` has no parameters for track languages and the
M3U is the only channel that carries `#EXTVLCOPT`.

### Binge playlists

Starting an episode with binge on creates a record and hands VLC
`/playlist/<token>.m3u`. VLC fetches it and pages through the season itself —
no round trip out to Safari between episodes.

The playlist is the **whole season**, not just what's ahead: your episode
leads, the rest of the season follows, earlier episodes are appended at the
end. VLC always starts an M3U at track one and has no start-index parameter,
so leading with your episode is the only way to start where you asked; keeping
the earlier ones on the end means you can still reach them from VLC's playlist
view.

Streams are resolved when VLC fetches the playlist, not when it's created, so
nothing is spent on episodes never watched. Each candidate gets a HEAD check
and only a *definite* refusal moves on — 403, 405 and 501 mean the method was
refused, and a timeout means the check failed, not the link. If every candidate
looks dead the best one goes in anyway, since a doubtful link beats a missing
episode.

Records expire after `PLAYLIST_TTL_HOURS` and are encrypted at rest, because
resolved debrid URLs are worth protecting.

### What is not tracked

Playing something writes nothing to your account — no Continue Watching entry,
nothing marked watched, no progress touched on other devices. VLC's x-callback
interface takes a URL and returns control; that's the whole contract, so
anything written would be a guess.

The one exception is explicit: **Add to Library** on a title's page, which
calls `sync_push_library_items`. It saves the show, not the episode, and shows
up in the native app like any other library entry.

---

## Limitations

**VLC opens a URL; it cannot open a torrent.** Addons that return a bare
`infoHash` — Torrentio and friends in their default configuration — appear in
the stream list dimmed and labelled, because there is nothing to hand over.
Point those addons at a debrid service (Real-Debrid, TorBox, AllDebrid) so they
return HTTPS links. The native iOS build has the same constraint; it just hides
it behind a bundled player.

Non-addon sources inside a collection folder (Trakt lists, TMDB ids) are
skipped — those need provider credentials this bridge doesn't hold.

---

## Troubleshooting

**Binge skips episodes.** Open `/playlist/<token>/report` in a browser — the
token is in the playlist URL. It lists every episode the playlist was built
from, which addon supplied the stream, how many candidates there were, what the
probe said, and the reason for anything missing.

The remaining case that genuinely drops an episode is no addon returning an
HTTP link for it at all. A debrid addon fixes that.

**VLC opens but nothing returns to the app.** `PUBLIC_URL` is wrong or not
reachable from the phone.

**A page hangs for ~15 seconds.** A dead addon. Calls fan out in parallel and
failures are dropped rather than propagated, so it costs one timeout rather
than a broken page — but the timeout is `HTTP_TIMEOUT_MS`.

**Signing in on the phone signed me out on the laptop.** It shouldn't; devices
are stored separately and sign-out is scoped local. If it happens, something
else called GoTrue with global scope on that account.

`GET /healthz` returns `{ ok, accounts, devices, playlists }`.

---

## HTTP endpoints

Everything under `/api` except login/signup/session requires a device cookie.

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/login`, `/api/signup`, `/api/logout` | Auth. Login and signup are rate limited. |
| `GET` | `/api/session` | Current device and account state. |
| `GET` | `/api/profiles` · `POST` `/api/profile` | Nuvio profile picker. |
| `POST` | `/api/theme`, `/api/prefs` | Per-account theme and playback settings. |
| `POST` | `/api/shortcut-token` | Generate or rotate the Shortcut token. |
| `GET` | `/api/home`, `/api/catalogs`, `/api/catalog` | Home rows and paged catalogs. |
| `GET` | `/api/collections`, `/api/collection/:id/:folderId` | Collections and folder contents. |
| `GET` | `/api/library` · `POST` `/api/library` | Library read, add, remove. |
| `GET` | `/api/search`, `/api/meta/:type/:id`, `/api/streams/:type/:id` | Addon protocol. |
| `GET` | `/play`, `/play/auto/:type/:id` | Hand-off to VLC. |
| `GET` | `/playlist/:token.m3u` | Binge playlist, resolved on fetch. |
| `GET` | `/playlist/:token/report` | Why an episode is missing. |
| `GET` | `/api/shortcut/play?q=` | Shortcut endpoint. `&format=json` for structured output. |
| `GET` | `/healthz` | Health check. |

---

## Project layout

```
src/nuvio.js      Supabase auth, profiles, addons table, library and sync RPCs
src/sessions.js   per-user session and account store, encrypted at rest
src/playlist.js   binge playlists — M3U records, resolved on demand
src/languages.js  language codes, subtitle matching, release-name hints
src/addons.js     Stremio addon protocol — manifests, catalogs, search, meta, streams
src/vlc.js        vlc-x-callback:// and vlc:// URL builders
src/server.js     routes, session handling, token refresh, stream ranking, probing
public/           the phone UI (vanilla JS, no build step)
scripts/          fetch-fonts.sh
```

Nuvio's `addons` table stores only a URL per addon, not a cached manifest, so
manifests are fetched on first use and cached for `CACHE_TTL_MS`.

---

## Addon protocol notes

The addon layer follows what the Nuvio clients do, which matters for addons
that stray from the simple case:

- **Extras are a path segment in a fixed order** — `search`, then `genre`, then
  `skip`, joined with `&`. Sending them as a query string, or reordered, makes
  some addons silently return the wrong page.
- **Both manifest dialects parse** — newer `extra: [{name, isRequired,
  options}]` and older `extraSupported` / `extraRequired` + `genres`.
- **Resource-level types and idPrefixes are respected**, so a Kitsu addon is
  never asked about `tt` ids. Beyond correctness this is a speed fix.
- **Catalogs with a required genre** get one supplied automatically; ones
  requiring some other extra are kept out of home rows but stay browsable.
- **Addons flagged `configurationRequired`** are dropped rather than queried.
- **Any type works** — anime, tv, channels, whatever an addon declares.
- **Row order** comes from `sync_pull_home_catalog_settings`, so disabled
  catalogs stay hidden and ordering matches the app.

---

## Notes

Self-hosted and single-purpose: it talks to your own Nuvio account and the
addons you installed there. It hosts no content and ships no addons.

# Fully Coded with Claude AI
