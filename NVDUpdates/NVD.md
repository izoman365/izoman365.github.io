---
layout: default
---

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

```

# The Scripts

The following is the script to pull from the NVD and putting it into the CSV

```bash
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


This is the script to get the summary made

```bash
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
