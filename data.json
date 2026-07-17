#!/usr/bin/env python3
"""
Immoweb region scraper
=======================

Fetches real-estate listings from immoweb.be for a given region/postal code
and saves them as a JSON file that can be visualized with map_viewer.html.

IMPORTANT NOTES (read before running):
- Immoweb's HTML/JS structure changes periodically. This script tries THREE
  extraction strategies in order (see extract_listings_from_html) so that if
  one breaks, another may still work. If NONE work, open a search results
  page in your browser, view source / inspect, and update the selectors
  marked with "ADJUST ME" below.
- Scraping likely falls outside Immoweb's Terms of Service. This script is
  provided for personal, non-commercial, rate-limited use only. You are
  responsible for how you use it. Check https://www.immoweb.be/robots.txt
  and their ToS before running this at any real scale.
- Be a good citizen: keep REQUEST_DELAY reasonably high, don't run this on
  a schedule hitting their servers constantly, and stop if you get blocked
  (repeated 403s means: stop, don't try to work around it).

Usage:
    python immoweb_scraper.py --postal-codes 1000,1050,1180 --type house-and-apartment --sale --max-pages 5
    python immoweb_scraper.py --postal-codes 4000 --type apartment --rent --max-pages 3 --out liege.json
"""

import argparse
import json
import re
import sys
import time
import random
from dataclasses import dataclass, asdict, field
from typing import Optional

import requests
from bs4 import BeautifulSoup

BASE_URL = "https://www.immoweb.be"
REQUEST_DELAY = (2.5, 5.0)  # random delay range (seconds) between requests -- be polite
HEADERS = {
    # A normal desktop browser UA. Update if requests start getting blocked.
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
        "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
    ),
    "Accept-Language": "en-US,en;q=0.9,fr;q=0.8,nl;q=0.7",
}


@dataclass
class Listing:
    id: Optional[str] = None
    title: Optional[str] = None
    property_type: Optional[str] = None
    transaction: Optional[str] = None  # "for-sale" / "for-rent"
    price: Optional[float] = None
    price_text: Optional[str] = None
    locality: Optional[str] = None
    postal_code: Optional[str] = None
    bedrooms: Optional[int] = None
    living_area_m2: Optional[float] = None
    latitude: Optional[float] = None
    longitude: Optional[float] = None
    url: Optional[str] = None
    raw: dict = field(default_factory=dict)  # keep original blob for debugging


def build_search_url(postal_code: str, property_type: str, transaction: str, page: int) -> str:
    """
    Builds an Immoweb search URL.
    transaction: "for-sale" or "for-rent"
    property_type: e.g. "house-and-apartment", "house", "apartment"

    ADJUST ME: Immoweb periodically changes the URL scheme for search results.
    If this stops returning results, open immoweb.be, run a manual search,
    and copy the resulting URL pattern here.
    """
    return (
        f"{BASE_URL}/en/search-results/{property_type}/{transaction}"
        f"?postalCode={postal_code}&page={page}&orderBy=relevance"
    )


def fetch(url: str) -> Optional[str]:
    try:
        resp = requests.get(url, headers=HEADERS, timeout=20)
        if resp.status_code == 403:
            print(f"[!] 403 Forbidden at {url}. Immoweb is blocking this request. "
                  f"Stop and reconsider (rotate UA / slow down / use a real browser session).",
                  file=sys.stderr)
            return None
        resp.raise_for_status()
        return resp.text
    except requests.RequestException as e:
        print(f"[!] Request failed for {url}: {e}", file=sys.stderr)
        return None


def extract_listings_from_html(html: str) -> list[dict]:
    """
    Tries multiple strategies to pull structured listing data out of a
    search-results page. Returns a list of raw dicts (site-specific shape,
    not yet normalized into Listing objects).
    """
    soup = BeautifulSoup(html, "html.parser")
    results: list[dict] = []

    # --- Strategy 1: __NEXT_DATA__ / embedded JSON state blob -----------
    # Many modern sites (Immoweb included, historically) embed the page's
    # initial data as JSON in a <script> tag. Look for common patterns.
    for script in soup.find_all("script"):
        script_id = script.get("id", "")
        if script_id in ("__NEXT_DATA__", "__INITIAL_STATE__") or (
            script.string and "window.classified" in (script.string or "")
        ):
            try:
                text = script.string or ""
                # Handle "window.X = {...}" assignment style
                if "=" in text and not text.strip().startswith("{"):
                    text = text.split("=", 1)[1].strip().rstrip(";")
                data = json.loads(text)
                found = _find_listing_arrays(data)
                if found:
                    results.extend(found)
            except (json.JSONDecodeError, AttributeError):
                continue

    if results:
        return results

    # --- Strategy 2: JSON-LD structured data (schema.org) ----------------
    for script in soup.find_all("script", type="application/ld+json"):
        try:
            data = json.loads(script.string or "")
            items = data if isinstance(data, list) else [data]
            for item in items:
                if item.get("@type") in ("RealEstateListing", "Product", "Offer"):
                    results.append(item)
        except (json.JSONDecodeError, AttributeError):
            continue

    if results:
        return results

    # --- Strategy 3: raw HTML card scraping (ADJUST ME) -------------------
    # Fallback: look for listing cards directly. Immoweb has used
    # article[class*="card"] elements historically. Selectors below are a
    # best guess and MOST LIKELY need updating -- inspect the live page
    # (right-click a listing -> Inspect) and adjust the selectors.
    cards = soup.select("article[class*='card'], li[class*='search-results__item']")
    for card in cards:
        link_el = card.select_one("a[href*='/en/classified/']") or card.select_one("a[href]")
        price_el = card.select_one("[class*='price']")
        title_el = card.select_one("[class*='title'], h2, h3")
        results.append({
            "url": link_el["href"] if link_el and link_el.has_attr("href") else None,
            "price_text": price_el.get_text(strip=True) if price_el else None,
            "title": title_el.get_text(strip=True) if title_el else None,
        })

    return results


