from fastapi import FastAPI, HTTPException, Query
from datetime import datetime, timezone
from urllib.parse import urlparse
import sqlite3
import yt_dlp

app = FastAPI(title="YouTube MP4 API")

DB = "requests.db"
DAILY_LIMIT = 10000

def init_db():
    with sqlite3.connect(DB) as db:
        db.execute("""
            CREATE TABLE IF NOT EXISTS usage (
                day TEXT PRIMARY KEY,
                count INTEGER NOT NULL
            )
        """)

def check_limit():
    day = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    with sqlite3.connect(DB, timeout=30) as db:
        db.execute("BEGIN IMMEDIATE")
        row = db.execute(
            "SELECT count FROM usage WHERE day = ?", (day,)
        ).fetchone()

        count = row[0] if row else 0

        if count >= DAILY_LIMIT:
            db.rollback()
            raise HTTPException(429, "Daily limit reached")

        db.execute("""
            INSERT INTO usage(day, count) VALUES(?, 1)
            ON CONFLICT(day) DO UPDATE SET count = count + 1
        """, (day,))
        db.commit()

@app.on_event("startup")
def startup():
    init_db()

@app.get("/")
def home():
    return {"status": "online", "daily_limit": DAILY_LIMIT}

@app.get("/api")
def get_video(url: str = Query(...)):
    host = (urlparse(url).hostname or "").lower()

    if host not in ("youtube.com", "www.youtube.com",
                    "m.youtube.com", "youtu.be", "www.youtube-nocookie.com"):
        raise HTTPException(400, "Only YouTube links are supported")

    check_limit()

    options = {
        "quiet": True,
        "no_warnings": True,
        "noplaylist": True,
        "format": "best[ext=mp4]",
    }

    try:
        with yt_dlp.YoutubeDL(options) as ydl:
            info = ydl.extract_info(url, download=False)

        if info.get("ext") != "mp4" or not info.get("url"):
            raise HTTPException(404, "MP4 format not available")

        return {
            "success": True,
            "title": info.get("title"),
            "format": "mp4",
            "download_url": info["url"],
            "note": "The link may expire."
        }

    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(400, "Could not process this video")
