# Doctoreto City Scraper (server-side)

The Linux/server half of the doctor-directory work: an exhaustive single-city scraper for
doctoreto.com that walks every listing page, downloads each doctor's photo, and emits an
RTL Excel workbook plus CSV and a printable PDF per city. Written to run as `root` on the
author's Debian VPS (`<server>`) under `/root/doctoreto-scraper`, where it produced the
12-city export set (ahvaz, ardabil, isfahan, karaj, mashhad, qom, rasht, shiraz, tabriz,
tehran, urmia, zanjan).
This folder holds the script together with one finished city - Tabriz, 2,580 doctors.

**Suggested repo name:** `doctoreto-city-scraper`
**Stack:** Python 3 (stdlib `urllib`/`ssl` + `concurrent.futures`), Pillow, XlsxWriter, pandas, headless Chromium for PDF
**Status:** finished
**Last modified:** 2026-09-06

## What it does

- `scrape_tabriz.py` - the whole program. Fetches
  `https://doctoreto.com/search?citySlug=<slug>&page=N` and parses the Next.js `__NEXT_DATA__`
  JSON, pulling the doctor list out of the React-Query `dehydratedState` (`queryKey ['doctors','list']`)
  to read `pagination.lastPage` / `pagination.total`.
- Fetches all remaining pages with a 30-worker `ThreadPoolExecutor`, 3 retries per page,
  4-second socket timeout, and a desktop Chrome `User-Agent` with `fa,en-US` accept-language.
- De-duplicates on `profile_link` (falling back to `name + phone`).
- `download_and_convert_images()` - pulls each `profile_image_url` and re-encodes it to a real
  JPEG in `exports/<city>/images/<doctor name>.jpg`.
- `export_excel_and_csv()` - XlsxWriter workbook, RTL, headers
  `ردیف | عکس | نام پزشک | تخصص | آدرس مطب | تلفن | لینک پروفایل`, with the doctor's photo
  **embedded in the عکس column** (0.35 scale), alongside a CSV twin.
- `export_pdf_cli()` - renders a temporary HTML sheet and calls
  `chromium --headless --no-sandbox --print-to-pdf=...`; a missing Chromium is caught and logged,
  so PDF is best-effort and Excel/CSV always land.
- Caching: if `cache.json` already exists for the city it is loaded instead of re-fetching, then
  rewritten after the image pass - interrupting and rerunning costs one page-1 request.

## Layout

```
scrape_tabriz.py     the scraper (Linux paths, run as root)
cache_tabriz.json    scraped record store: name, specialty, address, phone,
                     profile_image_url, profile_link, local_image_path  (2,580 records)
doctors_tabriz.xlsx  generated Excel deliverable with embedded photos (9.3 MB)
```

`cache_tabriz.json` and `doctors_tabriz.xlsx` are **output**, not source - regenerate them rather
than editing them. On the server everything is written under
`/root/doctoreto-scraper/exports/<city>/`.

## Running it

Intended for the Linux box (needs `chromium` on PATH and write access to `/root`):

```bash
python3 scrape_tabriz.py
```

To scrape a different city, edit the `TABRIZ_INFO` dict at the top - `name` (Persian),
`slug`, `query` (`search?citySlug=<slug>`) - the rest of the script is city-agnostic. SSL
verification is disabled (`CERT_NONE`) deliberately.

## Notes

- Local snapshot, not the live deployment: the copy that produced the 12 cities runs at
  `/root/doctoreto-scraper` on the author's VPS. Hence `finished` rather than `active`.
- Downstream consumer: `../doctoreto-scraper` (the merge + import side). Its `AHURA_PIPELINE.md`
  documents these exports as the input, and `import_server_cities.py` reads exactly the
  `doctors_<city>.xlsx` / `images/` / `cache.json` shape this script writes, then upserts them
  into the clinic directory API on visital.ir.
- Superseded for *license-number* work: doctoreto exposes no شماره نظام پزشکی, so
  `scrape_license_specialty.py` and `doctoreto_scraper.py` in the sibling folder are the current
  entry points. Comments were also deliberately left out here - the server is a loaded 1-core
  box, so `scraper_comments_server.py` runs locally instead.
