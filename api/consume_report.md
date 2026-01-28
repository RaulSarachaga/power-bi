# Power BI – Report Usage Monitoring (Python)

This repository demonstrates how to use the **Power BI REST API** to retrieve **report usage metrics** (most consumed reports) using **Activity Events** and **Python**.

---

## Overview

Using the **Activity Events API**, you can analyze how Power BI reports are consumed across your organization, including:

* Most viewed reports
* Usage per workspace
* User activity (views)
* Adoption analysis

---

## Prerequisites

Before running the script, ensure you have:

* Power BI tenant with **Audit logs enabled**
* Azure AD **App Registration**
* API permissions:

  * `PowerBI.Read.All`
  * `AuditLog.Read.All`
* Admin consent granted
* Python **3.9+**

---

## Install dependencies

```bash
pip install msal requests pandas
```

---

## Authentication

This implementation uses **Client Credentials Flow** with a **Service Principal**, following Microsoft security recommendations.

---

## Project structure

```text
powerbi-usage-monitoring/
│
├── main.py
├── README.md
└── powerbi_most_consumed_reports.csv
```

---

## main.py

```python
"""
Power BI Report Usage Monitoring
Author: Example
Description:
Reads Power BI Activity Events to identify the most consumed reports.
"""

import msal
import requests
import pandas as pd
from datetime import datetime, timedelta

# ---------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------

TENANT_ID = "<YOUR_TENANT_ID>"
CLIENT_ID = "<YOUR_CLIENT_ID>"
CLIENT_SECRET = "<YOUR_CLIENT_SECRET>"

AUTHORITY = f"https://login.microsoftonline.com/{TENANT_ID}"
SCOPE = ["https://analysis.windows.net/powerbi/api/.default"]

POWER_BI_ACTIVITY_EVENTS_URL = (
    "https://api.powerbi.com/v1.0/myorg/admin/activityevents"
)

# ---------------------------------------------------------------------
# Acquire access token
# ---------------------------------------------------------------------

def get_access_token():
    app = msal.ConfidentialClientApplication(
        CLIENT_ID,
        authority=AUTHORITY,
        client_credential=CLIENT_SECRET
    )

    token_response = app.acquire_token_for_client(scopes=SCOPE)

    if "access_token" not in token_response:
        raise Exception("Failed to acquire access token")

    return token_response["access_token"]

# ---------------------------------------------------------------------
# Retrieve Activity Events
# ---------------------------------------------------------------------

def get_activity_events(headers, start_time, end_time):
    url = (
        f"{POWER_BI_ACTIVITY_EVENTS_URL}"
        f"?startDateTime='{start_time}'"
        f"&endDateTime='{end_time}'"
    )

    events = []

    while url:
        response = requests.get(url, headers=headers)
        response.raise_for_status()

        result = response.json()
        events.extend(result.get("activityEventEntities", []))
        url = result.get("continuationUri")

    return events

# ---------------------------------------------------------------------
# Main execution
# ---------------------------------------------------------------------

def main():
    access_token = get_access_token()

    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }

    # Define time window (last 7 days)
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=7)

    start_time_str = start_time.strftime("%Y-%m-%dT%H:%M:%S")
    end_time_str = end_time.strftime("%Y-%m-%dT%H:%M:%S")

    events = get_activity_events(headers, start_time_str, end_time_str)

    df = pd.DataFrame(events)

    # Filter only report view events
    df_report_views = df[df["Activity"] == "ViewReport"]

    # Select relevant columns
    df_report_views = df_report_views[
        [
            "CreationTime",
            "UserId",
            "ReportId",
            "ReportName",
            "WorkspaceName"
        ]
    ]

    # Aggregate usage
    most_consumed_reports = (
        df_report_views
        .groupby(["WorkspaceName", "ReportName", "ReportId"])
        .size()
        .reset_index(name="ViewCount")
        .sort_values("ViewCount", ascending=False)
    )

    # Output results
    print(most_consumed_reports.head(10))

    most_consumed_reports.to_csv(
        "powerbi_most_consumed_reports.csv",
        index=False
    )

if __name__ == "__main__":
    main()
```

---

## Output example

```text
WorkspaceName   ReportName              ViewCount
Sales           Executive Overview      1245
Finance         Monthly P&L             982
Operations      KPI Dashboard           811
```

---

## Notes

* Activity Events are available for the **last 30 days**
* All timestamps are returned in **UTC**
* Requires **Power BI Administrator** or delegated permissions
* Data is tenant-wide (Admin API)

---

## Limitations

* Does not include dashboard tile interactions
* Does not distinguish Pro vs Premium views
* Near real-time data is not guaranteed

---
