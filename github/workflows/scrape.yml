#!/usr/bin/env python3
"""
Web Monitor Scraper
Überwacht Websites auf neue Threads oder Posts und speichert Ergebnisse.
"""

import json
import os
import re
import sys
import time
import urllib.request
import urllib.parse
import urllib.error
from datetime import datetime, timezone
from html.parser import HTMLParser


# ---------------------------------------------------------------------------
# Hilfsfunktionen
# ---------------------------------------------------------------------------

def log(msg):
    print(f"[{datetime.now().strftime('%H:%M:%S')}] {msg}", flush=True)


def load_json(path, default):
    if os.path.exists(path):
        with open(path, "r", encoding="utf-8") as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                return default
    return default


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


def fetch_url(url, timeout=20):
    """Lädt eine URL und gibt den HTML-Text zurück."""
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (iPad; CPU OS 16_0 like Mac OS X) "
            "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.0 "
            "Mobile/15E148 Safari/604.1"
        ),
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "de,en;q=0.5",
    }
    req = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            charset = "utf-8"
            ct = resp.headers.get_content_charset()
            if ct:
                charset = ct
            return resp.read().decode(charset, errors="replace")
    except Exception as e:
        log(f"  Fehler beim Laden von {url}: {e}")
        return None


# ---------------------------------------------------------------------------
# Einfacher HTML-Link-Parser
# ---------------------------------------------------------------------------

class LinkParser(HTMLParser):
    def __init__(self, base_url=""):
        super().__init__()
        self.links = []
        self.base_url = base_url
        self._current_href = None
        self._current_text = []

    def handle_starttag(self, tag, attrs):
        if tag == "a":
            attrs_dict = dict(attrs)
            href = attrs_dict.get("href", "")
            if href:
                self._current_href = href
                self._current_text = []

    def handle_endtag(self, tag):
        if tag == "a" and self._current_href:
            text = " ".join(self._current_text).strip()
            href = self._current_href
            if href.startswith("//"):
                parsed = urllib.parse.urlparse(self.base_url)
                href = f"{parsed.scheme}:{href}"
            elif href.startswith("/"):
                parsed = urllib.parse.urlparse(self.base_url)
                href = f"{parsed.scheme}://{parsed.netloc}{href}"
            elif not href.startswith("http"):
                href = urllib.parse.urljoin(self.base_url, href)
            self.links.append({"href": href, "text": text})
            self._current_href = None
            self._current_text = []

    def handle_data(self, data):
        if self._current_href is not None:
            self._current_text.append(data.strip())


def extract_links(html, base_url):
    parser = LinkParser(base_url)
    try:
        parser.feed(html)
    except Exception:
        pass
    return parser.links


# ---------------------------------------------------------------------------
# Typ 1: Neue Threads/Seiten suchen
# ---------------------------------------------------------------------------

def scrape_type1(site, history_site):
    """
    Sucht auf einer Übersichtsseite nach Links, die dem URL-Muster entsprechen.
    Gibt neue Einträge zurück (noch nicht in der History).
    """
    url = site["url"]
    pattern = site.get("pattern", "")
    tags = [t.strip().lower() for t in site.get("tags", "").split(",") if t.strip()]
    max_results = int(site.get("max_results", 50))

    known_urls = set(history_site.get("known_urls", []))
    log(f"  Typ 1 – Übersicht: {url}")
    log(f"  Muster: '{pattern}' | Tags: {tags}")

    html = fetch_url(url)
    if not html:
        return [], known_urls

    links = extract_links(html, url)
    log(f"  {len(links)} Links gefunden")

    new_entries = []
    all_urls = set(known_urls)

    for link in links:
        href = link["href"]
        text = link["text"]

        # Muster-Filter
        if pattern and pattern.lower() not in href.lower() and pattern.lower() not in text.lower():
            continue

        # Tag-Filter (mindestens ein Tag muss im Text oder URL vorkommen)
        if tags:
            combined = (href + " " + text).lower()
            if not any(tag in combined for tag in tags):
                continue

        if href not in all_urls:
            new_entries.append({
                "url": href,
                "title": text or href,
                "found_at": datetime.now(timezone.utc).isoformat(),
                "is_new": True,
            })
            all_urls.add(href)

        if len(new_entries) >= max_results:
            break

    log(f"  {len(new_entries)} neue Einträge")
    return new_entries, all_urls


# ---------------------------------------------------------------------------
# Typ 2: Neue Posts in bestehendem Thread suchen
# ---------------------------------------------------------------------------

class PostCountParser(HTMLParser):
    """Versucht, die Anzahl von Post-Elementen zu zählen."""
    def __init__(self, selectors):
        super().__init__()
        self.count = 0
        self.selectors = selectors  # Liste von class-Namen die auf Posts hindeuten

    def handle_starttag(self, tag, attrs):
        attrs_dict = dict(attrs)
        cls = attrs_dict.get("class", "")
        for sel in self.selectors:
            if sel in cls:
                self.count += 1
                break


