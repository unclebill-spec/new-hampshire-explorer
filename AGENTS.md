# New Hampshire Explorer — agent handoff

Static Leaflet map for a nurse household, the same shared app as the Kentucky, Tennessee, Massachusetts, Maine, Vermont, Montana, Wyoming, Idaho and Utah Explorers (`app.js`, `areas.js`, `perm.js`, `profiles.js`, `extras.js`, `style.css` are byte-identical copies of `/workspace/kentucky/explorer/*`; another worker edits them there, so run `scripts/sync_shared.sh` right before every build/publish and log any shared-file edit in KY `explorer/AGENTS.md`). `sw.js` differs only in its cache prefix. NH settings live in `explorer/build.py` `STATE` (written by `scripts/port_build.py --restate`, marker PORT_ST; Town wording: `cu` "Town", `cus` "").

Live: https://unclebill-spec.github.io/new-hampshire-explorer/ · repo unclebill-spec/new-hampshire-explorer · progress log: `/workspace/new-hampshire/STATUS.md` (newest first, ET).
Copied from the Utah code base (Oct 4 2026), including Idaho's bath-count fix. Every script in `scripts/` picks the state from the folder it runs in (`scripts/common.py` ST: NH, fips 33, ls prefix `nhx_`) and reads the price cap from `common.CAP` (NH 600000). Copied scripts were checked so that every output path points at /workspace/new-hampshire (the Utah copy once wrote into Idaho's folders).

## Caps (Bill, Oct 4 2026: "Let's do New Hampshire next and take it also to $600,000 for the housing budget on all homes"; other states keep their own caps)
5+ acres $300k–$600k; 1+ acre 3bd/2ba < $600k; near-hospital 1,600+ sqft 3bd/2ba < $600k (townhomes/condos OK, good condition, ≤ 10 min of a hospital with a 10+ bed ER). No cabin category.

## Blocks
259 towns, cities and unincorporated places (Census 2023 county subdivisions, the Vermont method; `scripts/common.py` -> `data/statewide/raw/nh_blocks_500k.zip`). "county" fields in the data hold the town name.

