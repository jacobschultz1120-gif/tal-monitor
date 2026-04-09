"""
tal_monitor.py — Target Account List Monitor
────────────────────────────────────────────────────────────────────────────
Watches 304 named accounts for any signal — news, job postings, leadership
changes, funding, expansion, or website content changes.

Two monitoring layers:
  1. Google News RSS — checks for any news mentioning each company
     (free, no API key, runs every 60 minutes)
  2. Website change detection — fingerprints careers and newsroom pages,
     alerts when content changes
     (free, runs every 4 hours)

All alerts go to a single dedicated Discord channel: #accounts
High-urgency signals (score 8+) trigger an @here mention.
Territory rules do NOT apply — every account on the list is monitored.

Runs continuously on Railway.
"""

import os
import re
import sys
import json
import time
import hashlib
import logging
from datetime import datetime, timezone, timedelta
from email.utils import parsedate_to_datetime
from urllib.parse import quote_plus

import feedparser
import requests
import anthropic

from config import (
    POLL_INTERVAL_SECONDS,
    WEBSITE_CHECK_INTERVAL_SECONDS,
    URGENT_SCORE_THRESHOLD,
    NEWS_RESULTS_PER_COMPANY,
    NEWS_LOOKBACK_HOURS,
    ACCOUNTS,
)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger(__name__)

# ─────────────────────────────────────────────────────────────────────────────
# DEDUPLICATION — file-backed, 7-day memory
# ─────────────────────────────────────────────────────────────────────────────

SEEN_FILE = "/tmp/tal_monitor_seen.json"
_seen: dict = {}
_SEEN_TTL = timedelta(days=7)


def _load_seen():
    global _seen
    if not os.path.exists(SEEN_FILE):
        _seen = {}
        return
    try:
        with open(SEEN_FILE) as f:
            raw = json.load(f)
        cutoff = datetime.now(timezone.utc) - _SEEN_TTL
        _seen = {k: v for k, v in raw.items()
                 if datetime.fromisoformat(v) > cutoff}
        log.info(f"Loaded {len(_seen)} seen entries from disk")
    except Exception as e:
        log.warning(f"Could not load seen file: {e} — starting fresh")
        _seen = {}


def _save_seen():
    try:
        with open(SEEN_FILE, "w") as f:
            json.dump(_seen, f)
    except Exception as e:
        log.warning(f"Could not save seen file: {e}")


def is_new(uid: str) -> bool:
    if uid in _seen:
        return False
    _seen[uid] = datetime.now(timezone.utc).isoformat()
    _save_seen()
    return True


# ─────────────────────────────────────────────────────────────────────────────
# WEBSITE FINGERPRINTING
# ─────────────────────────────────────────────────────────────────────────────

FINGERPRINT_FILE = "/tmp/tal_website_fingerprints.json"
_fingerprints: dict = {}


def _load_fingerprints():
    global _fingerprints
    if not os.path.exists(FINGERPRINT_FILE):
        _fingerprints = {}
        return
    try:
        with open(FINGERPRINT_FILE) as f:
            _fingerprints = json.load(f)
        log.info(f"Loaded {len(_fingerprints)} website fingerprints")
    except Exception as e:
        log.warning(f"Could not load fingerprints: {e}")
        _fingerprints = {}


def _save_fingerprints():
    try:
        with open(FINGERPRINT_FILE, "w") as f:
            json.dump(_fingerprints, f)
    except Exception as e:
        log.warning(f"Could not save fingerprints: {e}")


def fingerprint_page(url: str) -> str | None:
    """
    Fetch a page and return a hash of its meaningful text content.
    Strips scripts, styles, nav, footer, and other boilerplate before hashing
    so minor layout changes don't trigger false alerts.
    """
    try:
        headers = {
            "User-Agent": (
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/122.0.0.0 Safari/537.36"
            )
        }
        r = requests.get(url, headers=headers, timeout=15)
        if r.status_code != 200:
            return None

        html = r.text

        # Strip noise
        html = re.sub(r"<script[^>]*>.*?</script>", "", html, flags=re.DOTALL)
        html = re.sub(r"<style[^>]*>.*?</style>",   "", html, flags=re.DOTALL)
        html = re.sub(r"<nav[^>]*>.*?</nav>",        "", html, flags=re.DOTALL)
        html = re.sub(r"<footer[^>]*>.*?</footer>",  "", html, flags=re.DOTALL)
        html = re.sub(r"<header[^>]*>.*?</header>",  "", html, flags=re.DOTALL)
        html = re.sub(r"<[^>]+>", " ", html)
        html = re.sub(r"\s+", " ", html).strip()

        # Only hash meaningful content (first 5000 chars of cleaned text)
        content = html[:5000]
        return hashlib.sha256(content.encode()).hexdigest()

    except Exception:
        return None


