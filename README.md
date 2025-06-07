#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Liquipedia ‑ VCT Masters Toronto 일정 → iCalendar(.ics)
작성자 : YourName (youremail@example.com)
라이선스 : MIT
"""

import datetime
import os
import sys
import requests
from zoneinfo import ZoneInfo        # Python 3.9+
from ics import Calendar, Event

# 1) 사용자 설정 --------------------------------------------------
EVENT_SLUG = "VCT_2025:Masters_Toronto"   # Liquipedia 페이지 이름(수정)
OUTPUT_ICS = "vct_masters_toronto.ics"    # 결과 파일명
TZ_LOCAL   = ZoneInfo("Asia/Seoul")       # 표기할 시간대

HEADERS = {
    "User-Agent": (
        "VCTMastersTorontoCalendar/1.0 "
        "(https://github.com/yourid/vct-masters-toronto; "
        "contact: youremail@example.com)"
    )
}

# 2) Liquipedia Cargo API ----------------------------------------
BASE = "https://liquipedia.net/valorant/api.php"
QUERY = (
    "action=cargoquery"
    "&tables=Matches"
    "&fields=Matches.DateTime_UTC,Matches.Team1,Matches.Team2,Matches.Round,"
    "Matches.Bracket,"
    "Matches.OverviewPage"
    f"&where=Matches.OverviewPage%3D%22{EVENT_SLUG}%22"
    "&limit=500"
    "&order_by=Matches.DateTime_UTC"
    "&format=json"
)

def fetch_matches() -> list[dict]:
    url = f"{BASE}?{QUERY}"
    r = requests.get(url, headers=HEADERS, timeout=15)
    r.raise_for_status()
    return [row["title"] for row in r.json().get("cargoquery", [])]

# 3) iCalendar 변환 ----------------------------------------------
UTC = ZoneInfo("UTC")
DATE_FMT = "%Y-%m-%d %H:%M:%S"

def row_to_event(row: dict) -> Event:
    ev = Event()

    # 경기 시작 시각
    utc_dt = datetime.datetime.strptime(row["DateTime_UTC"], DATE_FMT).replace(tzinfo=UTC)
    local_dt = utc_dt.astimezone(TZ_LOCAL)

    ev.begin = local_dt
    ev.duration = datetime.timedelta(hours=2)   # 평균 경기 시간(임의값), 필요 시 조정
    ev.name = f"{row['Team1']} vs {row['Team2']}"
    ev.location = "Toronto, Canada"
    ev.description = f"Round: {row.get('Round','')}, Bracket: {row.get('Bracket','')}"
    ev.url = f"https://liquipedia.net/valorant/{EVENT_SLUG}"
    return ev

def build_calendar() -> Calendar:
    cal = Calendar()
    for row in fetch_matches():
        cal.events.add(row_to_event(row))
    return cal

# 4) 메인 ---------------------------------------------------------
def main() -> None:
    cal = build_calendar()
    with open(OUTPUT_ICS, "w", encoding="utf-8") as fp:
        fp.writelines(cal)
    print(f"[✓] {OUTPUT_ICS} 생성 — 경기 {len(cal.events)}건")

if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        print("오류:", e, file=sys.stderr)
        sys.exit(1)