def scrape_type2(site, history_site):
    """
    Überwacht eine Thread-URL auf neue Posts.
    Vergleicht die Post-Anzahl oder findet neue IDs.
    """
    url = site["url"]
    post_selectors = [s.strip() for s in site.get("post_selectors", "post,message,reply,comment,beitrag").split(",") if s.strip()]
    tags = [t.strip().lower() for t in site.get("tags", "").split(",") if t.strip()]

    last_count = history_site.get("last_post_count", 0)
    last_check = history_site.get("last_check", "")

    log(f"  Typ 2 – Thread: {url}")
    log(f"  Letzter bekannter Post-Count: {last_count}")

    html = fetch_url(url)
    if not html:
        return [], last_count

    # Post-Count über CSS-Klassen
    parser = PostCountParser(post_selectors)
    try:
        parser.feed(html)
    except Exception:
        pass
    current_count = parser.count

    # Fallback: Versuche Zahlen aus dem HTML zu lesen (z. B. "42 Antworten")
    if current_count == 0:
        matches = re.findall(r'(\d+)\s*(?:posts?|antworten?|replies|beitr[äa]ge?|kommentare?)', html, re.IGNORECASE)
        if matches:
            current_count = max(int(m) for m in matches)

    log(f"  Aktueller Post-Count: {current_count}")

    new_entries = []
    if current_count > last_count and last_count > 0:
        diff = current_count - last_count
        new_entries.append({
            "url": url,
            "title": f"{diff} neue Post(s) im Thread ({current_count} gesamt)",
            "found_at": datetime.now(timezone.utc).isoformat(),
            "is_new": True,
            "post_count": current_count,
        })
    elif last_count == 0 and current_count > 0:
        new_entries.append({
            "url": url,
            "title": f"Thread-Baseline gesetzt: {current_count} Posts",
            "found_at": datetime.now(timezone.utc).isoformat(),
            "is_new": False,
            "post_count": current_count,
        })

    return new_entries, current_count


# ---------------------------------------------------------------------------
# Hauptlogik
# ---------------------------------------------------------------------------

def main():
    config_path = "config.json"
    history_path = "history.json"
    results_path = "results.json"

    # --- Config laden oder Standard erstellen ---
    default_config = {
        "sites": [],
        "settings": {
            "max_new_per_run": 100,
            "delay_between_requests": 2
        }
    }
    config = load_json(config_path, default_config)
    if not os.path.exists(config_path):
        save_json(config_path, default_config)
        log("config.json erstellt (leer)")

    # --- History laden oder erstellen ---
    default_history = {"sites": {}, "last_run": ""}
    history = load_json(history_path, default_history)
    if not os.path.exists(history_path):
        save_json(history_path, default_history)
        log("history.json erstellt")

    # --- Ergebnisse laden (für Akkumulation) ---
    default_results = {"last_run": "", "runs": [], "all_entries": []}
    results = load_json(results_path, default_results)

    sites = config.get("sites", [])
    if not sites:
        log("Keine Websites in config.json konfiguriert. Bitte über die UI hinzufügen.")
        # Trotzdem Dateien aktualisieren
        history["last_run"] = datetime.now(timezone.utc).isoformat()
        save_json(history_path, history)
        results["last_run"] = history["last_run"]
        save_json(results_path, results)
        return

    delay = int(config.get("settings", {}).get("delay_between_requests", 2))
    run_entries = []
    run_timestamp = datetime.now(timezone.utc).isoformat()

    for site in sites:
        site_id = site.get("id", site.get("url", "unknown"))
        site_name = site.get("name", site_id)
        site_type = int(site.get("type", 1))
        active = site.get("active", True)

        if not active:
            log(f"Übersprungen (inaktiv): {site_name}")
            continue

        log(f"\n>>> Verarbeite: {site_name} (Typ {site_type})")
        history_site = history["sites"].get(site_id, {})

        try:
            if site_type == 1:
                new_entries, known_urls = scrape_type1(site, history_site)
                history["sites"][site_id] = {
                    "known_urls": list(known_urls),
                    "last_check": run_timestamp,
                    "last_new_count": len(new_entries),
                }
            elif site_type == 2:
                new_entries, post_count = scrape_type2(site, history_site)
                history["sites"][site_id] = {
                    "last_post_count": post_count,
                    "last_check": run_timestamp,
                    "last_new_count": len(new_entries),
                }
            else:
                log(f"  Unbekannter Typ: {site_type}")
                new_entries = []

            for e in new_entries:
                e["site_id"] = site_id
                e["site_name"] = site_name
                e["type"] = site_type

            run_entries.extend(new_entries)
            log(f"  Fertig: {len(new_entries)} neue Einträge")

        except Exception as ex:
            log(f"  FEHLER bei {site_name}: {ex}")

        time.sleep(delay)

    # --- Ergebnisse zusammenführen ---
    history["last_run"] = run_timestamp
    save_json(history_path, history)

    # Neue Einträge vorne anhängen
    existing = results.get("all_entries", [])
    # Duplikate vermeiden
    existing_urls = {e.get("url") for e in existing}
    truly_new = [e for e in run_entries if e.get("url") not in existing_urls]

    results["last_run"] = run_timestamp
    results["all_entries"] = truly_new + existing

    # Letzten Run speichern
    runs = results.get("runs", [])
    runs.insert(0, {
        "timestamp": run_timestamp,
        "new_count": len(truly_new),
        "sites_checked": len([s for s in sites if s.get("active", True)]),
    })
    results["runs"] = runs[:50]  # Max 50 Runs behalten
    save_json(results_path, results)

    log(f"\n=== Abgeschlossen: {len(truly_new)} neue Einträge insgesamt ===")
    log(f"history.json und results.json aktualisiert.")


if __name__ == "__main__":
    main()
