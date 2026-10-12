# ⚡ Shopify Catalog & Inventory Management Toolkit (GraphQL)

A focused, enterprise-grade Python automation utility using Shopify's Admin GraphQL API to resolve catalog collisions and accidental import runs.

This tool is specifically engineered around two core operations:

1. **Smart Duplicate SKU Resolver:** Automatically detects duplicate product listings that share identical SKUs across your catalog. It identifies the original, older product to safeguard its established SEO rankings, customer reviews, custom tags, and pricing, while cleanly purging the newer duplicate entries.
2. **Time-Range Batch Cleanup & Rollback:** Safely queries and bulk-deletes products created within an accidental bulk-import window or erroneous sync session, rolling back bad imports in seconds without affecting the rest of your store.

---

## 🚀 Key Modules Included

1. **Smart Duplicate SKU Resolver:** Detects duplicate variants across different products, preserves the oldest listing (keeping original tags, pricing, and SEO URLs), and purges newer duplicates.
2. **Time-Range Batch Cleanup & Rollback:** Safely isolates and deletes products created/imported during an accidental import window or specific date range.

---

## 🛠️ Prerequisites & Setup

### 1. Requirements

* Python 3.8+
* Install required packages:
```bash
python -m pip install requests pandas

```



### 2. Shopify Custom App Setup

1. In your Shopify Admin, navigate to **Settings > Apps and sales channels > Develop apps**.
2. Click **Create an app** (e.g., `Catalog Cleaner`).
3. Under **Configuration > Admin API integration**, grant the following scopes:
* `read_products`, `write_products`


4. Click **Install app** and copy your **Client ID** and **Client Secret** from the API credentials tab.

### 3. Store Configuration

Configure your credentials directly at the top of each script file:

```python
SHOP_NAME = "YOUR_STORE_SUBDOMAIN"  # e.g., "my-store" if url is my-store.myshopify.com
CLIENT_ID = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
API_VERSION = "2024-07"

```

---

## 📑 Feature Guides & Scripts

### Module 1: Smart Duplicate SKU Resolver (Preserving Originals)

#### The Problem

When importing multiple supplier catalog feeds or CSV files, duplicate products sharing identical SKUs are frequently generated. Deleting products at random can destroy older items that already have traffic, sales history, customer reviews, and optimized tags.

#### The Safe Resolution Flow

1. **Index & Group:** Scans all variants and groups them by `SKU`.
2. **Chronological Sorting:** If multiple products share a SKU, they are sorted by `createdAt`.
* **Oldest Product:** Kept untouched (preserves legacy tags, prices, and SEO).
* **Newer Products:** Flagged for cleanup.


3. **Audit Trail Export:** Always run with `DRY_RUN = True` first to export `duplicate_products_to_delete.csv`. Review the file before taking action.

#### Script: `find_and_remove_duplicate_skus.py`

