# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a Thai-language knowledge reference for the Dohome development team covering SAP Material Management (MM) and Financial Accounting (FI) integration — specifically stock costing, material valuation, and goods movement accounting.

## Content Overview

The single document `deep-research-report.md` covers:

- **Stock valuation fundamentals**: Stock = Quantity × Valuation Price; two price control methods — Moving Average Price (V) and Standard Price (S)
- **Material Master**: Key DDIC tables `MARA`, `MARC`, `MARD`, `MBEW` and their fields for valuation
- **Movement types**: 101 (GR for PO), 201 (GI for cost center), 261 (GI for order), 311 (SL transfer), 351 (plant transfer), 551 (scrapping), 601 (GI for delivery)
- **Account determination chain**: Movement Type → Value String → Transaction Key (BSX/WRX/PRD/GBB/KDM/UMB) → G/L Account
- **GR/IR clearing**: Three-way match across PO → GR → Invoice
- **Developer tables**: `MATDOC` (S/4HANA), `MKPF`/`MSEG` (ECC), `BKPF`/`BSEG` (FI), `EKKO`/`EKPO` (PO), `RBKP`/`RSEG` (invoice)
- **BAPIs**: `BAPI_GOODSMVT_CREATE`, `BAPI_MATERIAL_SAVEDATA`, `BAPI_INCOMINGINVOICE_CREATE`, always followed by `BAPI_TRANSACTION_COMMIT`
- **Retail flows**: DC-to-store transfers (stock in transit), physical inventory, backdated postings

## Key Domain Facts

- In S/4HANA, use `MATDOC` instead of `MKPF`/`MSEG`
- Material documents and FI documents have separate number ranges — do not conflate them
- `MBEW-VERPR` (moving average price) changes are not captured by SAP change pointers; track via material documents instead
- Valuation area set at company code level means plant transfers have no FI impact; at plant level they do
