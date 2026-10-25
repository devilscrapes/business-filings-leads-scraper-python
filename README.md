<p align="center"><a href="https://apify.com/DevilScrapes/new-business-filings-leads-scraper"><img src="assets/banner.svg" alt="New Business Filings — US State Registry Leads Feed — Python example" width="100%"></a></p>

# Business Filings Leads Scraper in Python: newly formed US companies by date range

<p align="center">
<a href="https://apify.com/DevilScrapes/new-business-filings-leads-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/business-filings-leads-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/business-filings-leads-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

State filing offices register new LLCs and corporations every business day, and four of them publish those records as open data. This repository is a tested Python client for the **[New Business Filings Leads Scraper](https://apify.com/DevilScrapes/new-business-filings-leads-scraper)** on Apify. Name a start date, optionally an end date and the states you want, and it hands back each company formed in that window: its legal name, the form it took, its standing where the state reports one, a business address, the registered agent and a link to the source record. Apify queries the registries for you; your script ends up with tidy `results.json` and `results.csv` files that pandas reads in one line.

> **Actor:** [New Business Filings — US State Registry Leads Feed](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [business-filings-leads-scraper-javascript](https://github.com/devilscrapes/business-filings-leads-scraper-javascript)

## 🔥 What you get

- **Query by formation date, not by name**: you supply a window with `dateFrom` and an optional `dateTo`, and the Actor asks each state dataset for every entity whose filing date falls inside it, newest first.
- **Four registries, one set of keys**: New York, Colorado, Connecticut and Oregon all label their columns differently; each row comes back with the same twelve fields regardless of state.
- **Oregon rows merged per company**: Oregon publishes one line per associated name, so the Actor groups them by registry number and lifts the principal (or mailing) address and the registered agent into a single record.
- **New York address fallback**: when a New York filer leaves the location columns blank, `principal_address` is filled from the service-of-process address instead of coming back empty.
- **Exact entity-type filter**: `entityTypes` is passed to each state as written, so `DLLC` narrows Colorado and `DOMESTIC LIMITED LIABILITY COMPANY` narrows New York.
- **A state that misbehaves is skipped, not fatal**: throttling and server errors are retried with growing pauses (honouring `Retry-After`), and a state that still errors is logged while the others finish.
- **Provenance on every record**: `source_record_url` queries the exact raw row and `registry_url` opens the state's dataset page.

## 🎯 Use cases

### Measure formation activity week by week

Economists, local development agencies and journalists can count new entities per state and per legal form. Run a long window once for the baseline, then a short one each Monday, and the trend is a single pivot table away.

### Hand sales a list of companies younger than a week

Insurers, accountants, payroll providers and business banks sell best to firms that have just been registered. Pull the last seven days, keep rows with an address and pass `fresh_leads.csv` to the team before the mailbox fills up with competitors' offers.

### Study which legal forms founders choose

Because `entity_type` is kept exactly as each state writes it, you can see the LLC-versus-corporation split, the share of foreign registrations and the use of benefit corporations, then map the raw labels to your own categories in one dictionary.

### Track registered-agent market share

Group recent filings by `registered_agent_name` to see which agent services dominate a state's new formations. It is a quick competitive view for anyone selling to, or competing with, those agents.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/business-filings-leads-scraper-python.git
cd business-filings-leads-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [New Business Filings — US State Registry Leads Feed](https://apify.com/DevilScrapes/new-business-filings-leads-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

If you strip out argument parsing and file export, the integration is just this. `call()` returns after the run finishes, and `iterate_items()` walks the dataset page by page.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/new-business-filings-leads-scraper").call(run_input={'dateFrom': '2026-10-01', 'jurisdictions': ['NY', 'CO'], 'maxResultsPerJurisdiction': 5, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` submits `input.json` unchanged. The sample asks New York and Colorado for companies formed since 1 October 2026 and keeps five from each, so a first run produces at most ten rows. `dateTo` is left out, which makes the Actor use today's UTC date; raise `maxResultsPerJurisdiction` (up to 10,000) once the output looks right.

```json
{
  "dateFrom": "2026-10-01",
  "jurisdictions": [
    "NY",
    "CO"
  ],
  "maxResultsPerJurisdiction": 5,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `dateFrom` | string | yes | `"2026-07-01"` | — |
| `dateTo` | string | no | — | — |
| `jurisdictions` | array | no | `["NY", "CO", "CT", "OR"]` | — |
| `entityTypes` | array | no | — | — |
| `maxResultsPerJurisdiction` | integer | no | `1000` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each record is one entity in one state. `formation_date` is a plain `YYYY-MM-DD` string, and the address fields are flattened to a single line. Expect gaps by state: New York and Oregon do not publish a status, and the Connecticut dataset carries no registered agent.

| Field | Type | Always present |
|---|---|---|
| `entity_id` | string | yes |
| `entity_name` | string | yes |
| `jurisdiction` | string | yes |
| `entity_type` | string or null | no |
| `formation_date` | string | yes |
| `status` | string or null | no |
| `principal_address` | string or null | no |
| `registered_agent_name` | string or null | no |
| `registered_agent_address` | string or null | no |
| `source_record_url` | string | yes |
| `registry_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row emitted | $0.005 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

Two things most users want first, in one pandas script (`pip install pandas`): a weekly count of new entities by state and by a coarse LLC/corporation/other label, and a deduplicated CSV of filings that came with an address, newest first.

```python
import json

import pandas as pd

filings = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
filings["formed"] = pd.to_datetime(filings["formation_date"])
filings["week"] = filings["formed"].dt.to_period("W").astype(str)
filings["kind"] = filings["entity_type"].fillna("").str.upper().map(
    lambda t: "LLC" if "LIMITED LIABILITY" in t or "LLC" in t
    else "Corporation" if "CORP" in t
    else "Other"
)

weekly = filings.pivot_table(index="week", columns=["jurisdiction", "kind"], values="entity_id",
                             aggfunc="count", fill_value=0)
print(weekly.to_string())

leads = (
    filings.dropna(subset=["principal_address"])
    .drop_duplicates(subset=["jurisdiction", "entity_id"])
    .sort_values("formed", ascending=False)
)
leads[["formation_date", "jurisdiction", "entity_name", "kind", "principal_address", "source_record_url"]].to_csv(
    "fresh_leads.csv", index=False
)
print(f"{len(leads)} entities with a mailable address saved to fresh_leads.csv")
```

## ❓ FAQ

### How fresh is the data?

As fresh as each state's open-data copy, which these portals refresh on their own schedule. Treat it as a daily feed. A one- or two-day lag is normal, so a rolling window that overlaps the previous run is safer than a single-day window.

### Why only these four states?

They publish keyless, machine-readable formation datasets maintained from their own filing records. More states can be added as separate adapters without changing the row shape.

### Can I filter to LLCs across all states with one value?

Not with a single label. Look at a sample run, collect the exact `entity_type` spellings each state uses and list all of them in `entityTypes`.

### What happens when the window has no filings?

If every selected state returns zero rows, the run is marked as failed with a status message naming the dates and states, and `main.py` exits with code 1. Widen the window or check that `dateFrom` is not in the future.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [New Business Filings — US State Registry Leads Feed on Apify Store](https://apify.com/DevilScrapes/new-business-filings-leads-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/business-filings-leads-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [UCC Lien Filing Scraper — CT + CO State Portals](https://apify.com/DevilScrapes/ucc-lien-filing-scraper) — Connecticut and Colorado lien filings, a signal that a young company has taken on secured debt.
- [US Building Permit Leads Scraper (Socrata)](https://apify.com/DevilScrapes/us-building-permit-leads-scraper) — permit records for spotting new businesses fitting out premises.
- [Socrata Open Data API Scraper](https://apify.com/DevilScrapes/socrata-open-data-scraper) — any other dataset on these and hundreds more Socrata portals.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with the New York, Colorado, Connecticut or Oregon Secretary of State offices and their open-data portals.