```python
import sys
import time
import requests
import pandas as pd
from datetime import datetime

# ==============================================================================
# 1. CONFIGURATION
# ==============================================================================
SHOP_NAME = "YOUR_STORE_SUBDOMAIN"
CLIENT_ID = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
API_VERSION = "2024-07"

# SAFETY TOGGLE: Keep True first to generate the audit CSV
DRY_RUN = True
ACTION_MODE = "DELETE"  # Options: "DELETE" or "DRAFT"

GRAPHQL_URL = f"https://{SHOP_NAME}.myshopify.com/admin/api/{API_VERSION}/graphql.json"

# ==============================================================================
# 2. AUTHENTICATION
# ==============================================================================
def get_access_token():
    auth_url = f"https://{SHOP_NAME}.myshopify.com/admin/oauth/access_token"
    payload = {
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "grant_type": "client_credentials"
    }
    resp = requests.post(auth_url, json=payload, headers={"Content-Type": "application/json"}).json()
    if "access_token" not in resp:
        sys.exit(f"[!] Authentication failed: {resp}")
    return resp["access_token"]

token = get_access_token()
headers = {"Content-Type": "application/json", "X-Shopify-Access-Token": token}

QUERY_VARIANTS = """
query getVariants($cursor: String) {
  productVariants(first: 250, after: $cursor) {
    pageInfo { hasNextPage endCursor }
    edges {
      node {
        sku
        product { id title handle createdAt status }
      }
    }
  }
}
"""

DELETE_MUTATION = """
mutation deleteProduct($input: ProductDeleteInput!) {
  productDelete(input: $input) {
    deletedProductId
    userErrors { field message }
  }
}
"""

# ==============================================================================
# 3. SCANNING & AUDIT
# ==============================================================================
print("[*] Scanning catalog for duplicate SKUs...")
sku_map = {}
cursor = None
has_next = True

while has_next:
    res = requests.post(GRAPHQL_URL, json={"query": QUERY_VARIANTS, "variables": {"cursor": cursor}}, headers=headers).json()
    edges = res.get("data", {}).get("productVariants", {}).get("edges", [])
    for edge in edges:
        node = edge["node"]
        raw_sku = (node.get("sku") or "").strip()
        prod = node.get("product")
        if not raw_sku or not prod:
            continue
        if raw_sku not in sku_map:
            sku_map[raw_sku] = {}
        sku_map[raw_sku][prod["id"]] = prod

    pinfo = res.get("data", {}).get("productVariants", {}).get("pageInfo", {})
    has_next = pinfo.get("hasNextPage", False)
    cursor = pinfo.get("endCursor")
    time.sleep(0.05)

audit_rows = []
to_remove = {}

for sku, prods in sku_map.items():
    if len(prods) > 1:
        sorted_prods = sorted(
            prods.values(),
            key=lambda x: datetime.fromisoformat(x["createdAt"].replace("Z", "+00:00"))
        )
        kept = sorted_prods[0]
        dupes = sorted_prods[1:]
        for d in dupes:
            audit_rows.append({
                "SKU": sku,
                "Kept_ID": kept["id"],
                "Kept_Title": kept["title"],
                "Kept_Created": kept["createdAt"],
                "Delete_ID": d["id"],
                "Delete_Title": d["title"],
                "Delete_Created": d["createdAt"]
            })
            to_remove[d["id"]] = d

if audit_rows:
    pd.DataFrame(audit_rows).to_csv("duplicate_products_to_delete.csv", index=False)
    print(f"[+] Audit saved: 'duplicate_products_to_delete.csv' ({len(to_remove)} products to clean).")

if DRY_RUN:
    print("\n[i] DRY_RUN is Active. No products were deleted. Check CSV, then set DRY_RUN = False.")
    sys.exit(0)

# ==============================================================================
# 4. EXECUTION
# ==============================================================================
print(f"\n[*] Executing {ACTION_MODE} on {len(to_remove)} duplicate items...")
for i, (pid, pinfo) in enumerate(to_remove.items(), 1):
    res = requests.post(GRAPHQL_URL, json={"query": DELETE_MUTATION, "variables": {"input": {"id": pid}}}, headers=headers).json()
    time.sleep(0.06)
    print(f"[{i}/{len(to_remove)}] Deleted duplicate: {pinfo['title'][:40]}...")

print("[+] Cleanup complete.")

```

---

### Module 2: Time-Range Batch Cleanup & Rollback

#### The Problem

Accidents happen during catalog management:

* A supplier CSV was imported with broken tags or incorrect prices.
* An automated sync app created unintended products between a specific time interval.
* You need to purge **only the products created or updated during that specific accident window** without touching the rest of your catalog.

#### The Safe Deletion Strategy

1. **Define Strict UTC Boundaries:** Shopify stores timestamps in ISO 8601 UTC format (e.g., `2026-10-12T09:00:00Z`).
2. **Use Server-Side Search Filtering:** Instead of scanning every product manually, query Shopify using date filters directly:
* Query syntax: `created_at:>=2026-10-12T09:00:00Z AND created_at:<={END_TIME_UTC}`


3. **Audit Preview Mode:** The script previews all matched products and writes `timerange_deletion_preview.csv` before running any deletions.

#### Script: `delete_products_by_timerange.py`

