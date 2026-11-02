# TMDB TUI — Terminal Movie/TV Browser & Player

> A keyboard-driven terminal UI for browsing TMDB (The Movie Database) and playing streams with **mpv**. Inspired by `btop`/`htop`/`claude-code` — live panels, zero mouse required.

---

## ✨ Features

| Category | Details |
|----------|---------|
| **Browse** | Search movies & TV, filter by year, direct TMDB ID lookup |
| **Navigate** | Results → Detail → Seasons → Episodes → Episode Detail |
| **Play** | One-key playback (`p`) with automatic provider fallback |
| **Providers** | 4KHDHub, Peestream, Dramachi *(extensible)* |
| **UI** | Color-coded panels, scrollable lists, contextual help bar |
| **Logging** | Debug logs written to `tmdb_tui.log` |

---

## 📦 Installation

### Prerequisites

| Tool | Install |
|------|---------|
| **Python** | 3.8+ |
| **mpv** | Required for playback |
| **curl_cffi** | `pip install curl_cffi` (for provider HTTP with browser impersonation) |
| **requests** | `pip install requests` |
| **beautifulsoup4** | `pip install beautifulsoup4` |
| **TMDB API Token** | [Get one free](https://www.themoviedb.org/settings/api) (Read Access Token) |

### Quick Start (Linux / macOS / Windows)

```bash
# 1. Clone the repo
git clone https://github.com/krouu-x/hhjklp.git
cd hhjklp

# 2. Install Python deps
pip install -r requirements.txt
# or manually:
# pip install requests curl_cffi beautifulsoup4

# 3. Set your TMDB token (required)
export TMDB_API_TOKEN="eyJhbGciOiJIUzI1NiJ9...."   # your Read Access Token

# 4. Run
python3 tmdb_tui.py
```

> **Windows (PowerShell):**
> ```powershell
> $env:TMDB_API_TOKEN = "your_token_here"
> python tmdb_tui.py
> ```

---

## 🔑 TMDB API Token

1. Create a free account at [themoviedb.org](https://www.themoviedb.org)
2. Go to **Settings → API** → *Create* → **Read Access Token (v4)**
3. Copy the token (starts with `eyJ...`)
4. Set it as an environment variable before running:

```bash
# Linux/macOS (bash/zsh)
export TMDB_API_TOKEN="eyJhbGciOiJIUzI1NiJ9...."

# Fish shell
set -x TMDB_API_TOKEN "eyJhbGciOiJIUzI1NiJ9...."

# Windows cmd
set TMDB_API_TOKEN=eyJhbGciOiJIUzI1NiJ9....

# Windows PowerShell
$env:TMDB_API_TOKEN = "eyJhbGciOiJIUzI1NiJ9...."
```

> The script includes a default token for convenience — **please use your own** to avoid rate limits and respect the API terms.

---

## ⌨️ Controls

The help bar at the bottom updates contextually. Here's the full map:

| Key | Context | Action |
|-----|---------|--------|
| **`/`** | Results | Enter search mode (type title, `Enter` to search) |
| **`y`** | Results | Enter year filter mode (type 4-digit year, `Enter`) |
| **`i`** | Results | TMDB ID lookup (type ID, `Enter`) |
| **`Tab`** | Results | Toggle ID lookup type: **Movie** ↔ **TV** |
| **`j` / `↓`** | Any list | Move selection down |
| **`k` / `↑`** | Any list | Move selection up |
| **`Enter`** | Results / Seasons / Episodes | Open selected item |
| **`s`** | Show Detail | Type season number directly |
| **`e`** | Episode List | Type episode number directly |
| **`1` / `2` / `3`** | Detail / Episodes / Ep Detail | Play via specific provider (1=4KHDHub, 2=Peestream, 3=Dramachi) |
| **`p`** | Detail / Episodes / Ep Detail | Auto-play (tries providers in order) |
| **`Esc`** | Any sub-screen | Go back one level |
| **`q` / `Q`** | Anywhere | Quit |

> **Tip:** In text-input modes (`/`, `y`, `i`, `s`, `e`), press `Esc` to cancel, `Backspace` to edit, `Enter` to submit.

---

## 🖥️ Screens Walkthrough

```
┌─────────────────────────────────────────────────────────────┐
│ TMDB TUI — search: movies + tv   |   id-lookup type: MOVIE  │
│ title: breaking bad    year: -    id: -                     │
├─────────────────────────────────────────────────────────────┤
│ Results:                                          [ 1/42 ]  │
│ █ Breaking Bad              2008  ★9.5  [TV]  id=1396       │
│   Better Call Saul          2015  ★9.0  [TV]  id=60059      │
│   El Camino: A Breaking...  2019  ★8.3  [MOVIE] id=554312   │
│   ...                                                          │
├─────────────────────────────────────────────────────────────┤
│  1 4k   2 Fastest   3 Backup   p Play (Auto)    j/k Move    │
│  Status: 3 result(s) for 'breaking bad'                      │
└─────────────────────────────────────────────────────────────┘
```

**Results → Detail → Seasons → Episodes → Episode Detail**

Each level adds context (overview, ratings, air dates) and playback options.

---

## 🎬 Providers

| Key | Name | Notes |
|-----|------|-------|
| `1` | **4KHDHub** | Multi-mirror, auto-resolves greenmotors/hubcloud/pixeldrain; parallel preflight |
| `2` | **Peestream** | Direct TMDB ID → stream API (SSE) |
| `3` | **Dramachi** | Search → season/episode → CDN links (referer-locked) |

Providers are tried in order on `p` (auto). Press `1`/`2`/`3` to force a specific one.

---

## 🛠️ Configuration

Environment variables (set before launch):

| Variable | Default | Description |
|----------|---------|-------------|
| `TMDB_API_TOKEN` | *(built-in fallback)* | Your TMDB v4 Read Access Token |
| `TMDB_BASE_URL` | `https://api.themoviedb.org/3` | TMDB API base (rarely changed) |
| `ESCDELAY` | `25` | ncurses escape-key delay (ms) — lower = snappier `Esc` |

---

## 📂 Project Structure

```
hhjklp/
├── tmdb_tui.py       # Main application (this file)
├── tmdb_tui.log      # Runtime logs (created on run)
├── requirements.txt  # Python dependencies
└── README.md         # This file
```

---

## 🐛 Troubleshooting

| Issue | Fix |
|-------|-----|
| **"Terminal too small"** | Resize to at least **50×10** (cols×rows) |
| **"mpv executable not found"** | Install `mpv` and ensure it's in `PATH` |
| **Search returns no results** | Check `TMDB_API_TOKEN` is valid; try without year filter |
| **Playback fails / black screen** | Try a different provider (`1`, `2`, `3`); check `tmdb_tui.log` |
| **Provider errors** | Mirrors rotate; wait a moment and retry. Logs show details. |
| **`ModuleNotFoundError: curl_cffi`** | `pip install curl_cffi` |

### Enable Debug Logging

```bash
# Logs are always written to tmdb_tui.log (INFO level)
# To see them live:
tail -f tmdb_tui.log
```

---

## ⚖️ Legal / Disclaimer

> **This application accesses third-party streaming sources via reverse-engineered APIs.**
>
> - **Use only for content you are legally entitled to access.**
> - The authors **do not host, store, or distribute** any copyrighted media.
> - Stream availability depends on external services and **may break at any time**.
> - This project is for **educational / personal use only**.

---

## 🤝 Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feat/awesome-thing`
3. Commit your changes: `git commit -m "Add awesome thing"`
4. Push and open a PR

Ideas welcome: new providers, UI polish, config file support, watchlist, subtitles, etc.

---

## 📄 License

MIT — see [LICENSE](LICENSE) if present, otherwise standard MIT terms apply.

---

## 🙏 Acknowledgments

- **TMDB** — for the excellent free API
- **mpv** — the best command-line media player
- **curl_cffi** — painless browser-impersonated HTTP
- **btop / htop / claude-code** — TUI design inspiration