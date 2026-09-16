# Security

## The threat model changed without the code changing

DewFront was designed and reviewed against a loopback threat model, and the code said so explicitly and correctly. The backend binds to loopback only. The write routes were reasoned about as reachable only from the local network. Comments in the source recorded that reasoning.

Then one commit put the app behind a public tunnel, and every one of those assumptions became false in the same moment. No application code changed. No test failed. Nothing in the deployment looked different from the inside.

**Public ingress is a change to the threat model, not to the deployment.** None of the first three findings below required a new bug. All three were written into the design correctly, against a world that stopped existing the day the tunnel came up.

An audit run the same day found three high severity issues, all live on the public internet. The two found later, below them, are not tunnel findings at all: they are the same control-at-one-surface mistake reappearing on surfaces nobody had counted, first a log file and then a reader's cookie jar.

## The findings

### Server side request forgery with host control

Webhook registration accepted any http or https URL, with no authentication and no private range check. The container runs on the host's network namespace, so the outbound request left from the host itself, with its own loopback, the local subnet and the tailnet all in reach. It was attacker triggerable on demand: register a target, then register a location whose forecast forces the insight that fires the webhook.

**Fixed by** requiring the write credential to register at all, and checking the resolved destination against a blocklist covering loopback, RFC1918, link local (including the cloud metadata address), carrier grade NAT, IPv6 unique local and link local, and the IPv4 in IPv6 forms (`::ffff:`, NAT64, 6to4) that smuggle a private address inside a public looking one.

**Two details that make the fix hold:**

- The destination is resolved and re checked **before every delivery**, not once at registration. Checking once is defeated by DNS rebinding: register a name that answers publicly, then answer loopback afterwards.
- The delivery refuses redirects. With redirects followed, a single `302` pointing at a private address undoes every check above it.

### Unauthenticated station ingest with no time bound

The station upload endpoint never validated the passkey its own protocol documents, and the timestamp parser trusted the value in the body with no future bound. One request dated four years ahead would pin a forged reading as the latest observation, permanently, because "latest" was a comparison the forged value always won.

**Fixed by** requiring the passkey, and bounding the timestamp. The endpoint still answers `200 ok` whether accepted or refused, deliberately: consumer station firmware has no error handling, so a 401 would present to the owner as a station that had gone offline. The refusal is a log line, and the endpoint returns no signal to anyone guessing the passkey.

### Full precision coordinates served to anyone

The location listing returned raw stored coordinates, including 15 decimal browser geolocation fixes belonging to real visitors, because the summary read path was persisting caller coordinates as a side effect of caching.

**Fixed by** rounding coordinates to three decimals, about 110 metres, on the way out, and deduping registration on the same rounding. The endpoint lists places, not people.

### The same coordinates, written to the access log

Found 2026-09-14, and it is the more instructive half of the finding above. Rounding on the way out fixed what the API **returned**. It did nothing about what the web server **recorded**, and nobody had counted the log as somewhere the data went.

Every `/api/` call on this app carries the reader's coordinates as query parameters, and nginx's default `main` format writes the full request line including the query string, alongside `$remote_addr` and `$http_x_forwarded_for`. So every page load appended a line pairing a location accurate to well under a millimetre with the address that asked for it:

```
"GET /api/summary/daily?lat=42.469519663759044&lon=-76.46970321488456" ... "<ip>"
```

The finding above was closed and the class was not. A control applied at one surface reads as a property of the system, and the log is a surface almost nothing audits.

**Fixed by** a dedicated log format for `/api/`: method, normalised path with no query string, status, size. No address, no forwarded address, no user agent, no referer. Enough to count traffic and catch a 5xx, nothing that identifies a reader or where they are. Verified live by requesting a fifteen decimal coordinate and reading back a log line that contained none of it.

The app now records its own coverage metric instead, snapped to the ~11 km grid the guest summary cache already used, emitted with nothing beside it. A cell is a town, not a person, and the granularity is pinned by a test so a future cache tuning cannot sharpen it into a location trail as a side effect.

### An analytics cookie the privacy notice said did not exist

Found 2026-09-16, three days after the privacy notice shipped, and it is the same lesson as the access log with the surface moved again.

