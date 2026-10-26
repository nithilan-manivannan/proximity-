# proximity+

**A minimal, fast local music player with automatic lyrics and AI auto tagging. Drop in a folder of messy, untagged MP3s and proximity+ cleans the whole library up by itself.**

proximity+ identifies every track with an AI model, fetches synchronized lyrics, pulls cover art, embeds tags into the files, and then plays your library with a karaoke style lyric view. Built on Electron, React, and Express, with Groq for identification and LRCLIB for lyrics.

Preview: <https://proximity-em5e.onrender.com/>

Download page: <https://nithilan-manivannan.github.io/proximity-/>

## Why it exists

Music downloaded from the internet arrives broken on purpose it was never intended: `03 - [320KBPS] blinding_lights(1).mp3` and friends. proximity+ takes filenames as the only input it needs. The AI resolves them into real title, artist, and album triples, cover art comes from the iTunes catalog, and synced lyrics come from LRCLIB. Your files come out renamed, tagged, and cover bearing, with zero manual work.

## Features

- **AI auto tagging**: noisy filenames resolve into title, artist, and album through Groq. User confirmed values become ground truth for later files, so corrections compound.
- **Automatic synchronized lyrics**: line by line highlighting as the song plays. If LRCLIB's timestamps drift, the AI nudges them back in line based on your listening.
- **Lyrics translation**: an API endpoint translates the lyrics into another language while the original stays visible.
- **Instant search**: fuzzy search across the whole library.
- **Playlists and queue**: playlist manager, queue manager, and a history drawer.
- **Full screen player**: an immersive mode with a visualizer and equalizer, plus a compact player bar for normal use.
- **Motion design**: tilt cards and fluid transitions, built with Motion for React, in a dark industrial red on black skin. A theme hook keeps styling switchable.
- **Remote control**: playback state syncs over REST and WebSocket, so a second window or device can steer playback.
- **Fully local**: your audio never leaves the machine. Only filenames travel to the AI for identification.

## Downloads

Grab the latest installer for your platform from the releases page.

- **Windows**: NSIS installer or portable single exe, x64.
- **macOS**: dmg for Intel and Apple Silicon.

## How it is built

The app is a weird stack on purpose: a web app that is also a desktop app.

| Piece | What it is |
|---|---|
| Front end | React 19 + Vite + Tailwind CSS v4, with Motion for animation |
| Server | Express, bundled by esbuild into a single `server.cjs` |
| Shell | Electron main process (`main.cjs`) that spawns the server and points a BrowserWindow at it |
| State | The web app persists library state in IndexedDB; the server keeps port stability (see below) |
| Identification | Groq (`openai/gpt-oss-120b`), filename to metadata |
| Lyrics | LRCLIB, free and keyless, with AI powered timestamp correction and translation routes |
| Cover art | iTunes Search API |

The Electron shell serves the frontend over `http://127.0.0.1` rather than `file://`, so the front end's relative `/api/...` fetches keep working. The server listens on a fixed loopback port (37419, with deterministic fallbacks) because IndexedDB is partitioned by origin, and origin includes the port. A stable port is the difference between "remembered albums" and "vanish on restart."

## API surface

Served by the server over Express (`server.ts` in dev, `dist/server.cjs` in production):

| Route | Purpose |
|---|---|
| `GET /api/health` | Liveness check |
| `GET /api/state` | Full library state snapshot |
| `POST /api/state` | Write library state |
| `POST /api/remote` | Remote playback commands |
| `POST /api/songs/identify` | Filename to title, artist, album via AI, with learning from past corrections |
| `POST /api/lyrics/find` | LRCLIB lookup |
| `POST /api/lyrics/sync-dropped` | AI corrected timestamp lines |
| `POST /api/lyrics/translate` | Translate lyrics to another language |

## Development

```bash
git clone https://github.com/nithilan-manivannan/proximity-
cd proximity-
npm install
cp .env.example .env   # add your Groq API key(s)
npm run dev            # tsx server on :3001
```

The Vite dev server proxies `/api/*` to the backend, per `vite.config.ts`. Keys never ship to the frontend during dev.

## Building and packaging

```bash
npm run build          # build web bundle, then server bundle with keys embedded
npm run electron:dev   # build and launch the Electron shell
npm run package        # electron-builder across win + mac, no publish
npm run package:win    # NSIS installer + portable exe
npm run package:mac    # dmg, x64 + arm64
npm run lint           # tsc --noEmit
```

API keys come from `GROQ_API_KEYS` in `.env`. Multiple keys, comma separated, are pooled and rotated automatically with rate limit handling, and they get embedded in the server bundle at build time so end users never see configuration. The server bundle is made by `scripts/build-server.mjs`.

## Repository structure

```
proximity-/
├── main.cjs               # Electron main process, server spawn, port strategy
├── server.ts              # Express API: identify, lyrics, state, remote
├── src/
│   ├── App.tsx · main.tsx
│   ├── components/        # player, fullscreen, lyrics, playlists, queue,
│   │                      # visualizer, equalizer, history, uploader
│   ├── context/AudioContext.tsx
│   ├── hooks/useTheme.ts
│   └── lib/               # IndexedDB and theme helpers
├── scripts/               # build-server, icons, page screenshots, tests
├── docs/                  # landing page build and demo assets
├── build/                 # app icons
├── .env.example           # GROQ_API_KEYS
└── package.json
```

## Distribution notes

Key pooling means you can hand the built exe or dmg to friends without any setup talk. On Intel Macs the dmg is universal across both architectures, so one build serves both.
