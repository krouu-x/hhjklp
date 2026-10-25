#!/usr/bin/env python3
"""
tmdb_tui.py — an interactive terminal UI for browsing TMDB (The Movie Database),
in the spirit of btop/htop/claude-code's TUI: live panels, keyboard-driven.

NOTE: set your TMDB API token before running:
    export TMDB_API_TOKEN="your_token_here"
"""
import os
os.environ.setdefault("ESCDELAY", "25")
import curses
import locale
import logging
import threading
import time
import textwrap
import subprocess
import json
import re
import base64
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from difflib import SequenceMatcher
from typing import Callable, List, Tuple, Dict, Optional
from urllib.parse import urlparse, parse_qs, quote

from curl_cffi import requests
from bs4 import BeautifulSoup

log = logging.getLogger("tmdb_tui")

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------

TMDB_API_TOKEN = os.environ.get(
    "TMDB_API_TOKEN",
    "eyJhbGciOiJIUzI1NiJ9.eyJhdWQiOiI1YjFjM2M4ZDg3MTU1MWU2NDg0N2FlNGQ5NTRlMDgwNyIsIm5iZiI6MTc4NjU2MTM5Ni43OTMsInN1YiI6IjZhN2NjMzc0ZmI3YTFjMjkwMTQ0ZmRlYSIsInNjb3BlcyI6WyJhcGlfcmVhZCJdLCJ2ZXJzaW9uIjoxfQ.heOiSMRVHPZiCerBZK-91bqtqclEDjCbnzbYpFQo56w",
)
TMDB_BASE_URL = os.environ.get("TMDB_BASE_URL", "https://api.themoviedb.org/3")

HEADERS = {
    "Authorization": f"Bearer {TMDB_API_TOKEN}",
    "Accept": "application/json",
}

MODE_NORMAL = "normal"
MODE_SEARCH = "search"
MODE_YEAR = "year"
MODE_ID = "id"
MODE_SEASON = "season"
MODE_EPISODE = "episode"

SCREEN_RESULTS = "results"
SCREEN_DETAIL = "detail"
SCREEN_EPISODES = "episodes"
SCREEN_EPISODE_DETAIL = "episode_detail"

MIN_WIDTH = 50
MIN_HEIGHT = 10

BROWSER_HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "Accept-Language": "en-US,en;q=0.9",
    "Accept-Encoding": "gzip, deflate, br",
    "Connection": "keep-alive",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "none",
    "Sec-Fetch-User": "?1",
}

# Provider 4 tuning knobs
PAGE_TIMEOUT = 12           # seconds, per page fetch
MIRROR_WORKERS = 4         # bounded parallelism for mirror resolution
MAX_MIRRORS_PER_RELEASE = 6
SEARCH_TTL = 600           # seconds search results stay cached
SEARCH_CACHE_MAX = 64      # cache bound


@dataclass
class StreamProvider:
    name: str
    key: str
    # (tmdb_id, title, media_type, season_num, episode_num, year) -> [(url, headers)]
    fetch_func: Callable[..., List[Tuple[str, Dict[str, str]]]]


# =============================================================================
# 4KHDHub provider
# =============================================================================

FOURKHDHUB_DOMAINS = [
    "https://4khdhub.link",
    "https://4khdhub.dad",
    "https://4khdhub.one",
    "https://4khub.com",
]

_SEARCH_CACHE: Dict[str, Tuple[float, List[Dict]]] = {}
_SEARCH_CACHE_LOCK = threading.Lock()
_TLS = threading.local()


def _4k_session() -> requests.Session:
    """One impersonated session per calling thread (curl_cffi sessions are not
    documented as thread-safe, so workers each get their own)."""
    s = getattr(_TLS, "session", None)
    if s is None:
        s = requests.Session(impersonate="chrome120")
        s.headers.update(BROWSER_HEADERS)
        _TLS.session = s
    return s


def _norm_title(t: str) -> str:
    """Lowercase, strip (year)/Season N/punctuation for comparison."""
    t = t.lower()
    t = re.sub(r"\((?:19|20)\d{2}\)", " ", t)
    t = re.sub(r"\bseason\s*\d+\b", " ", t)
    t = re.sub(r"[^a-z0-9]+", " ", t)
    return " ".join(t.split())


def _4khdhub_search(query: str) -> List[Dict]:
    key = _norm_title(query)
    now = time.time()

    with _SEARCH_CACHE_LOCK:
        hit = _SEARCH_CACHE.get(key)
        if hit and now - hit[0] < SEARCH_TTL:
            log.info("search cache hit: %r", query)
            return hit[1]

    session = _4k_session()
    last_err = None
    items: List[Dict] = []
    seen_paths = set()

    for domain in FOURKHDHUB_DOMAINS:
        url = f"{domain}/?s={quote(query)}"
        try:
            resp = session.get(url, timeout=PAGE_TIMEOUT)
            if resp.status_code != 200:
                log.warning("search %s -> HTTP %s", domain, resp.status_code)
                continue

            soup = BeautifulSoup(resp.text, "html.parser")
            cards = soup.select("a.movie-card") or soup.select(
                ".post-title a, .entry-title a, article a, .movie-item a"
            )

            for card in cards:
                href = card.get("href")
                if not href:
                    continue

                title_el = card.select_one(".movie-card-title, .title, h2, h3")
                title = title_el.get_text(strip=True) if title_el else card.get_text(strip=True)
                if not title or len(title) < 2:
                    continue

                meta_el = card.select_one(".movie-card-meta, .meta")
                meta = meta_el.get_text(strip=True) if meta_el else title
                year_match = re.search(r"\b(19|20)\d{2}\b", meta)
                year = year_match.group(0) if year_match else None

                media_type = (
                    "Series"
                    if any(k in href.lower() for k in ["-series-", "/series/", "season"])
                    else "Movie"
                )
                path = urlparse(href).path
                if path in seen_paths:  # cross-domain duplicates
                    continue
                seen_paths.add(path)

                items.append({
                    "id": path,
                    "title": title,
                    "media_type": media_type,
                    "year": year,
                    "base_domain": domain,
                })

            if items:
                log.info("search %r: %d hits via %s", query, len(items), domain)
                break

        except Exception as e:
            last_err = e
            log.warning("search %s failed: %s", domain, e)
            continue

    if not items:
        raise Exception(
            f"No 4KHDHub search results (mirrors down or blocked). Last error: {last_err}"
        )

    with _SEARCH_CACHE_LOCK:
        if len(_SEARCH_CACHE) >= SEARCH_CACHE_MAX:
            _SEARCH_CACHE.clear()
        _SEARCH_CACHE[key] = (time.time(), items)
    return items