```python
import sys
import time
import requests
import pandas as pd

# ==============================================================================
# 1. CONFIGURATION
# ==============================================================================
SHOP_NAME = "YOUR_STORE_SUBDOMAIN"
CLIENT_ID = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
API_VERSION = "2024-07"

# Set your target time window in ISO-8601 UTC format
START_TIME_UTC = "2026-10-12T09:00:00Z"
END_TIME_UTC   = "2026-10-12T11:30:00Z"

# SAFETY TOGGLE: Set to False only after reviewing the generated preview CSV
DRY_RUN = True

GRAPHQL_URL = f"https://{SHOP_NAME}.myshopify.com/admin/api/{API_VERSION}/graphql.json"

# ==============================================================================
# 2. AUTHENTICATION
# ==============================================================================
auth_url = f"https://{SHOP_NAME}.myshopify.com/admin/oauth/access_token"
token_resp = requests.post(auth_url, json={
    "client_id": CLIENT_ID,
    "client_secret": CLIENT_SECRET,
    "grant_type": "client_credentials"
}, headers={"Content-Type": "application/json"}).json()

if "access_token" not in token_resp:
    sys.exit(f"[!] Authentication failed: {token_resp}")

headers = {"Content-Type": "application/json", "X-Shopify-Access-Token": token_resp["access_token"]}

# ==============================================================================
# 3. SERVER-SIDE SEARCH QUERY
# ==============================================================================
SEARCH_QUERY = f"created_at:>={START_TIME_UTC} AND created_at:<={END_TIME_UTC}"

PRODUCTS_BY_DATE_QUERY = """
query getProductsByTimeRange($query: String!, $cursor: String) {
  products(first: 250, query: $query, after: $cursor) {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        id
        title
        handle
        createdAt
        status
      }
    }
  }
}
"""

PRODUCT_DELETE_MUTATION = """
mutation deleteProduct($input: ProductDeleteInput!) {
  productDelete(input: $input) {
    deletedProductId
    userErrors {
      field
      message
    }
  }
}
"""

print(f"[*] Searching for products created between {START_TIME_UTC} and {END_TIME_UTC}...")
matched_products = []
cursor = None
has_next = True

while has_next:
    res = requests.post(GRAPHQL_URL, json={
        "query": PRODUCTS_BY_DATE_QUERY,
        "variables": {"query": SEARCH_QUERY, "cursor": cursor}
    }, headers=headers).json()

    data = res.get("data", {}).get("products", {})
    edges = data.get("edges", [])

    for edge in edges:
        matched_products.append(edge["node"])

    pinfo = data.get("pageInfo", {})
    has_next = pinfo.get("hasNextPage", False)
    cursor = pinfo.get("endCursor")
    time.sleep(0.05)

print(f"[+] Total products matching time range: {len(matched_products)}")

if not matched_products:
    print("[!] No products found in this time range. Exiting.")
    sys.exit(0)

# Export safety audit CSV
df_preview = pd.DataFrame(matched_products)
df_preview.to_csv("timerange_deletion_preview.csv", index=False)
print("[+] Preview list exported to 'timerange_deletion_preview.csv'.")

# ==============================================================================
# 4. DRY-RUN VERIFICATION
# ==============================================================================
if DRY_RUN:
    print("\n" + "=" * 60)
    print("DRY-RUN MODE ACTIVE: NO PRODUCTS WERE DELETED")
    print(f"Matched {len(matched_products)} products.")
    print("Inspect 'timerange_deletion_preview.csv'.")
    print("When ready to delete permanently, set DRY_RUN = False and rerun.")
    print("=" * 60)
    sys.exit(0)

# ==============================================================================
# 5. PERMANENT DELETION EXECUTION
# ==============================================================================
print(f"\n[*] Deleting {len(matched_products)} products from accidental batch...")
deleted = 0
failed = 0

for i, prod in enumerate(matched_products, 1):
    m_res = requests.post(GRAPHQL_URL, json={
        "query": PRODUCT_DELETE_MUTATION,
        "variables": {"input": {"id": prod["id"]}}
    }, headers=headers).json()

    errors = m_res.get("data", {}).get("productDelete", {}).get("userErrors", [])
    if errors:
        print(f"[!] Error on '{prod['title']}': {errors[0]['message']}")
        failed += 1
    else:
        deleted += 1
        if deleted % 25 == 0 or i == len(matched_products):
            print(f"[+] Deleted {deleted} / {len(matched_products)} products...")

    time.sleep(0.06)

print("\n" + "=" * 60)
print("BATCH DELETION COMPLETED")
print(f"Successfully Deleted : {deleted}")
print(f"Failed               : {failed}")
print("=" * 60)

```

---

## 🔒 Security & Safe Usage Checklist

* **Keep Credentials Secure:** Do not share your Client Secret or publish scripts with hardcoded credentials to public locations.
* **Always Run in `DRY_RUN = True` First:** Verify the generated CSV file before toggling `DRY_RUN = False`.
* **Verify Timezones:** Shopify GraphQL timestamps use UTC (`Z`). Convert your local store time to UTC when setting `START_TIME_UTC` and `END_TIME_UTC`.
* **Rate Limits:** The scripts use built-in delays (`time.sleep(0.06)`) to stay well within Shopify's standard GraphQL leaky-bucket rate limits.

---

## 📄 License

This project is licensed under the MIT License.
