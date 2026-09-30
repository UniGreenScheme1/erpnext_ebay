# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`erpnext_ebay` is a Frappe/ERPNext application that integrates ERPNext with eBay. It syncs orders, customers, payments, and listings between eBay's REST APIs and ERPNext's DocType system.

## Commands

**Install app into bench:**
```bash
bench get-app erpnext_ebay
bench --site <site> install-app erpnext_ebay
```

**Run tests:**
```bash
bench run-tests --app erpnext_ebay
bench run-tests --app erpnext_ebay --doctype "eBay Order"
```

**Lint:**
```bash
pylint erpnext_ebay/  # uses .pylintrc at repo root
```

**Development server:**
```bash
bench start  # starts web, socketio, scheduler, workers per Procfile
```

**Migrate after DocType changes:**
```bash
bench --site <site> migrate
```

## Architecture

This app follows standard Frappe app conventions. The integration point is `hooks.py`, which wires everything into Frappe's event system.

### Core Sync Pipeline

1. **`sync_orders_rest.py`** (largest module, ~1950 lines) — main synchronisation engine. Fetches completed eBay orders via the Sell Fulfillment REST API and creates ERPNext Sales Invoices, Customers, and Addresses. Also handles refunds and return items.

2. **`ebay_get_requests.py`** — constructs eBay API request objects (order details, transactions, listings).

3. **`ebay_requests_rest.py`** — thin wrappers calling the `ebay_rest` SDK: `get_order`, `get_orders`, `get_transactions`.

4. **`ebay_tokens.py`** — manages eBay OAuth token lifecycle (refresh, storage in ERPNext).

5. **`sync_mp_transactions.py`** — syncs eBay Managed Payments transactions and matches refunds to existing orders.

### Supporting Modules

- **`ebay_categories.py`** — syncs eBay category hierarchy into ERPNext
- **`ebay_constants.py`** — constants and eBay→ERPNext field mappings
- **`country_data.py`** — marketplace-specific country code mappings (~927 lines)
- **`tasks.py`** — scheduled task entry points (daily: category sync, shipping carrier sync, log cleanup after 120 days)

### Custom DocType Extensions

- **`custom_classes/sales_invoice.py`** — extends ERPNext Sales Invoice with eBay-specific calculations and validations
- **`custom_methods/`** — DocType event handlers for Item, Customer, and Sales Invoice

### Platform Abstraction

**`online_selling/`** — base classes (`base.py`, `platform_ebay.py`) designed to support multiple marketplaces. Currently only eBay is implemented.

### DocTypes

Key custom DocTypes: eBay Manager, eBay Manager Settings (holds API credentials), eBay Order, eBay Pending Order, eBay Sync Log, eBay Shipping Carrier, Online Selling Platform, Online Selling Item.

## Key Conventions

- **Background jobs:** Sync operations are enqueued via Frappe's background job system (Redis + workers). Do not call sync functions synchronously from web requests.
- **API credentials:** Stored in eBay Manager Settings DocType — never hardcoded.
- **Frappe patterns:** Use `frappe.get_doc`, `frappe.get_value`, `frappe.db.get_value` for data access. DocType changes require `bench migrate`.
- **Customization permissions:** When a customization file in `erpnext_ebay/erpnext_ebay/custom/` is changed and its `custom_perms` list becomes empty (`[]`) where it previously held entries, flag this to the user. It is likely due to the permissions being exported incorrectly (e.g. 'Export custom permissions' left unticked in Customize Form) rather than a deliberate removal. On migrate, an empty list leaves a site's existing Custom DocPerms untouched, so the loss goes unnoticed, whereas a non-empty list replaces all Custom DocPerms for that DocType.
- **Python version:** Requires Python >= 3.13.
- **Dependencies:** Declared in `pyproject.toml` (flit_core build system). No `requirements.txt`.