def _4khdhub_pick_match(
    results: List[Dict], title: str, year: str, media_type: str
) -> Tuple[Dict, float]:
    """Rank candidates by title similarity, media type, and release year.
    Returns (best, score). Logs a warning when the top two scores are close."""
    want = _norm_title(title)
    target_type = "Series" if media_type == "tv" else "Movie"

    def score(r: Dict) -> float:
        s = SequenceMatcher(None, want, _norm_title(r.get("title", ""))).ratio() * 10.0
        s += 3.0 if r.get("media_type") == target_type else -2.0
        ry, wy = r.get("year"), (year or None)
        if wy and ry:
            s += 4.0 if ry == wy else -3.0
        return s

    ranked = sorted(results, key=score, reverse=True)
    best = ranked[0]
    best_score = score(best)
    if len(ranked) > 1 and score(ranked[1]) >= best_score - 0.5:
        log.warning(
            "ambiguous match for %r: %r (%.1f) vs %r (%.1f)",
            title, ranked[0]["title"], best_score, ranked[1]["title"], score(ranked[1]),
        )
    return best, best_score


def _4khdhub_get_releases(
    domain: str, id_path: str, season: int = 0, episode: int = 0
) -> List[Dict]:
    url = f"{domain}{id_path}"
    session = _4k_session()
    resp = session.get(url, timeout=PAGE_TIMEOUT)
    resp.raise_for_status()
    soup = BeautifulSoup(resp.text, "html.parser")

    if season > 0:
        items = soup.select("#episodes .episode-download-item")
        title_sel = ".episode-file-title"
    else:
        items = soup.select(".download-item")
        title_sel = ".file-title"

    releases = []
    for item in items:
        title_el = item.select_one(title_sel)
        filename = title_el.get_text(strip=True) if title_el else ""
        if not filename or filename.lower().endswith(".zip"):
            continue

        if season > 0:
            m = re.search(r"S(\d+)E(\d+)", filename, re.IGNORECASE)
            if not m:
                continue
            if int(m.group(1)) != season or int(m.group(2)) != episode:
                continue

        mirrors = []
        for link in item.select("a[href]"):
            href = link.get("href", "")
            if not href.startswith("https://") or "logout" in href:
                continue
            label = link.get_text(strip=True) or "Source"
            mirrors.append({"url": href, "label": label})

        if mirrors:
            releases.append({"filename": filename, "mirrors": mirrors})

    return releases


def _4khdhub_decode_greenmotors(html: str) -> Optional[str]:
    match = re.search(r"s\(\s*'o'\s*,\s*'([^']+)'\s*", html)
    if not match:
        return None

    payload = match.group(1)
    try:
        step1 = base64.b64decode(payload).decode("utf-8")
        step2 = base64.b64decode(step1).decode("utf-8")
        step3 = step2.translate(str.maketrans(
            "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz",
            "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm"
        ))
        step4 = base64.b64decode(step3).decode("utf-8")
        data = json.loads(step4)

        target_b64 = data.get("o") or data.get("url") or data.get("link")
        if target_b64:
            return base64.b64decode(target_b64).decode("utf-8")
    except Exception:
        pass
    return None


def _4khdhub_resolve_mirror(mirror_url: str, session: requests.Session) -> Optional[str]:
    try:
        # Step 1: Intermediate greenmotors / greenmountmotors mediators
        if any(d in mirror_url for d in ["greenmotors", "greenmountmotors"]):
            resp = session.get(mirror_url, timeout=10)
            target = _4khdhub_decode_greenmotors(resp.text)
            if target:
                return _4khdhub_resolve_mirror(target, session)
            return None

        # Step 2: HubDrive route -> redirect to HubCloud
        elif "hubdrive." in mirror_url:
            resp = session.get(mirror_url, timeout=10)
            soup = BeautifulSoup(resp.text, "html.parser")
            for a in soup.select("a[href]"):
                href = a.get("href", "")
                if "hubcloud." in href:
                    return _4khdhub_resolve_mirror(href, session)

        # Step 3: HubCloud extraction
        elif "hubcloud." in mirror_url:
            resp = session.get(mirror_url, timeout=10)
            soup = BeautifulSoup(resp.text, "html.parser")

            resolver_url = None
            for a in soup.find_all("a", href=True):
                href = a["href"]
                if "/download/" in href or "download" in a.get("id", "") or "btn" in " ".join(a.get("class", [])):
                    if href.startswith("https://"):
                        resolver_url = href
                        break

            if not resolver_url:
                return None

            resp2 = session.get(resolver_url, timeout=10)
            soup2 = BeautifulSoup(resp2.text, "html.parser")

            # Handle pages.dev Base64 redirect targets
            for a in soup2.select('a[href*="pages.dev"]'):
                parsed = urlparse(a["href"])
                qs = parse_qs(parsed.query)
                if "u" in qs:
                    try:
                        decoded = base64.b64decode(qs["u"][0]).decode("utf-8")
                        if decoded.startswith("https://"):
                            return decoded.replace("pixeldrain.dev", "pixeldrain.com")
                    except Exception:
                        pass

            # Fallback regex extraction for Pixeldrain / raw direct storage links
            text = resp2.text
            m = re.search(r"https?://(?:pixeldrain\.(?:dev|com)/(?:u|api/file)/)[^\s\"'<>]+", text)
            if m:
                pd_url = m.group(0).replace("pixeldrain.dev", "pixeldrain.com")
                if "/u/" in pd_url:
                    pd_url = pd_url.replace("/u/", "/api/file/") + "?download"
                elif "/api/file/" in pd_url and "?download" not in pd_url:
                    pd_url += "?download"
                return pd_url

            return None
        else:
            return mirror_url
    except Exception:
        return None


def _4khdhub_preflight(url: str, session: requests.Session) -> Optional[str]:
    if not url or not url.startswith("https://"):
        return None

    parsed = urlparse(url)
    if parsed.hostname in ("localhost", "127.0.0.1") or url.endswith(".zip"):
        return None

    try:
        resp = session.get(url, timeout=10, stream=True)
        try:
            if resp.status_code < 400:
                ct = resp.headers.get("Content-Type", "").lower()
                if "text/html" in ct or "text/plain" in ct:
                    return None
                return resp.url
            return None
        finally:
            resp.close()  # stream=True responses must be closed explicitly
    except Exception:
        return None