def _find_listing_arrays(data, depth: int = 0) -> list[dict]:
    """Recursively search a nested dict/list for what looks like a list of
    listing objects (heuristic: dicts containing a 'price' or 'classified' key)."""
    if depth > 6:
        return []
    found = []
    if isinstance(data, dict):
        for key, value in data.items():
            if isinstance(value, list) and value and isinstance(value[0], dict):
                sample = value[0]
                if any(k in sample for k in ("price", "classified", "bedroomCount", "id")):
                    found.extend(value)
                else:
                    found.extend(_find_listing_arrays(value, depth + 1))
            elif isinstance(value, (dict, list)):
                found.extend(_find_listing_arrays(value, depth + 1))
    elif isinstance(data, list):
        for item in data:
            found.extend(_find_listing_arrays(item, depth + 1))
    return found


def normalize(raw: dict, postal_code: str, transaction: str) -> Listing:
    """Best-effort normalization -- field names vary depending on which
    extraction strategy produced `raw`, so we try several likely keys."""

    def g(*keys, default=None):
        for k in keys:
            if isinstance(raw, dict) and k in raw and raw[k] is not None:
                return raw[k]
        return default

    price = g("price", "priceValue")
    if isinstance(price, dict):
        price = price.get("mainValue") or price.get("value")

    lat = g("latitude", "lat")
    lng = g("longitude", "lng", "long")
    geo = g("geolocation", "geo")
    if isinstance(geo, dict):
        lat = lat or geo.get("latitude") or geo.get("lat")
        lng = lng or geo.get("longitude") or geo.get("long")

    return Listing(
        id=str(g("id", "classifiedId", default="")) or None,
        title=g("title", "name"),
        property_type=g("propertyType", "type"),
        transaction=transaction,
        price=float(price) if isinstance(price, (int, float)) else None,
        price_text=g("price_text", "displayPrice"),
        locality=g("locality", "city"),
        postal_code=str(g("postalCode", default=postal_code)),
        bedrooms=g("bedroomCount", "bedrooms"),
        living_area_m2=g("netHabitableSurface", "livingArea"),
        latitude=float(lat) if isinstance(lat, (int, float, str)) and str(lat) else None,
        longitude=float(lng) if isinstance(lng, (int, float, str)) and str(lng) else None,
        url=g("url", "cardUrl") or (BASE_URL + raw["url"] if g("url", default="").startswith("/") else g("url")),
        raw=raw,
    )


def scrape_region(postal_codes: list[str], property_type: str, transaction: str, max_pages: int) -> list[Listing]:
    all_listings: list[Listing] = []
    for postal_code in postal_codes:
        print(f"\n=== Postal code {postal_code} ===")
        for page in range(1, max_pages + 1):
            url = build_search_url(postal_code, property_type, transaction, page)
            print(f"  Fetching page {page}: {url}")
            html = fetch(url)
            if not html:
                break
            raw_items = extract_listings_from_html(html)
            if not raw_items:
                print(f"  No listings found on page {page} (end of results, or selectors need updating).")
                break
            for raw in raw_items:
                all_listings.append(normalize(raw, postal_code, transaction))
            print(f"  -> {len(raw_items)} listings extracted")
            time.sleep(random.uniform(*REQUEST_DELAY))
    return all_listings


def main():
    parser = argparse.ArgumentParser(description="Scrape Immoweb listings for a region.")
    parser.add_argument("--postal-codes", required=True, help="Comma-separated postal codes, e.g. 1000,1050")
    parser.add_argument("--type", default="house-and-apartment",
                         help="Property type: house-and-apartment | house | apartment")
    parser.add_argument("--sale", action="store_true", help="Search for-sale listings")
    parser.add_argument("--rent", action="store_true", help="Search for-rent listings")
    parser.add_argument("--max-pages", type=int, default=3, help="Max pages to fetch per postal code")
    parser.add_argument("--out", default="listings.json", help="Output JSON file")
    args = parser.parse_args()

    transaction = "for-rent" if args.rent else "for-sale"
    postal_codes = [p.strip() for p in args.postal_codes.split(",") if p.strip()]

    listings = scrape_region(postal_codes, args.type, transaction, args.max_pages)

    out_data = [asdict(l) for l in listings]
    with open(args.out, "w", encoding="utf-8") as f:
        json.dump(out_data, f, ensure_ascii=False, indent=2)

    print(f"\nSaved {len(out_data)} listings to {args.out}")
    with_coords = sum(1 for l in listings if l.latitude and l.longitude)
    print(f"({with_coords} of them have coordinates and will show on the map)")


if __name__ == "__main__":
    main()