The notice said "no cookies and no third-party trackers", and added that there was no consent banner because nothing was stored on the reader's device. The site had been loading Google Analytics on both public hostnames since long before, and GA4 sets `_ga` and `_ga_<id>` with a two year expiry by default. Both cookies were live on every real visit.

Nothing in the repository was wrong in isolation. The analytics block was correct code doing exactly what its comment said, gated on hostname so local and CI runs never polluted the property. The notice was correct prose about a design nobody had checked the code against. The two were written three days apart, in different files, by someone reasoning about each on its own, and **prose is not executed**, so no gate compared them.

**Fixed by** denying `analytics_storage` through consent mode before the loader runs, which stops the cookies while page views still count. The tests that now hold it are in the app repository: one asserts the notice and the analytics block agree, another that consent is denied before the loader, since consent set afterwards is consent granted for the first hit and the first hit is the one that writes the cookie.

**The instructive part is the fix that did not work.** The first attempt set `client_storage: 'none'`, shipped it, and the site went on setting both cookies with the option live in the served page. `client_storage` is a Universal Analytics parameter: GA4's gtag accepts it, raises no error and ignores it. That is the worst available shape for a privacy control, because it reads as correct in review and is wrong only in a browser. It was caught by a probe that drives a real browser at the deployed site and reads `document.cookie`, and by nothing else. Four variants measured at the real origin settled it:

```
client_storage: 'none'                     -> _ga, _ga_0PHZLR4GN4
consent default analytics_storage denied   -> no cookies
both                                       -> no cookies
no analytics at all          (the control) -> no cookies, no Google hosts
```

The dead option was then removed rather than kept beside the working one. A setting that does nothing is worse than no setting, because the next reader believes it.

## The controls now

| Surface | Control |
|---|---|
| Location registration | Bearer token, fails closed when unconfigured |
| Webhook registration | Bearer token, plus destination blocklist |
| Webhook delivery | Destination re resolved and re checked per delivery, redirects refused |
| Station ingest | Shared passkey in the body, timestamp bounded, constant response |
| Location listing | Coordinates rounded on output |
| Access log | Query string and caller address never recorded on `/api/` |
| Coverage metric | ~11 km grid cell only, with no identifier beside it |
| Analytics | `analytics_storage` denied before the tag loads, so no cookie and no client id is ever written. Page views count; returning visitors and sessions do not exist. Held by a test that fails if the privacy notice and the analytics code disagree, and by a probe that reads `document.cookie` on the deployed site |
| Webhook listing | Origins only, the full URL is never returned |
| All reads | Open by design. The whole site is public weather data |
| Personal station reads | Operator key held server side, never returned; station ids normalised and length capped before they reach an upstream URL |
| Pollen reads | Google key held server side, never returned. A non 2xx from the upstream is never read or surfaced, because Google echoes the failing request back and the request carries the key. Coordinates are rounded to a grid before they reach the client, so the cache key space is not whatever precision a caller chose to send |
| Transport | HSTS on the tunnel, rendered through an nginx `map` so the plain HTTP LAN path asserts nothing untrue |
| Service worker | Allowlist, not denylist: the worker can only store the shell files named at build time, and does not intercept `/api/` or any upstream. Served `Cache-Control: no-store` from its own nginx block so the edge never holds a copy, with two kill switches, `?sw=off` for one reader and a build flag that ships a self-unregistering worker to every reader |
| Errors | `{error, detail}` with no path, no upstream URL and no environment value |

Both credential gates fail closed. With the variable unset, the routes they guard refuse everything rather than falling open, so a deploy that forgets a secret leaves a working read only site rather than an open write surface. The boot log reports whether each gate is configured and never the values.

The two FEATURE keys, Weather Underground and Google Pollen, fail closed in the other sense: absent, the feature reports itself unconfigured and the upstream is never called at all. For the pollen key that is a spending control as well as a security one. It is the only upstream in this app that costs money per request, so an unconfigured deployment must make zero calls rather than degraded ones, and the boot log's `pollenConfigured` line is how that is read back after a deploy without sending a request.

The gate runs before routing, so an unauthenticated request to a path that does not exist answers 401 rather than 404. That is one fewer way to enumerate the API from outside.

