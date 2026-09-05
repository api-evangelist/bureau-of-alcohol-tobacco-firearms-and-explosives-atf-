---
name: find-atf-federal-firearms-licensees
description: >-
  Locate Federal Firearms Licensees or ATF offices by state, city, ZIP, licence type or map extent
  using ATF's public ArcGIS feature services. Use for "which FFLs are in X", "where is the nearest
  ATF field division", or bulk export of the licensee location dataset.
api: ATF Federal Firearm Licensees locations (ArcGIS Feature Service)
base_url: https://services6.arcgis.com/PrP5ZtrES07DmVmv/arcgis/rest/services
auth: none
operations:
  - query_federal_firearms_licensees
  - query_atf_offices
---

# Find Federal Firearms Licensees and ATF offices

Two public Esri feature services, both anonymous, both query-only. There is no OpenAPI here — the
service descriptor is the contract, and it is saved verbatim in this repository under
`geoservices/`. Field names below come from that descriptor, not from guesswork.

## Steps

1. **Size the answer before fetching it.**
   `GET .../Federal_Firearm_Licensees_locations/FeatureServer/0/query?where=1%3D1&returnCountOnly=true&f=json`
   Returns `{"count": 77514}` for the whole layer. Add your `where` clause first and re-count — this
   is one cheap request that tells you whether you are about to page 39 times or once.

2. **Filter by attribute.** The business identity lives in the `USER_*` columns copied from ATF's own
   FFL listing:
   `USER_LICENSE_NAME`, `USER_BUSINESS_NAME`, `USER_PREMISE_STREET`, `USER_PREMISE_CITY`,
   `USER_PREMISE_STATE`, `USER_PREMISE_ZIP_CODE`, `USER_VOICE_PHONE`, and the licence number split
   across `USER_LIC_REGN`, `USER_LIC_DIST`, `USER_LIC_CNTY`, `USER_LIC_TYPE`, `USER_LIC_SEQN`.
   Example: `where=USER_PREMISE_STATE='TX' AND USER_LIC_TYPE=1`
   Everything else in the 90 fields (`Match_addr`, `Score`, `Addr_type`, `StPreDir`, …) is Esri
   geocoder output describing how well the address resolved — it describes the geocode, not the
   licensee. Do not present `Score` as a quality rating of the business.

3. **Page.** `maxRecordCount` is **2000**. Use `resultOffset` and `resultRecordCount`, and check
   `exceededTransferLimit` on every page. Request only the fields you need with `outFields` rather
   than `*`.

4. **Ask for GeoJSON when mapping.** `f=geojson` is accepted; `f=json` returns the Esri envelope.
   Coordinates are EPSG:4269 (NAD83).

5. **ATF offices** use the same grammar against
   `.../ATF_Office_Locations/FeatureServer/0/query` — 537 records, 9 fields
   (`user_office_type`, `user_office_name`, `user_address`, `user_city`, `user_state`,
   `user_field_division`, `user_zipcode`, `globalid`). It fits in one request. Group by
   `user_field_division` to roll offices up to their division.

## Rules

- **Errors arrive inside HTTP 200.** ArcGIS returns `{"error":{"code":400,"message":"..."}}` with a
  200 status. Check for an `error` key on every response before using it.
- **Reassemble the licence number yourself** if you need to join to another ATF source: the layer
  stores it decomposed across five integer columns and never stores the printed 15-character form.
- **These are addresses of real businesses, and of individuals where the licensee operates from
  home.** ATF publishes them lawfully, but say plainly what you are doing before bulk-exporting
  77,514 premises records, and do not enrich them with anything ATF did not publish.
- Data currency: the FFL layer was last edited **2026-06-17**, offices **2026-04-16**. Neither is
  live; state the date with the answer.