def get_page_urls(account: dict) -> dict[str, str]:
    """Return careers and newsroom URLs to monitor for a given account."""
    base = account.get("website", "").rstrip("/")
    if not base:
        return {}

    return {
        "careers":  f"{base}/careers",
        "newsroom": f"{base}/news",
    }


# ─────────────────────────────────────────────────────────────────────────────
# GOOGLE NEWS
# ─────────────────────────────────────────────────────────────────────────────

def fetch_google_news(company_name: str) -> list[dict]:
    """
    Search Google News RSS for a company name.
    Returns list of article dicts published within NEWS_LOOKBACK_HOURS.
    Free — no API key required.
    """
    query = quote_plus(f'"{company_name}"')
    url   = f"https://news.google.com/rss/search?q={query}&hl=en-US&gl=US&ceid=US:en"

    try:
        headers = {"User-Agent": "Mozilla/5.0 (compatible; TerritoryMonitor/1.0)"}
        r = requests.get(url, headers=headers, timeout=15)
        r.raise_for_status()
        parsed = feedparser.parse(r.content)
    except Exception as e:
        log.debug(f"  Google News error ({company_name}): {e}")
        return []

    cutoff  = datetime.now(timezone.utc) - timedelta(hours=NEWS_LOOKBACK_HOURS)
    articles = []

    for entry in parsed.entries[:NEWS_RESULTS_PER_COMPANY]:
        # Parse date
        pub = None
        if hasattr(entry, "published"):
            try:
                pub = parsedate_to_datetime(entry.published)
            except Exception:
                pass

        if pub and pub < cutoff:
            continue

        title  = getattr(entry, "title", "")
        link   = getattr(entry, "link",  "")
        source = getattr(entry, "source", {})
        source_name = source.get("title", "Google News") if isinstance(source, dict) else "Google News"

        uid = f"gn-{company_name}-{link}"
        if not is_new(uid):
            continue

        articles.append({
            "title":       title,
            "url":         link,
            "source":      source_name,
            "published":   pub.strftime("%b %d %Y %H:%M UTC") if pub else "Recent",
            "company":     company_name,
            "signal_type": "news",
        })

    return articles


# ─────────────────────────────────────────────────────────────────────────────
# AI SCORING
# ─────────────────────────────────────────────────────────────────────────────

SCORE_PROMPT = """\
You are a sales intelligence analyst for a NetSuite ERP account executive.

The rep has a named account list of target companies they want to sell NetSuite to.
These are pre-qualified accounts — territory and industry have already been confirmed.
Your job is to evaluate whether a signal (news article or website change) represents
a meaningful buying opportunity or a reason to reach out NOW.

REVENUE SWEET SPOT: $0–$20M annual revenue.
AUTO-SCORE LOW (1–2): Clearly routine PR, award/recognition with no operational change,
unrelated acquisition, or something involving a completely different company with a
similar name.

SCORING FRAMEWORK:

Score 9–10 — Act today:
  • New CFO, Controller, VP Finance, or finance leadership hire
  • Acquisition or merger — consolidation need is immediate
  • First significant government contract or enterprise partnership
  • Funding announcement with operational mandate (Series A+)

Score 7–8 — Strong signal, reach out this week:
  • New location, facility, or geographic expansion
  • New product line or channel launch (DTC, ecommerce, wholesale)
  • Leadership change (CEO, COO, President)
  • Website careers page now shows finance or ERP-related job

Score 5–6 — Moderate, worth monitoring:
  • General growth announcement or milestone
  • New hire in operations, supply chain, or IT (not finance)
  • Partnership or distribution agreement

Score 3–4 — Weak, probably routine:
  • Award, recognition, or certification
  • Industry association news
  • Event sponsorship or speaking engagement

Score 1–2 — Ignore:
  • Clearly unrelated to this company
  • Pure vanity PR with no operational change
  • Different company with similar name

Return ONLY valid JSON — no markdown, no backticks:

{
  "score": <integer 1–10>,
  "reason": <one sentence — why this is or isn't worth acting on>,
  "signal_type": <"exec_change", "funding", "acquisition", "expansion",
                  "new_contract", "product_launch", "job_posting",
                  "website_change", "leadership", "general_news", or "irrelevant">,
  "urgency": <"today", "this_week", "monitor", or "ignore">,
  "suggested_angle": <one sentence — how to open the conversation if score is 6+,
                      otherwise null>
}

COMPANY: {company_name}
INDUSTRY: {industry}
SIGNAL:
"""


