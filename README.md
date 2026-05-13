# Inventory Check

Mobile-friendly single-page web app for running rental yard inventory checks from a phone or tablet.

**Version:** v1.0.0
**Created by:** CJ Rutter

## What it does

- Accepts two kinds of CSV input (auto-detected by columns):
  - **Assets** — keyed by Asset ID, organized by Equipment Class (cat class). Mark each Accounted / Missing / Damaged with optional note.
  - **Bulk Inventory** — keyed by Part #, with Bin Location prominent. Enter a counted quantity and status auto-derives Match / Short / Over (or override manually).
- Search across asset #, part #, bin, make/model/serial, description.
- Filter chips (All / Unchecked / Accounted / Missing / Damaged — labels switch to Match/Short/Over on the Bulk tab).
- Progress auto-saves per tab in the browser (`localStorage`), so reloads or accidental closes don't wipe your check.
- Export:
  - Download CSV (active tab, both tabs, or filtered scopes like "Exceptions").
  - Email summary via `mailto:` (opens device mail app with a short summary in the body).
- Feedback button sends comments to cj.rutter@equipmentshare.com with app version + device info appended.

## How to use

1. Open [`index.html`](index.html) in any modern browser. No server needed.
2. Tap **Load CSV** and pick either `Inventory check.csv` or `Bulk INV.csv` (sample files included in this repo). Loading a second CSV populates the other tab.
3. Walk the yard, tap status buttons / enter counts.
4. Tap **Export** when done.

## Sample data

Sample CSVs are intentionally **not** committed to this repo because real exports contain serial numbers, OEC, and rate data. Use your own export from the inventory system at runtime — data stays on the device.

## Versioning

The current version is displayed in the footer and the feedback email subject. Bump `APP_VERSION` in `index.html` and tag the release (`git tag vX.Y.Z`) when shipping changes.
