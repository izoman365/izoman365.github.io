---
layout: default
---
[Back To Main Page](../index.md)


I created a script to call on my google home to pull CVEs from the National Vulnerability Datbase with a CVSS score of 7 or higher to put into a CSV to be summarized by Gemini and dictated to me through my google home speaker



# Setting up ssh on my server

I needed to make sure ssh was ready to go
```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
sudo ufw allow ssh
```


# Installing python

I need python pip installed to get the libraries I need to get the script to run correctly. 
I cannot get them all installed normally so I had to make a virtual environment as well

```bash
sudo apt install python3-pip
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
pip install requests
pip install pychromecast
pip install pandas
pip install google-generativeai
pip install gTTS

```

# The Scripts

The following is the script to pull from the NVD and putting it into the CSV

```python
import requests
import csv
from datetime import datetime, timedelta, timezone

# --- time window (last 24h) ---
now = datetime.now(timezone.utc)
yesterday = now - timedelta(days=1)

start_date = yesterday.strftime("%Y-%m-%dT%H:%M:%S.000")
end_date = now.strftime("%Y-%m-%dT%H:%M:%S.000")

url = (
    "https://services.nvd.nist.gov/rest/json/cves/2.0"
    f"?pubStartDate={start_date}&pubEndDate={end_date}"
)

response = requests.get(url)
data = response.json()

results = []

for item in data.get("vulnerabilities", []):
    cve = item["cve"]

    try:
        metrics = cve["metrics"]["cvssMetricV31"][0]["cvssData"]
        score = metrics["baseScore"]
    except:
        continue

    if score >= 7.0:
        results.append({
            "cve_id": cve["id"],
            "score": score,
            "description": cve["descriptions"][0]["value"]
        })

# --- write CSV ---
with open("cve_report.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["cve_id", "score", "description"])
    writer.writeheader()
    writer.writerows(results)

print(f"Saved {len(results)} CVEs")
```


This is the script to get the summary made and create a txt document as well as a mp3 file to send to the google nest

```python
import requests
import csv
import pandas as pd
from datetime import datetime, timedelta, timezone

from google import genai
from gtts import gTTS
import os

# ----------------------------
# CONFIG
# ----------------------------
API_KEY = "YOUR_GEMINI_API_KEY"

# ----------------------------
# 1. GET CVEs FROM NVD (LAST 24 HOURS)
# ----------------------------
now = datetime.now(timezone.utc)
yesterday = now - timedelta(days=1)

start_date = yesterday.strftime("%Y-%m-%dT%H:%M:%S.000")
end_date = now.strftime("%Y-%m-%dT%H:%M:%S.000")

url = (
    "https://services.nvd.nist.gov/rest/json/cves/2.0"
    f"?pubStartDate={start_date}&pubEndDate={end_date}"
)

response = requests.get(url)
data = response.json()

results = []

for item in data.get("vulnerabilities", []):
    cve = item["cve"]

    try:
        metrics = cve["metrics"]["cvssMetricV31"][0]["cvssData"]
        score = metrics["baseScore"]
    except:
        continue

    if score >= 7.0:
        results.append({
            "cve_id": cve["id"],
            "score": score,
            "description": cve["descriptions"][0]["value"]
        })

# ----------------------------
# 2. SAVE RAW CSV
# ----------------------------
with open("cve_report.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["cve_id", "score", "description"])
    writer.writeheader()
    writer.writerows(results)

df = pd.DataFrame(results)
text_block = df.to_string(index=False)

# ----------------------------
# 3. GEMINI SUMMARY
# ----------------------------
client = genai.Client(api_key=API_KEY)

prompt = f"""
You are a cybersecurity analyst.

Write a concise SOC morning briefing.

Focus on:
- Critical vulnerabilities
- Remote code execution risks
- Likely exploitation
- Keep it suitable for spoken audio (no bullet spam)

Data:
{text_block}
"""

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=prompt
)

summary = response.text

# ----------------------------
# 4. BUILD SOC REPORT (FORMATTED TXT)
# ----------------------------
now_str = datetime.now().strftime("%Y-%m-%d %H:%M UTC")

report = f"""
========================================
SOC MORNING CYBER THREAT REPORT
========================================

Date: {now_str}
Source: NVD API (Last 24 Hours)
Generated: Automated Python CVE Pipeline

----------------------------------------
EXECUTIVE SUMMARY
----------------------------------------
{summary}

----------------------------------------
PRIORITY ACTION ITEMS
----------------------------------------
- Review CVEs with CVSS ≥ 9 immediately
- Prioritize remotely exploitable vulnerabilities
- Check if affected systems are exposed externally
- Validate against CISA KEV catalog if applicable

----------------------------------------
END OF REPORT
========================================
"""

# OVERWRITE TXT FILE
with open("cve_briefing.txt", "w", encoding="utf-8") as f:
    f.write(report)

# ----------------------------
# 5. TEXT-TO-SPEECH (MP3)
# ----------------------------
tmp_mp3 = "cve_briefing.mp3.tmp"
final_mp3 = "cve_briefing.mp3"

tts = gTTS(text=summary, lang="en")
tts.save(tmp_mp3)

# atomic replace (safer overwrite)
os.replace(tmp_mp3, final_mp3)

print("Report generated successfully.")
print("TXT: cve_briefing.txt")
print("MP3: cve_briefing.mp3")
```
# Creating the Web Service

I needed a way to host my mp3 so that the Google Nest could grab it
I set up the server to be able to share the mp3 through html for it to be grabbed as a system service to run on boot
```bash
sudo nano /etc/systemd/system/cve-http.service

[Unit]
Description=CVE HTTP Server
After=network.target

[Service]
WorkingDirectory=/home/youruser/cve_project
ExecStart=/usr/bin/python3 -m http.server 8000
Restart=always
User=youruser

[Install]
WantedBy=multi-user.target

```
Then enabled the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable cve-http.service
sudo systemctl start cve-http.service
```

#Cron Jobs

I need to make cron jobs to have the server update the csv with new info daily and update the txt and mp3 summaries

```bash
0 7 * * * /media/izo/Izo-FileShare/AptShare/NVD/venv/bin/python /media/izo/Izo-FileShare/AptShare/NVD/NVD.py
0 7 * * * /media/izo/Izo-FileShare/AptShare/NVD/venv/bin/python /media/izo/Izo-FileShare/AptShare/NVD/Gemi-Summary.py
```




#Next Steps: Working Getting the mp3 to play on the google nest speaker