def score_signal(account: dict, signal: dict) -> dict:
    api_key = os.environ.get("ANTHROPIC_API_KEY", "")
    if not api_key:
        signal.update({
            "ai_score": 5, "ai_reason": "AI unavailable",
            "signal_type": "general_news", "urgency": "monitor",
            "suggested_angle": None,
        })
        return signal

    prompt = SCORE_PROMPT.format(
        company_name=account["name"],
        industry=f"{account.get('industry','')} — {account.get('subindustry','')}",
    )

    content = signal.get("content", signal.get("title", ""))
    text    = f"HEADLINE/CHANGE: {signal.get('title','')}\n\nDETAIL: {content[:800]}"

    try:
        client   = anthropic.Anthropic(api_key=api_key)
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=400,
            messages=[{"role": "user", "content": prompt + text}],
        )
        raw = response.content[0].text.strip()
        raw = re.sub(r"^```[a-z]*\n?", "", raw)
        raw = re.sub(r"\n?```$", "", raw)
        parsed = json.loads(raw)

        signal.update({
            "ai_score":       int(parsed.get("score", 0)),
            "ai_reason":      str(parsed.get("reason", "—")),
            "signal_type":    str(parsed.get("signal_type", "general_news")),
            "urgency":        str(parsed.get("urgency", "monitor")),
            "suggested_angle": parsed.get("suggested_angle"),
        })
        log.info(
            f"  [{signal['ai_score']}/10] {account['name']} — "
            f"{signal.get('title','change')[:60]}"
        )

    except Exception as e:
        log.warning(f"  AI error ({account['name']}): {e}")
        signal.update({
            "ai_score": 5, "ai_reason": f"AI error: {str(e)[:60]}",
            "signal_type": "general_news", "urgency": "monitor",
            "suggested_angle": None,
        })

    return signal


# ─────────────────────────────────────────────────────────────────────────────
# DISCORD ALERT
# ─────────────────────────────────────────────────────────────────────────────

SIGNAL_COLORS = {
    "exec_change":    0x7F77DD,  # purple
    "funding":        0xEF9F27,  # amber
    "acquisition":    0xE24B4A,  # red
    "expansion":      0x1D9E75,  # green
    "new_contract":   0x378ADD,  # blue
    "product_launch": 0xD85A30,  # coral
    "job_posting":    0x378ADD,  # blue
    "website_change": 0x888780,  # gray
    "leadership":     0x7F77DD,  # purple
    "general_news":   0x888780,  # gray
    "irrelevant":     0x888780,  # gray
}

URGENCY_LABELS = {
    "today":     "🔴  Act today",
    "this_week": "🟡  Act this week",
    "monitor":   "🔵  Monitor",
    "ignore":    "⬜  Low signal",
}

def score_bar(n: int) -> str:
    return f"`{'█' * n}{'░' * (10 - n)}`"


