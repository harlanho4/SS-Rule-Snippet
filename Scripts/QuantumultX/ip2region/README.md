# Quantumult X ip2region GeoIP

Cloudflare Workers + private R2 XDB database; IPv4 and IPv6 support, using the upstream [ip2region](https://github.com/lionsoul2014/ip2region) data (Apache-2.0).

## Quantumult X

```ini
geo_location_checker = https://qx-ip2region.haonan-h6709.workers.dev/geo, https://qx-ip2region.haonan-h6709.workers.dev/qx.js
```

The `/geo` endpoint obtains the actual proxy egress IP from Cloudflare's `CF-Connecting-IP` header. It does not accept arbitrary user-supplied IP addresses. The result format is compatible with Orz-3 IP.js: `query, country, regionName, city, isp, org, countryCode`. The `org` field remains empty because ip2region XDB does not provide a separate data-center organization field.

## Architecture

- Worker: `qx-ip2region` at `qx-ip2region.haonan-h6709.workers.dev`
- R2: private Standard bucket `qx-ip2region-xdb`
- Daily scheduled update: `0 3 * * *` (UTC)
- Origin: official `data/ip2region_v4.xdb` and `data/ip2region_v6.xdb` from ip2region's `master` branch
- Query: cache the 512 KiB XDB vector index per Worker isolate; fetch only requested segment-index records and region strings using R2 range requests
- Updates: metadata SHA comparison, staged object uploads, header/size/sample query validation, manifest switch after both versions validate
- Failure: preserves previous active manifest; old version files remain in R2 for manual rollback.

## Routes

| Path | Function |
| --- | --- |
| `GET /geo` | GeoIP information for the connected client's public IP |
| `GET /qx.js` | Quantumult X geo_location_checker display script |
| `GET /health` | Initialization state, last successful update and current upstream blob hashes |
| `GET /bootstrap` | One-time initialization only (inert after first successful initialization) |

The geo checker request needs to reach this Worker over the Quantumult X selected proxy to detect the proxy egress IP.

## Deployment

The code is in [worker.js](worker.js). Deploy it as a module Worker with an R2 binding named `IP_DB` referring to `qx-ip2region-xdb`, set compatibility date to at least `2026-11-04`, enable `workers.dev` and set a Cron Trigger to `0 3 * * *`. The deployed Worker was initially uploaded using the Cloudflare API; updating GitHub source alone does not automatically update the deployed Worker. The repository directory is the source of truth for subsequent changes.

To start initial synchronization, visit the `/bootstrap` route once. It initiates synchronization asynchronously; the scheduled trigger is also able to perform initialization. Check `/health` for both active SHA values.

**Operational caveats:** The request must be routed through the selected proxy. IP geolocation may be approximate for hosting providers. For GDPR/privacy-conscious configurations do not persist request IP logs. XDB object size is ~11 MB IPv4 and ~37 MB IPv6; both are streamed into R2 during updates to avoid storing whole files in memory. R2 range requests increase Class B operation usage. Always monitor Cloudflare Worker errors and schedule failures.

## Attribution

© ip2region authors. XDB data and reference design are provided by [lionsoul2014/ip2region](https://github.com/lionsoul2014/ip2region) under Apache-2.0.