def _4khdhub_try_mirror(mirror_url: str) -> Optional[str]:
    """Resolve + preflight one mirror. Runs in a worker thread; never raises."""
    try:
        resolved = _4khdhub_resolve_mirror(mirror_url, _4k_session())
        if not resolved:
            log.info("resolve failed: %s", mirror_url)
            return None
        playable = _4khdhub_preflight(resolved, _4k_session())
        if playable:
            log.info("playable: %s", playable)
        return playable
    except Exception as e:
        log.warning("mirror %s failed: %s", mirror_url, e)
        return None


def _fetch_fourkhdhub(
    tmdb_id: str, title: str, media_type: str,
    season_num: str, episode_num: str, year: str = "",
) -> List[Tuple[str, Dict[str, str]]]:
    started = time.perf_counter()
    log.info("fetch title=%r type=%s year=%s", title, media_type, year)

    results = _4khdhub_search(title)
    match, score = _4khdhub_pick_match(results, title, year, media_type)
    log.info("selected %r (%r) score=%.1f", match["title"], match["base_domain"], score)

    season = int(season_num) if media_type == "tv" and season_num else 0
    episode = int(episode_num) if media_type == "tv" and episode_num else 0

    releases = _4khdhub_get_releases(match["base_domain"], match["id"], season, episode)
    if not releases:
        raise Exception("No 4KHDHub releases found")
    log.info("%d releases for S%sE%s", len(releases), season or "-", episode or "-")

    # Flatten to unique mirror jobs, bounded per release.
    jobs, seen = [], set()
    for release in releases:
        for mirror in release["mirrors"][:MAX_MIRRORS_PER_RELEASE]:
            if mirror["url"] not in seen:
                seen.add(mirror["url"])
                jobs.append(mirror["url"])

    candidate_streams: List[Tuple[str, Dict[str, str]]] = []
    if jobs:
        with ThreadPoolExecutor(max_workers=min(MIRROR_WORKERS, len(jobs))) as ex:
            futures = {ex.submit(_4khdhub_try_mirror, u): u for u in jobs}
            for fut in as_completed(futures):
                playable = fut.result()
                if playable:
                    candidate_streams.append((
                        playable,
                        {"User-Agent": BROWSER_HEADERS["User-Agent"]},
                    ))

    elapsed = time.perf_counter() - started
    log.info("fetch done: %d playable stream(s) in %.2fs", len(candidate_streams), elapsed)

    if not candidate_streams:
        raise Exception("No playable 4KHDHub streams resolved (see tmdb_tui.log)")
    return candidate_streams


# =============================================================================
# Dramachi provider
# =============================================================================

def _fetch_dramachi(
    tmdb_id: str, title: str, media_type: str,
    season_num: str, episode_num: str, year: str = "",
) -> List[Tuple[str, Dict[str, str]]]:
    base_url = "https://api.nodeobjects.com/"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"}

    queries_to_try = [title]
    no_year = re.sub(r"\s*\(\d{4}\)\s*$", "", title).strip()
    if no_year != title:
        queries_to_try.append(no_year)
    cleaned = re.sub(r"[:\-]", " ", title).strip()
    if cleaned != title:
        queries_to_try.append(cleaned)
    if ":" in title:
        main_title = title.split(":")[0].strip()
        if main_title != title:
            queries_to_try.append(main_title)

    items = []
    for query in queries_to_try:
        encoded_q = quote(query, safe="")
        search_url = f"{base_url}?interface=search&q={encoded_q}&filter=all&page=1"
        resp = requests.get(search_url, headers=headers, timeout=15)
        resp.raise_for_status()
        data = resp.json()
        items = data.get("data") or []
        if items:
            break

    if not items:
        raise Exception(f"No Dramachi search results found for '{title}'")

    is_movie_target = (media_type == "movie")
    match = None
    for item in items:
        content = (item.get("content") or "").lower()
        is_movie_result = content in ["movies", "movie"]
        if is_movie_result == is_movie_target:
            match = item
            break

    if not match:
        match = items[0]

    title_id = str(match.get("id", "")).strip()
    if not title_id:
        raise Exception("Dramachi match missing ID")

    rip_to_query = "hd Rip"
    if media_type == "tv" and season_num:
        try:
            season_int = int(season_num)
            season_key = f"Season {season_int:02d}"
        except ValueError:
            season_key = "Season 01"

        details_url = f"{base_url}?interface=title_v2&id={quote(title_id)}"
        det_resp = requests.get(details_url, headers=headers, timeout=15)
        det_resp.raise_for_status()
        det_data = det_resp.json()

        seasons = det_data.get("seasons") or {}
        target_rip = None

        if season_key in seasons:
            versions = seasons[season_key].get("versions") or []
            if versions:
                target_rip = versions[0].get("rip")

        if not target_rip:
            for s_name, s_data in seasons.items():
                versions = s_data.get("versions") or []
                if versions:
                    target_rip = versions[0].get("rip")
                    break

        if not target_rip:
            target_rip = season_key
        rip_to_query = target_rip

    eplist_url = f"{base_url}?interface=eplist&season={quote(rip_to_query)}&id={quote(title_id)}"
    ep_resp = requests.get(eplist_url, headers=headers, timeout=15)
    ep_resp.raise_for_status()
    ep_data = ep_resp.json()
    episodes = ep_data.get("episode_list") or []

    if not episodes:
        raise Exception(f"No episodes found for Dramachi RIP: {rip_to_query}")

    def parse_ep_num(f_title: str) -> int:
        m = re.search(r"E(\d+)", f_title, re.IGNORECASE)
        if m:
            return int(m.group(1))
        tokens = f_title.split()
        if tokens:
            try:
                return int(tokens[-1])
            except ValueError:
                pass
        return 9999

    episodes.sort(key=lambda ep: parse_ep_num(ep.get("f_title", "")))

    target_ep = int(episode_num) if episode_num else 0
    if target_ep > 0:
        matched_episodes = [ep for ep in episodes if parse_ep_num(ep.get("f_title", "")) == target_ep]
        if not matched_episodes and target_ep <= len(episodes):
            matched_episodes = [episodes[target_ep - 1]]
    else:
        matched_episodes = episodes

    if not matched_episodes:
        raise Exception(f"Episode {episode_num} not found in Dramachi")

    candidate_streams = []
    for ep in matched_episodes:
        fid = ep.get("fid")
        disk = ep.get("disk")
        if not fid or not disk:
            continue

        file_url = f"{base_url}?interface=getFile&fid={quote(str(fid))}&findex={quote(str(disk))}"
        file_resp = requests.get(file_url, headers=headers, timeout=15)
        file_resp.raise_for_status()
        file_data = file_resp.json()

        host_info = file_data.get("hostInfo")
        file_info_list = file_data.get("fileInfo") or []

        if not host_info or not file_info_list:
            continue

        host = str(host_info.get("host", "")).strip("/")
        raw_url = file_info_list[0].get("url", "")

        if host and raw_url:
            stream_url = f"https://{host}/cdn/{raw_url.lstrip('/')}"
            hdrs = {
                "Referer": "https://dramachi.com/",
                "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
            }
            candidate_streams.append((stream_url, hdrs))

    if not candidate_streams:
        raise Exception("No playable streams resolved from Dramachi")

    return candidate_streams