def send_alert(account: dict, signal: dict) -> bool:
    webhook = os.environ.get("DISCORD_TAL_WEBHOOK_URL", "")
    if not webhook:
        log.error("DISCORD_TAL_WEBHOOK_URL not set")
        return False

    score   = signal.get("ai_score", 0)
    urgency = signal.get("urgency", "monitor")
    stype   = signal.get("signal_type", "general_news")
    color   = SIGNAL_COLORS.get(stype, 0x888780)

    # @here for urgent signals
    mention = "@here\n" if score >= URGENT_SCORE_THRESHOLD else ""

    fields = [
        {
            "name":   f"Score  {score}/10  •  {URGENCY_LABELS.get(urgency, urgency)}",
            "value":  score_bar(score),
            "inline": False,
        },
        {
            "name":   "Why this matters",
            "value":  signal.get("ai_reason", "—"),
            "inline": False,
        },
        {
            "name":   "Company",
            "value":  f"{account['name']}  •  {account.get('city','')}, {account.get('state','')}",
            "inline": True,
        },
        {
            "name":   "Industry",
            "value":  f"{account.get('industry','')} — {account.get('subindustry','')}",
            "inline": True,
        },
        {
            "name":   "Signal type",
            "value":  stype.replace("_", " ").title(),
            "inline": True,
        },
        {
            "name":   "Source",
            "value":  signal.get("source", "—") + "  •  " + signal.get("published", "—"),
            "inline": True,
        },
    ]

    angle = signal.get("suggested_angle")
    if angle:
        fields.append({
            "name":   "Suggested outreach angle",
            "value":  angle,
            "inline": False,
        })

    if account.get("website"):
        fields.append({
            "name":   "Company website",
            "value":  account["website"],
            "inline": False,
        })

    embed = {
        "title":       f"🎯  {account['name']}  —  {stype.replace('_',' ').title()}",
        "url":         signal.get("url", ""),
        "color":       color,
        "description": mention + signal.get("title", "Website content change detected"),
        "fields":      fields,
        "footer":      {"text": "TAL Monitor — Named Accounts"},
        "timestamp":   datetime.now(timezone.utc).isoformat(),
    }

    try:
        r = requests.post(
            webhook,
            json={"username": "TAL Monitor", "embeds": [embed]},
            timeout=10,
        )
        r.raise_for_status()
        return True
    except requests.RequestException as e:
        log.error(f"  Discord failed: {e}")
        return False


# ─────────────────────────────────────────────────────────────────────────────
# HEARTBEAT
# ─────────────────────────────────────────────────────────────────────────────

_last_heartbeat: datetime | None = None


def maybe_heartbeat():
    global _last_heartbeat
    now = datetime.now(timezone.utc)
    # Fire once per day at 15:12 UTC (8:12 AM Pacific)
    if not (now.hour == 15 and now.minute < 18):
        return
    if _last_heartbeat and (now - _last_heartbeat).total_seconds() < 23 * 3600:
        return

    webhook = os.environ.get("DISCORD_TAL_WEBHOOK_URL", "")
    if not webhook:
        return

    embed = {
        "title":  "🟢  TAL Monitor — Daily Check-in",
        "color":  0x1D9E75,
        "description": f"Running normally. Watching {len(ACCOUNTS)} named accounts.",
        "fields": [
            {"name": "Status",    "value": "✅  Running continuously",          "inline": False},
            {"name": "Accounts",  "value": f"{len(ACCOUNTS)} companies on watch list", "inline": True},
            {"name": "News check","value": "Every 60 minutes via Google News",  "inline": True},
            {"name": "Web check", "value": "Every 4 hours (careers + newsroom)","inline": True},
            {"name": "Check-in",  "value": now.strftime("%A %B %d %Y  •  %H:%M UTC"), "inline": False},
        ],
        "footer":    {"text": "TAL Monitor — Named Accounts  •  Daily heartbeat"},
        "timestamp": now.isoformat(),
    }

    try:
        r = requests.post(webhook,
            json={"username": "TAL Monitor", "embeds": [embed]}, timeout=10)
        r.raise_for_status()
        _last_heartbeat = now
        log.info("TAL heartbeat sent")
    except Exception as e:
        log.warning(f"Heartbeat failed: {e}")


# ─────────────────────────────────────────────────────────────────────────────
# NEWS SWEEP
# ─────────────────────────────────────────────────────────────────────────────

def run_news_sweep() -> int:
    """Check Google News for all 304 accounts. Returns number of alerts sent."""
    log.info(f"News sweep — {len(ACCOUNTS)} accounts")
    sent = 0
    for i, account in enumerate(ACCOUNTS, 1):
        articles = fetch_google_news(account["name"])
        for article in articles:
            scored = score_signal(account, article)
            if scored.get("ai_score", 0) >= 4:  # show anything 4+
                if send_alert(account, scored):
                    sent += 1
        # Polite delay between companies
        if i % 10 == 0:
            log.info(f"  Progress: {i}/{len(ACCOUNTS)} accounts checked")
            time.sleep(2)
        else:
            time.sleep(0.5)
    log.info(f"News sweep complete — {sent} alerts sent")
    return sent