## Verifying from outside, not by reading the config

The cheapest lesson of the same audit: three security headers were defined at the server level in nginx and were reaching **zero** responses, because `add_header` does not inherit into a location block that declares its own headers. The config looked right. One `curl -sSI` from outside the host proved it was not.

Every control in the table above was verified from off the host after deployment. A control confirmed by reading the configuration that declares it has been confirmed to exist, not to work.

The analytics cookie above is the sharpest case of that sentence in this project, because there was no configuration to misread. The option was present, spelled correctly, in the deployed page, and it did nothing at all. Reading the source proved the author's intent and nothing about the reader's browser. **Where a control lives in somebody else's runtime, the only honest check runs in that runtime**, which is why the probe for it drives a real browser rather than parsing the page.

That probe carries one non obvious requirement worth stating, since it is how the check silently becomes useless. The analytics block suppresses itself for automation, on `navigator.webdriver`, so that Playwright runs and screenshot reshoots against the live site are not counted as visitors. A probe that does not mask that flag therefore reports no tracker and no cookie on a site that tracks every real reader: a clean result produced by the measurement disabling the thing it measures.

The HSTS header added later made the same point twice. It went into all four nginx location blocks rather than the server level alone, for exactly the reason above, and its `max-age` is a deliberate five minutes: HSTS is a one way door for its own lifetime, a browser that has seen it refuses plain HTTP to the host until it expires, and no server side change shortens that. Five minutes means a mistake costs five minutes.

Then the check on it came back clean and was wrong. The header appeared absent on `/assets/`, which is the exact shape of a real scoping failure. It was a stale Cloudflare edge object: `cf-cache-status: HIT` with an `age` of 4,417 seconds, predating the deploy by over an hour. Disproved two ways, with a cache busting query string that forced a new cache key, and with a request straight at nginx bypassing the edge. **On a site behind a CDN, a header check against a cacheable path can lie.** Read `cf-cache-status` and `age` before believing a negative, or bypass the edge outright.

The service worker's script is the path where that lesson had teeth. Before it existed, `/sw.js` fell through to the SPA catch-all, and the header check on that URL showed Cloudflare caching it by extension with `max-age=14400` written over nginx's own `no-cache`. A worker held at the edge can be neither updated nor killed, whatever the deploy does, so the block that serves it sends `no-store`, and the deploy is not finished until the live response says so.

## What is deliberately not defended

Stating this is part of the design rather than an omission:

- **Reads are unauthenticated.** The site is public weather data and a login would protect nothing.
- **A reader's own personal weather station key is stored in their browser in clear text.** This is the optional path where a reader supplies their own Weather Underground key in Settings and the browser calls the upstream directly. It is the user's own key, scoped to reading weather data, held on their own device, in an app with no server side session. The app says so where the key is entered.

  The deployment's **own** Weather Underground key is a different thing and is handled differently: it lives in the server's environment, powers the station picker's discovery and observation lookups through `/api/pws/*`, and is never sent to the browser. The SPA asks that service for **readings** rather than being handed a credential and pointed at the upstream. Both paths exist on purpose, and only the one where the key belongs to the reader puts a key in a browser.

- **The CARTO basemap key is published in the bundle.** It is a build time Vite value, so it is inlined into the shipped JavaScript in plain text and anyone can read it. That is acceptable because it is a basemap tile key, scoped to reads of public map tiles, and it is treated as published rather than secret. It is called out here rather than left for someone to discover, because the same file is one careless line away from carrying something that is not safe to publish: the module holding it documents that anything added there ends up in the shipped JavaScript.
- **There is no rate limiting on reads.** The tunnel in front of the app is where that belongs, and putting a second implementation behind it would be a control nobody maintains. The one read that can cost money is treated differently, because a rate limit bounds the rate and never the bill: a daily summary for a point outside the register is snapped to an 11 kilometre grid, cached in memory, and counted against a per hour ceiling on gateway completions. Past the ceiling the caller gets the composed narrative, labelled as such, until the clock hour turns. Registered places have their own per location cache and never touch it.

## Related

Longer incident write ups from the same systems, in a fixed five part shape that always includes what was rejected: [field-notes](https://github.com/pete-builds/field-notes).