# =============================================================================
# Other providers
# =============================================================================

def _fetch_moviebox(
    tmdb_id: str, title: str, media_type: str,
    season_num: str, episode_num: str, year: str = "",
) -> List[Tuple[str, Dict[str, str]]]:
    try:
        import moviebox_tui
    except ImportError as e:
        raise Exception(f"Rust provider not installed: {e}. Run 'maturin develop --release --features python'")

    if not hasattr(moviebox_tui, "search_moviebox") or not hasattr(moviebox_tui, "get_moviebox_streams"):
        raise AttributeError("moviebox_tui module is missing required functions")

    try:
        search_res_json = moviebox_tui.search_moviebox(title)
    except Exception as e:
        raise Exception(f"MovieBox search call failed: {e}")

    try:
        search_results = json.loads(search_res_json)
    except Exception as e:
        raise Exception(f"MovieBox search returned invalid JSON: {e}")

    if not isinstance(search_results, list) or not search_results:
        raise Exception("No matching MovieBox result found")

    target_type = "tv" if media_type == "tv" else "movie"
    match = next(
        (r for r in search_results
         if isinstance(r, dict) and r.get("media_type") == target_type and r.get("id")),
        None,
    )
    if not match:
        match = next((r for r in search_results if isinstance(r, dict) and r.get("id")), None)

    if not match or not match.get("id"):
        raise Exception("No matching MovieBox result found with a valid id")

    subject_id = str(match["id"]).strip()
    season_val = int(season_num) if media_type == "tv" and season_num else None
    episode_val = int(episode_num) if media_type == "tv" and episode_num else None

    try:
        streams_json = moviebox_tui.get_moviebox_streams(
            subject_id=subject_id, season=season_val, episode=episode_val
        )
    except Exception as e:
        raise Exception(f"MovieBox get_streams call failed: {e}")

    try:
        streams = json.loads(streams_json)
    except Exception as e:
        raise Exception(f"MovieBox streams returned invalid JSON: {e}")

    candidate_streams = []
    for s in streams:
        if not isinstance(s, dict):
            continue
        url = s.get("url")
        if not url or not isinstance(url, str) or not url.strip():
            continue

        raw_headers = s.get("headers")
        hdr_dict: Dict[str, str] = {}
        if isinstance(raw_headers, dict):
            hdr_dict = {str(k): str(v) for k, v in raw_headers.items()}
        elif isinstance(raw_headers, list):
            for item in raw_headers:
                if isinstance(item, (list, tuple)) and len(item) == 2:
                    hdr_dict[str(item[0])] = str(item[1])
                elif isinstance(item, dict):
                    for k, v in item.items():
                        hdr_dict[str(k)] = str(v)
        candidate_streams.append((url.strip(), hdr_dict))

    if not candidate_streams:
        raise Exception("No valid streams returned by MovieBox")
    return candidate_streams