# ─────────────────────────────────────────────────────────────────────────────
# WEBSITE CHANGE SWEEP
# ─────────────────────────────────────────────────────────────────────────────

def run_website_sweep() -> int:
    """Check careers and newsroom pages for all accounts. Returns alerts sent."""
    log.info(f"Website sweep — {len(ACCOUNTS)} accounts")
    sent = 0
    for account in ACCOUNTS:
        if not account.get("website"):
            continue

        pages = get_page_urls(account)
        for page_type, url in pages.items():
            key        = f"{account['name']}::{page_type}"
            new_hash   = fingerprint_page(url)
            if new_hash is None:
                continue  # page not accessible

            old_hash = _fingerprints.get(key)
            if old_hash is None:
                # First time seeing this page — store fingerprint, no alert
                _fingerprints[key] = new_hash
                _save_fingerprints()
                continue

            if new_hash != old_hash:
                # Page changed — alert
                _fingerprints[key] = new_hash
                _save_fingerprints()

                signal = {
                    "title":    f"{page_type.title()} page updated",
                    "content":  f"The {page_type} page at {url} has changed since last check.",
                    "url":      url,
                    "source":   "Website monitor",
                    "published": datetime.now(timezone.utc).strftime("%b %d %Y %H:%M UTC"),
                    "signal_type": "website_change",
                }
                scored = score_signal(account, signal)
                if scored.get("ai_score", 0) >= 4:
                    if send_alert(account, scored):
                        sent += 1

        time.sleep(0.3)

    log.info(f"Website sweep complete — {sent} alerts sent")
    return sent


# ─────────────────────────────────────────────────────────────────────────────
# MAIN LOOP
# ─────────────────────────────────────────────────────────────────────────────

def validate_env() -> bool:
    if not os.environ.get("DISCORD_TAL_WEBHOOK_URL"):
        log.error("DISCORD_TAL_WEBHOOK_URL is not set — add it in Railway Variables")
        return False
    if not os.environ.get("ANTHROPIC_API_KEY"):
        log.error("ANTHROPIC_API_KEY is not set — add it in Railway Variables")
        return False
    return True


def main():
    import signal as _signal
    def _shutdown(sig, frame):
        log.info("Shutdown signal received — exiting cleanly")
        raise SystemExit(0)
    _signal.signal(_signal.SIGTERM, _shutdown)
    _signal.signal(_signal.SIGINT,  _shutdown)

    log.info("=" * 60)
    log.info("TAL Monitor starting up")
    log.info(f"Watching {len(ACCOUNTS)} named accounts")
    log.info(f"News sweep:    every {POLL_INTERVAL_SECONDS // 60} minutes")
    log.info(f"Website sweep: every {WEBSITE_CHECK_INTERVAL_SECONDS // 60} minutes")
    log.info("=" * 60)

    if not validate_env():
        sys.exit(1)

    _load_seen()
    _load_fingerprints()

    last_website_sweep = datetime.now(timezone.utc) - timedelta(hours=5)  # run soon

    cycle = 0
    while True:
        cycle += 1
        now = datetime.now(timezone.utc)
        log.info(f"─── Cycle {cycle} ──────────────────────────────")

        try:
            run_news_sweep()
        except Exception as e:
            log.error(f"News sweep error: {e}", exc_info=True)

        # Website sweep runs every WEBSITE_CHECK_INTERVAL_SECONDS
        if (now - last_website_sweep).total_seconds() >= WEBSITE_CHECK_INTERVAL_SECONDS:
            try:
                run_website_sweep()
                last_website_sweep = now
            except Exception as e:
                log.error(f"Website sweep error: {e}", exc_info=True)

        try:
            maybe_heartbeat()
        except Exception as e:
            log.warning(f"Heartbeat error: {e}")

        log.info(f"Sleeping {POLL_INTERVAL_SECONDS // 60} min until next news sweep…\n")
        try:
            time.sleep(POLL_INTERVAL_SECONDS)
        except SystemExit:
            log.info("Exiting cleanly")
            return


if __name__ == "__main__":
    main()
