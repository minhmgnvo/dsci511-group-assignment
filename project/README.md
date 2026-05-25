# F1 Data Collection Pipeline

A data collection project for Formula 1 telemetry and performance data, built for **DSCI-511**.

## Overview

Formula 1 is one of the most data-intensive sports in the world — a single car generates millions of data points per lap. This project aggregates F1 data from multiple public sources using REST APIs and web scraping to build a unified dataset for analysis.

## Data Sources

### 1. OpenF1 API
- **URL:** https://openf1.org
- Free, open REST API — no authentication required
- Provides real-time and historical F1 telemetry (speed, throttle, brake pressure, tire data, etc.)

### 2. Web Scraping
- **Sources:** Wikipedia race result tables, https://www.statsf1.com
- Collects circuit metadata, race entry lists, and constructor/chassis info per season

## Collection Methods

| Method        | Source            | Tool/Library         |
|---------------|-------------------|----------------------|
| REST API      | openf1.org        | `requests`           |
| Web Scraping  | Wikipedia, statsf1| `BeautifulSoup`      |


## 