def _fetch_vidlove(
    tmdb_id: str, title: str, media_type: str,
    season_num: str, episode_num: str, year: str = "",
) -> List[Tuple[str, Dict[str, str]]]:
    endpoint = "tv" if media_type == "tv" else "movie"
    url = f"https://api.vidlove.cc/{endpoint}"
    headers = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36"}
    sources = ["megaknight", "vidapi", "cinefreak", "moviebox2"]

    for source in sources:
        params = {"id": tmdb_id, "mode": "json", "sources": source, "hevc": 1}
        if media_type == "tv":
            params["season"] = season_num
            params["episode"] = episode_num

        r = requests.get(url, headers=headers, params=params, timeout=15)
        r.raise_for_status()
        data = r.json()

        source_data = data.get("source")
        if source_data and isinstance(source_data, dict):
            stream_url = source_data.get("url")
            if stream_url:
                stream_headers = source_data.get(
                    "headers", {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
                )
                return [(stream_url, stream_headers)]
    raise Exception("All Vidlove sources failed")



def _fetch_peestream(
    tmdb_id: str,
    title: str,
    media_type: str,
    season_num: str,
    episode_num: str,
    year: str = "",
) -> List[Tuple[str, Dict[str, str]]]:

    url = "https://api.peestream.in/api/search/stream"
    params = {
        "q": tmdb_id,
        "type": media_type,
    }

    if media_type == "tv":
        params["season"] = season_num
        params["episode"] = episode_num

    response = requests.get(url, params=params, timeout=15)
    response.raise_for_status()

    # Parse SSE events and find the final JSON payload.
    data = None
    event_name = None

    for line in response.text.splitlines():
        if line.startswith("event:"):
            event_name = line[6:].strip()

        elif line.startswith("data:") and event_name == "done":
            data = json.loads(line[5:].strip())
            break

    if data is None:
        raise Exception("Peestream returned no done event")

    candidate_streams = []

    for result in data.get("results", []):
        for stream in result.get("streams", []):
            stream_url = stream.get("url")

            if stream_url:
                candidate_streams.append(
                    (stream_url, stream.get("headers", {}))
                )

    if not candidate_streams:
        raise Exception(
            f"No Peestream results found for {tmdb_id}"
        )

    return candidate_streams



# =============================================================================
# Provider configuration
# =============================================================================

PROVIDERS = [
    StreamProvider(name="4k", key="1", fetch_func=_fetch_fourkhdhub),
    StreamProvider(name="Fastest", key="2", fetch_func= _fetch_peestream),
    StreamProvider(name="Backup", key="3", fetch_func=_fetch_dramachi),
]


# ---------------------------------------------------------------------------
# TMDB API client
# ---------------------------------------------------------------------------

class TMDBClient:
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update(HEADERS)

    def _get_json(self, url, params=None):
        try:
            resp = self.session.get(url, params=params, timeout=10)
        except requests.exceptions.Timeout:
            return None, "Request timed out."
        except requests.exceptions.RequestException as e:
            return None, f"Network error: {e}"

        if resp.status_code != 200:
            return None, f"HTTP {resp.status_code}"

        try:
            return resp.json(), None
        except ValueError:
            return None, "Malformed JSON response."

    def search_multi(self, title: str, year: str = ""):
        data, err = self._get_json(f"{TMDB_BASE_URL}/search/multi", params={"query": title})
        if err:
            return [], err

        results = [r for r in data.get("results", []) if r.get("media_type") in ("movie", "tv")]
        if year:
            results = [
                r for r in results
                if (r.get("release_date") or r.get("first_air_date") or "")[:4] == year
            ]
        return results, None

    def get_by_id(self, media_type: str, tmdb_id):
        endpoint = "movie" if media_type == "movie" else "tv"
        return self._get_json(f"{TMDB_BASE_URL}/{endpoint}/{tmdb_id}")

    def get_season(self, tv_id, season_number):
        return self._get_json(f"{TMDB_BASE_URL}/tv/{tv_id}/season/{season_number}")


def safe_addstr(win, y, x, text, attr=0):
    h, w = win.getmaxyx()
    if 0 <= y < h and 0 <= x < w:
        try:
            win.addstr(y, x, text[: max(0, w - x - 1)], attr)
        except curses.error:
            pass


# ---------------------------------------------------------------------------
# App
# ---------------------------------------------------------------------------

class App:
    def __init__(self, stdscr):
        self.stdscr = stdscr
        self.client = TMDBClient()
        self.id_lookup_type = "movie"

        self.title_query = ""
        self.year_query = ""
        self.id_query = ""

        self.screen = SCREEN_RESULTS
        self.mode = MODE_NORMAL
        self.input_buffer = ""

        self.results = []
        self.selected = 0
        self.results_offset = 0
        self.detail = None

        self.season_selected = 0
        self.season_offset = 0
        self.season_data = None
        self.episodes = []
        self.episode_selected = 0
        self.episode_offset = 0
        self.episode_detail = None

        self.status = "Ready. / to search, i for ID lookup, Tab sets ID type, q to quit."
        self.error = None

        curses.curs_set(0)
        curses.start_colors_compat = curses.start_color()
        curses.use_default_colors()
        curses.init_pair(1, curses.COLOR_CYAN, -1)
        curses.init_pair(2, curses.COLOR_YELLOW, -1)
        curses.init_pair(3, curses.COLOR_GREEN, -1)
        curses.init_pair(4, curses.COLOR_RED, -1)
        curses.init_pair(5, curses.COLOR_BLACK, curses.COLOR_CYAN)
        curses.init_pair(6, curses.COLOR_MAGENTA, -1)

        stdscr.nodelay(False)
        stdscr.keypad(True)

    @staticmethod
    def adjust_scroll(selected, offset, height):
        if height <= 0:
            return 0
        if selected < offset:
            offset = selected
        elif selected >= offset + height:
            offset = selected - height + 1
        return max(0, offset)

    @staticmethod
    def validate_season_number(value: str):
        if not value.isdigit():
            return None, f"'{value}' is not a valid season number."
        return int(value), None

    @staticmethod
    def validate_episode_number(value: str):
        if not value.isdigit():
            return None, f"'{value}' is not a valid episode number."
        num = int(value)
        if num <= 0:
            return None, "Episode number must be a positive integer."
        return num, None

    def get_tv_seasons(self):
        if not self.detail or self.detail.get("media_type") != "tv":
            return []
        return sorted(self.detail.get("seasons", []), key=lambda s: s.get("season_number", 0))

    def find_episode_by_number(self, num: int):
        for idx, ep in enumerate(self.episodes):
            if ep.get("episode_number") == num:
                return idx
        return None

    def _get_provider_idx_by_key(self, key: int) -> Optional[int]:
        char = chr(key)
        for idx, provider in enumerate(PROVIDERS):
            if provider.key == char:
                return idx
        return None

    def do_search(self):
        if not self.title_query:
            self.status = "Type a title first (/)."
            return
        self.status = f"Searching movies + TV for '{self.title_query}'..."
        self.draw()
        results, err = self.client.search_multi(self.title_query, self.year_query)
        self.error = err
        if err:
            self.status = "Search failed."
            return
        self.results = results
        self.selected = 0
        self.results_offset = 0
        self.detail = None
        self.screen = SCREEN_RESULTS
        self.status = (
            f"{len(results)} result(s) for '{self.title_query}'"
            + (f" ({self.year_query})" if self.year_query else "")
        )

    def do_id_lookup(self):
        if not self.id_query:
            self.status = "Type an ID first (i)."
            return
        self.status = f"Fetching {self.id_lookup_type} id={self.id_query}..."
        self.draw()
        data, err = self.client.get_by_id(self.id_lookup_type, self.id_query)
        self.error = err
        if err:
            self.status = "Lookup failed."
            return
        data["media_type"] = self.id_lookup_type
        self.results = []
        self.selected = 0
        self._enter_detail(data)
        self.status = f"Loaded id={self.id_query}"

    def open_selected_detail(self):
        if not self.results:
            return
        item = self.results[self.selected]
        media_type = item.get("media_type", "movie")
        self.status = f"Loading details for id={item.get('id')}..."
        self.draw()
        data, err = self.client.get_by_id(media_type, item.get("id"))
        self.error = err
        if err:
            self.status = f"Failed to load details: {err}"
            return
        data["media_type"] = media_type
        self._enter_detail(data)
        self.status = "Details loaded."

    def _enter_detail(self, data):
        self.detail = data
        self.season_selected = 0
        self.season_offset = 0
        self.season_data = None
        self.episodes = []
        self.episode_detail = None
        self.screen = SCREEN_DETAIL

    def load_season(self, season_number_str: str):
        if self.detail is None or self.detail.get("media_type") != "tv":
            return
        num, err = self.validate_season_number(season_number_str)
        if err:
            self.status = err
            return
        seasons = self.get_tv_seasons()
        if num not in [s.get("season_number") for s in seasons]:
            self.status = f"Season {num} does not exist for this show."
            return
        tv_id = self.detail.get("id")
        self.status = f"Loading season {num}..."
        self.draw()
        data, err = self.client.get_season(tv_id, num)
        self.error = err
        if err:
            self.status = f"Failed to load Season {num}: {err}"
            return
        self.season_data = data
        self.episodes = data.get("episodes", [])
        self.episode_selected = 0
        self.episode_offset = 0
        self.episode_detail = None
        self.screen = SCREEN_EPISODES
        for idx, s in enumerate(seasons):
            if s.get("season_number") == num:
                self.season_selected = idx
                break
        self.status = f"Season {num}: {len(self.episodes)} episode(s)."

    def select_season(self, season: dict):
        self.load_season(str(season.get("season_number", "")))

    def open_episode_by_index(self, idx: int):
        if not (0 <= idx < len(self.episodes)):
            self.status = "No such episode in this season."
            return
        self.episode_selected = idx
        self.episode_detail = self.episodes[idx]
        self.screen = SCREEN_EPISODE_DETAIL
        self.status = "Esc to go back, p to play."

    def open_episode_by_number(self, value: str):
        num, err = self.validate_episode_number(value)
        if err:
            self.status = err
            return
        idx = self.find_episode_by_number(num)
        if idx is None:
            season_num = self.season_data.get("season_number", "?") if self.season_data else "?"
            self.status = f"Episode {num} not found in Season {season_num}."
            return
        self.open_episode_by_index(idx)

    def play_stream(self, start_provider_idx=None):
        if not self.detail:
            self.status = "No media selected."
            return

        media_type = self.detail.get("media_type", "movie")
        tmdb_id = str(self.detail.get("id", ""))
        title = self.detail.get("title") or self.detail.get("name", "")
        year = (self.detail.get("release_date") or self.detail.get("first_air_date") or "")[:4]

        season_num = "1"
        episode_num = "1"
        if media_type == "tv":
            if not self.season_data or not self.episode_detail:
                self.status = "Please select a season and episode first."
                return
            season_num = str(self.season_data.get("season_number", "1"))
            episode_num = str(self.episode_detail.get("episode_number", "1"))

        start_idx = start_provider_idx if start_provider_idx is not None else 0
        start_idx = max(0, min(start_idx, len(PROVIDERS) - 1))

        for idx in range(start_idx, len(PROVIDERS)):
            provider = PROVIDERS[idx]
            self.status = f"Fetching streams from {provider.name}..."
            self.draw()
            self.stdscr.refresh()

            try:
                candidate_streams = provider.fetch_func(
                    tmdb_id, title, media_type, season_num, episode_num, year=year
                )
            except Exception as e:
                log.warning("%s failed: %s", provider.name, e)
                self.status = f"{provider.name} failed, falling back to next provider..."
                self.draw()
                self.stdscr.refresh()
                continue

            provider_success = False
            for i, (stream_url, stream_headers) in enumerate(candidate_streams):
                mpv_cmd = ["mpv", "--no-ytdl"]

                ua = stream_headers.get("User-Agent") or stream_headers.get("user-agent")
                if ua:
                    mpv_cmd.append(f"--user-agent={ua}")

                referer = stream_headers.get("Referer") or stream_headers.get("referer")
                if referer:
                    mpv_cmd.append(f"--referrer={referer}")

                # mpv's documented syntax: comma-separated "Field: value" pairs.
                # No embedded quotes -- this is an argv list, not a shell command.
                extra_headers = [
                    f"{k}: {v}" for k, v in stream_headers.items()
                    if k.lower() not in ("user-agent", "referer")
                ]
                if extra_headers:
                    mpv_cmd.append("--http-header-fields=" + ",".join(extra_headers))

                mpv_cmd.append(stream_url)

                try:
                    self.status = f"Playing via {provider.name} (Stream {i + 1})..."
                    self.draw()
                    self.stdscr.refresh()

                    curses.endwin()
                    result = subprocess.run(mpv_cmd)
                    curses.reset_prog_mode()
                    curses.doupdate()
                    self.stdscr.refresh()

                    if result.returncode == 0:
                        provider_success = True
                        break
                    else:
                        log.warning("Stream %d failed (mpv code %d)", i + 1, result.returncode)

                except FileNotFoundError:
                    self.status = "Error: 'mpv' executable not found."
                    provider_success = True
                    break
                except Exception as e:
                    log.warning("Playback error: %s", e)

            if provider_success:
                self.status = "Playback finished."
                self.draw()
                self.stdscr.refresh()
                return

            self.status = f"All streams in {provider.name} failed. Trying next provider..."
            self.draw()
            self.stdscr.refresh()

        self.status = "All providers failed to find or play streams."
        self.draw()
        self.stdscr.refresh()

    def draw(self):
        stdscr = self.stdscr
        stdscr.erase()
        h, w = stdscr.getmaxyx()

        if h < MIN_HEIGHT or w < MIN_WIDTH:
            self.draw_too_small(h, w)
            stdscr.refresh()
            return

        header = f" TMDB TUI — search: movies + tv   |   id-lookup type: {self.id_lookup_type.upper()} (Tab) "
        safe_addstr(stdscr, 0, 0, header.ljust(w), curses.color_pair(1) | curses.A_BOLD | curses.A_REVERSE)
        bar = f" title: {self.title_query or '-'}    year: {self.year_query or '-'}    id: {self.id_query or '-'} "
        safe_addstr(stdscr, 1, 0, bar.ljust(w), curses.color_pair(2))

        list_top = 3
        list_bottom = h - 3
        list_height = max(0, list_bottom - list_top)

        if self.screen == SCREEN_RESULTS:
            self.draw_results(list_top, list_height, w)
        elif self.screen == SCREEN_DETAIL:
            self.draw_detail(list_top, list_bottom, list_height, w)
        elif self.screen == SCREEN_EPISODES:
            self.draw_episode_list(list_top, list_height, w)
        elif self.screen == SCREEN_EPISODE_DETAIL:
            self.draw_episode_detail(list_top, list_bottom, w)

        if self.mode != MODE_NORMAL:
            prompt = {
                MODE_SEARCH: "Search title> ",
                MODE_YEAR: "Year> ",
                MODE_ID: "TMDB ID> ",
                MODE_SEASON: "Season number (0 = Specials)> ",
                MODE_EPISODE: "Episode number> ",
            }[self.mode]
            safe_addstr(stdscr, h - 2, 0, (prompt + self.input_buffer).ljust(w), curses.A_BOLD)
        else:
            safe_addstr(stdscr, h - 2, 0, self.help_text().ljust(w), curses.color_pair(2) | curses.A_DIM)

        status_line = self.error if self.error else self.status
        color = curses.color_pair(4) if self.error else curses.color_pair(3)
        safe_addstr(stdscr, h - 1, 0, status_line.ljust(w), color)
        stdscr.refresh()

    def draw_too_small(self, h, w):
        stdscr = self.stdscr
        lines = [
            "Terminal too small",
            f"Current: {w}x{h}",
            f"Need at least: {MIN_WIDTH}x{MIN_HEIGHT}",
            "Resize the window or zoom out.",
        ]
        start_row = max(0, (h - len(lines)) // 2)
        for i, line in enumerate(lines):
            row = start_row + i
            if row >= h:
                break
            col = max(0, (w - len(line)) // 2)
            safe_addstr(stdscr, row, col, line, curses.color_pair(4) | curses.A_BOLD)

    def help_text(self):
        provider_help = "  ".join([f"{p.key} {p.name}" for p in PROVIDERS]) + "  p Play (Auto)"
        if self.screen == SCREEN_RESULTS:
            return " / Search   y Year   i ID lookup   Tab Movie/TV ID type   j/k Move   Enter Open   q Quit"
        if self.screen == SCREEN_DETAIL:
            if self.detail and self.detail.get("media_type") == "tv":
                return f"{provider_help}   j/k Select season   Enter Open   s Type season #   Esc Back"
            return f"{provider_help}   Esc Back"
        if self.screen == SCREEN_EPISODES:
            return f"{provider_help}   j/k Select ep   Enter Open   e Jump   Esc Back"
        if self.screen == SCREEN_EPISODE_DETAIL:
            return f"{provider_help}   Esc Back"
        return ""

    @staticmethod
    def _media_tag(mtype):
        return "[TV]  " if mtype == "tv" else "[MOVIE]"

    def draw_results(self, list_top, list_height, w):
        stdscr = self.stdscr
        safe_addstr(stdscr, list_top - 1, 0, "Results:".ljust(w), curses.A_UNDERLINE)
        if not self.results:
            safe_addstr(stdscr, list_top, 2, "(no results yet)")
            return
        self.results_offset = self.adjust_scroll(self.selected, self.results_offset, list_height)
        visible = self.results[self.results_offset: self.results_offset + list_height]
        for row, item in enumerate(visible):
            i = row + self.results_offset
            name = item.get("title") or item.get("name") or "?"
            date = item.get("release_date") or item.get("first_air_date") or "----"
            rating = item.get("vote_average") or 0
            mtype = item.get("media_type", "?")
            tag = self._media_tag(mtype)
            line = f"{name[:42]:<42} {date[:4]:>6}  \u2605{rating:.1f}  {tag} id={item.get('id')}"
            attr = curses.color_pair(5) if i == self.selected else 0
            safe_addstr(stdscr, list_top + row, 2, line, attr)
        if len(self.results) > list_height:
            safe_addstr(stdscr, list_top - 1, w - 14, f"[{self.selected + 1}/{len(self.results)}]")

    def draw_detail(self, top, bottom, list_height, w):
        stdscr = self.stdscr
        d = self.detail
        name = d.get("title") or d.get("name") or "?"
        date = d.get("release_date") or d.get("first_air_date") or "?"
        rating = d.get("vote_average") or 0
        genres = ", ".join(g["name"] for g in d.get("genres", []))
        overview = d.get("overview") or "(no overview)"
        mtype = d.get("media_type", "?")
        tmdb_id = d.get("id", "?")

        row = top
        if row < bottom:
            safe_addstr(
                stdscr, row, 2,
                f"{name}  ({date[:4] if date else '?'})  {self._media_tag(mtype)} id={tmdb_id}",
                curses.A_BOLD,
            )
            row += 1
        if row < bottom:
            safe_addstr(stdscr, row, 2, f"Rating: {rating:.1f}   Genres: {genres}")
            row += 2

        if mtype == "tv":
            seasons = self.get_tv_seasons()
            if row < bottom:
                safe_addstr(stdscr, row, 2, f"Seasons ({len(seasons)}):", curses.A_UNDERLINE)
                row += 1
            seasons_top = row
            seasons_height = max(0, bottom - seasons_top)
            if not seasons:
                if seasons_top < bottom:
                    safe_addstr(stdscr, seasons_top, 4, "(no season data available)")
                return
            self.season_offset = self.adjust_scroll(self.season_selected, self.season_offset, seasons_height)
            visible = seasons[self.season_offset: self.season_offset + seasons_height]
            for i, season in enumerate(visible):
                idx = i + self.season_offset
                num = season.get("season_number", "?")
                sname = season.get("name") or "?"
                ep_count = season.get("episode_count", 0)
                line = f"S{num:<3} {sname[:35]:<35} {ep_count} episode(s)"
                attr = curses.color_pair(5) if idx == self.season_selected else 0
                if seasons_top + i < bottom:
                    safe_addstr(stdscr, seasons_top + i, 4, line, attr)
        else:
            if row < bottom:
                safe_addstr(stdscr, row, 2, "Overview:", curses.A_UNDERLINE)
                row += 1
            for line in textwrap.wrap(overview, max(10, w - 6)):
                if row >= bottom:
                    break
                safe_addstr(stdscr, row, 4, line)
                row += 1

    def draw_episode_list(self, list_top, list_height, w):
        stdscr = self.stdscr
        show_name = self.detail.get("name") if self.detail else "?"
        season_num = self.season_data.get("season_number", "?") if self.season_data else "?"
        header = f"{show_name} — Season {season_num}  ({len(self.episodes)} episodes)"
        safe_addstr(stdscr, list_top - 1, 0, header.ljust(w), curses.A_UNDERLINE)
        if not self.episodes:
            safe_addstr(stdscr, list_top, 2, "(no episodes found for this season)")
            return
        self.episode_offset = self.adjust_scroll(self.episode_selected, self.episode_offset, list_height)
        visible = self.episodes[self.episode_offset: self.episode_offset + list_height]
        for row, ep in enumerate(visible):
            i = row + self.episode_offset
            num = ep.get("episode_number", "?")
            name = ep.get("name") or "?"
            date = ep.get("air_date") or "----------"
            rating = ep.get("vote_average") or 0
            line = f"E{num:<3} {name[:42]:<42} {date}  \u2605{rating:.1f}"
            attr = curses.color_pair(5) if i == self.episode_selected else 0
            safe_addstr(stdscr, list_top + row, 2, line, attr)

    def draw_episode_detail(self, top, bottom, w):
        stdscr = self.stdscr
        ep = self.episode_detail
        if ep is None:
            safe_addstr(stdscr, top, 2, "(no episode loaded)")
            return
        show_name = self.detail.get("name") if self.detail else "?"
        season_num = ep.get("season_number", "?")
        num = ep.get("episode_number", "?")
        name = ep.get("name") or "?"
        date = ep.get("air_date") or "?"
        rating = ep.get("vote_average") or 0
        overview = ep.get("overview") or "(no overview)"
        row = top
        if row < bottom:
            safe_addstr(stdscr, row, 2, f"{show_name} — S{season_num}E{num}: {name}", curses.A_BOLD)
            row += 1
        if row < bottom:
            safe_addstr(stdscr, row, 2, f"Air date: {date}   Rating: {rating:.1f}")
            row += 2
        if row < bottom:
            safe_addstr(stdscr, row, 2, "Overview:", curses.A_UNDERLINE)
            row += 1
        for line in textwrap.wrap(overview, max(10, w - 6)):
            if row >= bottom:
                break
            safe_addstr(stdscr, row, 4, line)
            row += 1

    def handle_key(self, key):
        if key == curses.KEY_RESIZE:
            curses.update_lines_cols()
            self.stdscr.clear()
            self.stdscr.refresh()
            return
        if self.mode != MODE_NORMAL:
            self.handle_text_input(key)
            return
        if key in (ord("q"), ord("Q")):
            return "quit"
        if self.screen == SCREEN_RESULTS:
            self.handle_key_results(key)
        elif self.screen == SCREEN_DETAIL:
            self.handle_key_detail(key)
        elif self.screen == SCREEN_EPISODES:
            self.handle_key_episodes(key)
        elif self.screen == SCREEN_EPISODE_DETAIL:
            self.handle_key_episode_detail(key)

    def handle_key_results(self, key):
        if key == 9:
            self.id_lookup_type = "tv" if self.id_lookup_type == "movie" else "movie"
            self.status = f"ID-lookup type set to {self.id_lookup_type.upper()} (used by the 'i' command)."
        elif key == ord("/"):
            self.mode = MODE_SEARCH
            self.input_buffer = self.title_query
        elif key == ord("y"):
            self.mode = MODE_YEAR
            self.input_buffer = self.year_query
        elif key == ord("i"):
            self.mode = MODE_ID
            self.input_buffer = self.id_query
        elif key in (curses.KEY_DOWN, ord("j")):
            if self.results:
                self.selected = min(self.selected + 1, len(self.results) - 1)
        elif key in (curses.KEY_UP, ord("k")):
            if self.results:
                self.selected = max(self.selected - 1, 0)
        elif key in (curses.KEY_ENTER, 10, 13):
            self.open_selected_detail()

    def handle_key_detail(self, key):
        if key == 27:
            self.detail = None
            self.season_data = None
            self.episodes = []
            self.episode_detail = None
            self.error = None
            self.screen = SCREEN_RESULTS
            self.status = "Back to results."
            return
        provider_idx = self._get_provider_idx_by_key(key)
        if provider_idx is not None:
            self.play_stream(start_provider_idx=provider_idx)
            return
        if key in (ord("p"), ord("P")):
            self.play_stream(start_provider_idx=None)
            return
        if not self.detail or self.detail.get("media_type") != "tv":
            return
        seasons = self.get_tv_seasons()
        if key in (curses.KEY_DOWN, ord("j")):
            if seasons:
                self.season_selected = min(self.season_selected + 1, len(seasons) - 1)
        elif key in (curses.KEY_UP, ord("k")):
            if seasons:
                self.season_selected = max(self.season_selected - 1, 0)
        elif key in (curses.KEY_ENTER, 10, 13):
            if seasons:
                self.select_season(seasons[self.season_selected])
        elif key == ord("s"):
            self.mode = MODE_SEASON
            self.input_buffer = ""

    def handle_key_episodes(self, key):
        provider_idx = self._get_provider_idx_by_key(key)
        if provider_idx is not None:
            self.play_stream(start_provider_idx=provider_idx)
            return
        elif key in (ord("p"), ord("P")):
            self.play_stream(start_provider_idx=None)
            return
        elif key == ord("e"):
            self.mode = MODE_EPISODE
            self.input_buffer = ""
        elif key in (curses.KEY_DOWN, ord("j")):
            if self.episodes:
                self.episode_selected = min(self.episode_selected + 1, len(self.episodes) - 1)
        elif key in (curses.KEY_UP, ord("k")):
            if self.episodes:
                self.episode_selected = max(self.episode_selected - 1, 0)
        elif key in (curses.KEY_ENTER, 10, 13):
            self.open_episode_by_index(self.episode_selected)
        elif key == 27:
            self.screen = SCREEN_DETAIL
            self.error = None
            self.status = "Back to show details."

    def handle_key_episode_detail(self, key):
        provider_idx = self._get_provider_idx_by_key(key)
        if provider_idx is not None:
            self.play_stream(start_provider_idx=provider_idx)
            return
        elif key in (ord("p"), ord("P")):
            self.play_stream(start_provider_idx=None)
            return
        elif key == 27:
            self.screen = SCREEN_EPISODES
            self.error = None
            self.status = "Back to episode list."

    def handle_text_input(self, key):
        if key == 27:
            self.mode = MODE_NORMAL
            self.input_buffer = ""
            return
        if key in (curses.KEY_ENTER, 10, 13):
            value = self.input_buffer.strip()
            mode = self.mode
            self.mode = MODE_NORMAL
            if mode == MODE_SEARCH:
                self.title_query = value
                self.do_search()
            elif mode == MODE_YEAR:
                self.year_query = value
                if self.title_query:
                    self.do_search()
            elif mode == MODE_ID:
                self.id_query = value
                self.do_id_lookup()
            elif mode == MODE_SEASON:
                if value:
                    self.load_season(value)
                else:
                    self.status = "Type a season number first."
            elif mode == MODE_EPISODE:
                if value:
                    self.open_episode_by_number(value)
                else:
                    self.status = "Type an episode number first."
            return
        if key in (curses.KEY_BACKSPACE, 127, 8):
            self.input_buffer = self.input_buffer[:-1]
            return
        if 32 <= key <= 126:
            self.input_buffer += chr(key)

    def run(self):
        self.draw()
        while True:
            key = self.stdscr.getch()
            result = self.handle_key(key)
            if result == "quit":
                break
            self.draw()


def main(stdscr):
    app = App(stdscr)
    app.run()


if __name__ == "__main__":
    logging.basicConfig(
        filename="tmdb_tui.log",
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s %(message)s",
    )
    locale.setlocale(locale.LC_ALL, "")
    curses.wrapper(main)
