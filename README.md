# STUB

A simple keyboard-driven TUI for browsing movies and TV shows with TMDB and playing streams with `mpv`.

## Requirements

- Python 3
- `requests`
- `mpv` (required for playback)
- A TMDB API Read Access Token

## Install

### Windows

Install Python from https://www.python.org/downloads/

Install mpv:
- Download it from https://mpv.io/installation/
- Extract it somewhere, then add the folder containing `mpv.exe` to your PATH.

Then:

```powershell
pip install requests
python tmdb_tui.py
```

### Linux

Install Python, pip and mpv:

```bash
# Debian / Ubuntu
sudo apt install python3 python3-pip mpv

# Fedora
sudo dnf install python3 python3-pip mpv

# Arch
sudo pacman -S python python-pip mpv
```

Then:

```bash
pip install requests
python3 tmdb_tui.py
```

### macOS

Install Homebrew if you don't already have it, then:

```bash
brew install python mpv
pip3 install requests
python3 tmdb_tui.py
```

## TMDB Token

This app already has tmdb token pls dont misuse it or spam or use it for your projects, if u want u can generate a free token ur self from tmdb website.

### Linux / macOS

```bash
export TMDB_API_TOKEN="your_token_here"
```

### Windows PowerShell

```powershell
$env:TMDB_API_TOKEN="your_token_here"
```

The code should read the token from the environment instead of hard-coding it.

## Controls

```text
/       Search
y       Filter by year
i       TMDB ID lookup
Tab     Switch Movie/TV for ID lookup
j / k   Move up/down
Enter   Select
Esc     Go back
s       Select season
e       Jump to episode
p       Play
q       Quit
```

The UI is intentionally simple, so the controls are shown at the bottom of the terminal too.

## Running

```bash
python3 tmdb_tui.py
```

On Windows:

```powershell
python tmdb_tui.py
```

Make sure your terminal is large enough for the interface.

## Notes

- TMDB is used for movie/TV information.
- `mpv` is only needed for playback.
- Playback depends on external stream APIs and may not always work.
- Use the application only with content and services you are legally allowed to access.