## Data pipeline (run from /workspace/new-hampshire; pandas scripts use /workspace/kentucky/.venv/bin/python, the rest /usr/bin/python3)
1. `scripts/hospitals_research.py` (NH branch: the Oct 1 research file + CMS; 26 NH hospitals + 10 border Level I centers: Maine Medical, UVM, Lahey, MGH, BWH, BIDMC, Tufts, BMC, UMass Worcester, Baystate; CMS lists Concord Hospital's ER as "No", overridden for Level I–III trauma centers) -> `data/nhx_hospitals.json`; `scripts/layers.py` (TOWNS: ACS county-subdivision values, 24 suppressed medians -> county median; Gazetteer centroids; New England trauma scale; RN wages O*NET/BLS OEWS May 2025 by county -> BLS area `data/statewide/raw/oews_area_of.json`; NCES schools + SEDA, towns without a school use the nearest schools).
2. `scripts/appeal_build.py` (OSRM drives, cached; scales (15,60)/(30,120)/(15,60)); `scripts/er_beds.py` -> `data/hospital_er_beds.json` (23 qualifying 10+ bed ERs: 13 NH acute + 10 border).
3. Climate (`climate/build_clim.py` BOX NH, stations above 1200 m excluded = Mount Washington summit) -> `data/clim.json`; activities `data/osm/wd_act.py` + `data/osm/wp_cat_act.py` (+ 7 hand trails).
4. Compare: `cd compare && python3 land.py hillsborough-county-nh merrimack-county-nh strafford-county-nh`, then `compare/areas_build.py` -> `data/areas.json` (Manchester, Nashua, Concord, Dover, Rochester; SEDA LEAIDs in CITIES).
5. Homes: `scripts/zsearch.py` (10 counties × 3 searches at $600k) -> `data/zsearch/`; `scripts/listings_build.py` (PER_COUNTY NH 5 = per town; `MAX_NEW_DETAIL` env; detail cache) -> `listings.json` + `listing-photos/`; bargains run inside build.py; `scripts/top_lists.py`.
6. Permanent RN jobs: `scripts/perm_jobs.py` -> `data/perm_jobs.json`. Sources: Dartmouth Health iCIMS (DHMC, New London) + HealthcareSource (Alice Peck Day, Cheshire, Valley Regional); Workday: SolutionHealth (Elliot, SNHMC), MGB (Wentworth-Douglass), BILH (Exeter), Concord Hospital (Concord, Laconia, Franklin), Covenant (St. Joseph); UKG: Huggins, Monadnock; browser: North Country Healthcare ADP (flaky: rerun `--only nch` and merge if it returns 0), Speare Paycom; HCA last: careers.hcahealthcare.com answers 403, so `data/hca_websearch.json` (web search) is used. Use `PERM_JOBS=0` on publish to keep the merged file.
7. Travel RN jobs: `/workspace/tj_nh` (Vivian + Advantis), then `scripts/travel_jobs.py` (needs explorer/data/data.js).
8. Phase 4: airports (MHT, PSM, LEB + BOS, PWM, BTV), attractions, Crexi for-sale (`data/forsale/st_forsale.py` HAND_DROP/RELABEL), `explorer/fetch_thumbs.py` after a build.
9. Border items: `/workspace/border` (`scripts/static.py NH MA VT ME`, `scripts/make_border.py NH MA VT ME`), copy `out/<ST>.json` -> each `explorer/border.json`. MA/VT/ME are covered maps (homes/jobs both ways, each filtered by the receiving map's caps); Québec is static.
10. Ski areas + peaks: `/workspace/mtn/scripts/make_state.py NH` -> `explorer/mtn.json` + `img/mtn/`.
11. Publish: `sh scripts/sync_shared.sh && cd publish && PATH=/usr/bin:$PATH PERM_JOBS=0 ./publish.sh -m "msg"` (live check share/county-manchester.html).
12. Tests: `perf/smoke.py BASE TAG`, `perf/test_homes.py`, `perf/test_perm.py` (hid elliot-hospital-manchester), `perf/test_p4.py`, `perf/loadtime.py URL`, `perf/sw_check.py URL...`; screenshots in `perf/shots/`.

## Known gaps (Oct 4 2026)
- Homes 306: 5+ ac 112, 1+ ac 160 (cap), near-hospital 34. Only 80 of 370 near-hospital candidates are within 10 min of one of the 23 qualifying 10+ bed ERs (NH critical-access hospitals have smaller ERs); 17 skipped for condition wording.
- Perm RN jobs 588 (537 mapped, 165 list pay). Speare (Paycom) showed 0 postings; North Country Healthcare ADP is partial (31 of 33 read). Not collected: MaineHealth Memorial (North Conway), Cottage Hospital (Cloudflare), Littleton Regional (ADP Workforce Now), New London/Lakes Region small sites outside the readers.
- HCA (Portsmouth Regional, Parkland, Frisbie, Catholic Medical Center) jobs are an 18-posting web-search sample (careers site 403; not circumvented), no pay.
- Travel jobs 268 at 18 hospitals; 9 unmatched postings (VMS Keene/Colebrook, Sullivan County Health Care, D-H Manchester clinic).
- 24 towns with suppressed ACS home values use the county median; 72 towns without their own school use the nearest schools. RN employment counts null (BLS limits).
- Ellacoya State Park has no coordinates; some peaks (Mount Major, Mount Hale, Mount Cube, ...) not built. OSRM times for remote northern unincorporated places are long (up to 314 min).
- Shared app.js still says "about 19% list pay" in the perm panel (KY figure); county history layer empty.

### 50+ acre lots under $250k (Oct 4, 2026 ~10:31 AM ET, big-land worker)
- Black-star layer `big-land` (50+ ac, < $250k, land or home), "50+ ac" button, Map key row, card; shared code from the KY explorer (see KY explorer/AGENTS.md, same date). build.py (marker BIGLAND) merges `/workspace/new-hampshire/bigland.json`.
- Refresh: `/usr/bin/python3 /workspace/bigland/bigland.py NH --refresh` before build/publish (keeps the old file if Zillow blocks). Notes: /workspace/bigland/PROGRESS.md.
