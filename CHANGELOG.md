# Changelog

What changed in DewFront and why, newest first.

This is backfilled from the production repository's history, which is private. Dates are UTC and taken from the commits themselves. Numbers in parentheses are pull requests in that repository; work done directly on `main` carries no number.

The app has run as `0.5.0` since the decision-first redesign on 2026-08-17 and there are no version tags, so this changelog is organised by date rather than by release. Every dated section below shipped to [dewfront.com](https://dewfront.com).

---

## 2026-09-13

**A day stopped meaning midnight to midnight, and the outage banner stopped guessing.**

- Split the forecast day at 7 AM and 7 PM, and give each half its own rain chance and its own dew point band. The calendar day was wrong in a way that read as confident: measured on the live site this morning, the Ithaca row advertised a 94 percent chance of rain on a day whose hours between 10 AM and 5 PM never went above 5 percent. The 94 was a 1 AM thunderstorm, which anybody watching the sky called the previous night's storm, filed under the following morning because the clock had rolled over. The error did not run one way either, so it could not be read around: two rows later the same rollup understated a day whose rain was all in the small hours.
- The boundary is not invented here. NWS publishes two 12-hour periods per day with a separate probability on each, and read live from `api.weather.gov` on this date its "Tonight" carried 62 percent while the following "Monday" carried 3 percent: never merged upstream, merged only by this app. Apple's WeatherKit defines its overnight forecast as 7 PM to 7 AM, and Google's Weather API defines daytime as 7 AM to 7 PM with its own probability on each half. Two of those landed on 7 AM independently. MET Norway sidesteps the question entirely by publishing 6-hour blocks.
- The split is the load-bearing half of the change, not the boundary. A single number for a whole day cannot say "dry afternoon, wet night" no matter which hour it is taken from, which is why every service above publishes two.
- The row's dew point band is now the MEAN of each half rather than the overnight minimum. The minimum is the single most favourable hour of the twenty four, and it painted a day averaging 67°F "comfortable" green with the contradicting figure printed in grey beside it on the same row. The range bar's gradient runs the night band into the day band, so the row shows the air changing rather than settling on one colour, and the legend now names the colour channel: the bar encodes temperature in its length and dew point in its colour, and only the length had ever been explained.
- Caught while building, not after: a warm front after dark can leave the night warmer than the day before it, so the daytime high and the overnight low can invert. Taking the difference without sorting gave that day a negative bar width, which the minimum-length clamp turned into a three percent stub pinned at the wrong end of the track. The geometry sorts its ends now, with a test.
- Wind and UV moved into the day's expansion. The row carries two readings well or five badly, and rain and dew point are the two anybody acts on.
- **Stop announcing an outage the app never checked for, then remove the notice entirely.** The stale banner read "Last forecast, just now. The weather service is not answering", which are two sentences that cannot both be true. The forecast query holds a fifteen minute freshness window and seeds itself from the browser's saved copy, so a reload inside that window served the saved forecast and never contacted the provider at all, while the banner took the mere presence of saved data as proof the provider was down. Serving a cache is evidence of a cache. The query now revalidates on every mount, which turns the guess into an observation. The banner itself then came off the top of the page on request, along with its retry button; what stays is the replay behind it, so an outage still leaves the last good forecast on screen rather than an empty page. The cost is stated rather than buried: during an outage the page shows an old reading with nothing marking it as old, and the hero's "Stale" chip does not cover this, because that chip is about the age of a station observation and this is about the forecast. Revalidating on every mount is what makes the trade defensible.
- **One phrase builder was serving two different grammars, and it showed.** "Wait until dew point reaches 61°F eases off." That sentence was live on the public API, reaching the MCP server, the webhooks and the daily summary; a quieter version of the same fault read "The dew point never settles: dew point reaches 75°F overnight." One function returned a clause, which is what two of the four verdict sentences want and exactly what the other two do not: those name the blocker themselves and then need a predicate to follow it. Split in two, each written for its slot. The test that covered the broken sentence passed against it, because it only checked that the clause was present rather than that the sentence read.
- **A daily summary is now computed once a day rather than once a request.** With the gateway off by design, every call took the fallback path and the fallback path cached nothing: two calls 4.6 seconds apart returned different timestamps, each having re-fetched a full forecast and a station reading to rebuild the same sentence. The three hour freshness rule above it was dead code in the app's actual configuration. The old rule said "never cache a fallback" for a reason that had since expired twice over, so it is now about WHY the fallback happened: no gateway configured means cache it, because that is the steady state; a configured gateway that failed means do not, because that is transient and the next request should retry. A cached fallback is dropped the instant a gateway appears.
- **The sun sets every day, and that is not a warning.** Sampled across eight US locations, the rapid temperature drop detector fired in five, every one of them nightfall, all at warning severity. A strip that cries wolf on sunset gets skimmed past on the day a front arrives. The threshold is now conditional on when: inside the ordinary evening cooling band, an hour before sunset to five hours after, a drop has to reach fourteen degrees; outside it the original eight stands. Each candidate is judged against its own hour before it can become the steepest, or a thirteen degree sunset would win the comparison, fail the evening bar, and take a nine degree afternoon front down with it. Re-sampled after: two of eight still fire and both earn it.
- **Report the spell, not the moment, and speak on the steady days.** The dew point sentence took the furthest excursion in its twelve hour window, so a single hour could speak for a night: a dew point touching "humid" at 3 AM and muggy either side was reported as "dew point drops into the humid range overnight". It now requires a band to hold three hours before it counts, then takes the biggest move among the survivors. Taking the longest run instead was the first attempt and was worse, because a humid evening that drops to dry for five hours has more humid hours than dry ones and those five are exactly when the windows go up. Separately, the second sentence used to appear only when something CHANGED, which sounds like restraint and reads as the opposite: Miami and New Iberia, oppressive around the clock, got one bare clause while milder places got three, so the places with the most persistent discomfort said the least about it. And "drops into the humid range" was what an improving night produced, which reads as bad news to anybody not holding the comfort scale in their head; the words are "eases to" and "turns" now.

## 2026-09-10

**Two air readings, and the first upstream that costs money per request.**

- Show the air quality data the app already had. Every page load has fetched a week of hourly AQI, PM2.5, PM10, ozone and NO2 since air quality was added, and rendered one number out of it: a tile with the composite index and its EPA band. The Details screen now carries a panel with the whole scale under the reading, the four pollutants with their concentrations, and the day's course around now. No new upstream, no key, no extra request. The panel also names the pollutant driving the index, and that comes from Open-Meteo's own `us_aqi_*` sub-indices rather than being derived here: the US AQI is defined as the maximum of them, so the one equal to the composite is the driver. When SO2 or CO is the driver, which the app does not display, no visible pollutant matches and the panel says nothing rather than blaming the highest one it can see.
- Add a pollen screen, on Google's Pollen API. Open-Meteo cannot answer this outside Europe: its pollen fields come from the CAMS European domain and North America returns HTTP 200 with every value `null`, which is a green request and a dead panel. Probed at Ithaca with Berlin as a control on the same request.
- The pollen upstream is the first one here that is BILLED per request, and that changed the design rather than just the config. It is held server side, because a key in a bundle is a published key and a public site's page loads are not a budget anybody controls. Answers are cached for six hours on coordinates rounded to about 11 kilometres, which is coarser than pollen varies and coarse enough that a town shares one billed call rather than one per reader. **Failures are cached too**, and that took a second attempt: the cache returns null for a miss, so storing the snapshot directly made a cached failure indistinguishable from nothing cached. It compiled, it passed, and it would have re-billed every page load for exactly the two likeliest failures, a location the model does not cover and an exhausted quota. The value is boxed, and there is a test that reads three calls instead of one without the box.
- Pollen is reached from Details rather than from the tab strip. The strip activates on arrow keys, so a tab there would spend a call on anyone passing through on the way to somewhere else. The query is enabled only while its screen is open, and the condition lives beside the state that decides it rather than inside the component, so a later change to how the screen mounts cannot quietly undo it. An end to end test counts the requests, because a unit test cannot see whether one left the browser.
- Unconfigured, uncovered and unreachable get three different sentences. They send a reader to three different places: the deployment, nowhere, and back in an hour. A pollen index of zero stays distinct from an absent one throughout, because "no pollen in the air" and "this place is not modelled" are opposite answers and a null folded into a zero asserts the wrong one confidently.

## 2026-08-27

**The rain chance stopped coming from a model that disagreed with the government.**

- Take the rain chance from the National Weather Service, and quote the forecaster (#20). The hero's chance of rain came from Open-Meteo with no `models=` parameter, which means `best_match`. On this date in Ithaca that picked a model reading 40 percent and overcast while NWS was calling 85 percent and thunderstorms between 2pm and 4pm. ECMWF said 98 percent and ICON said 90 percent with a thunderstorm code, so the outlier was the one the app happened to be printing. The hero now takes the probability from NWS where NWS covers the point, and prints the forecaster's own sentence with attribution. Outside the US the point lookup answers 404 and the app falls back to Open-Meteo's daily figure.
- One forecast per page, not three (#21). Three separate components had each grown their own copy of the forecast line.
- Headline the peak hour, not a period figure the hourly strip exceeds (#22). The hero quoted a period probability while the strip underneath it showed a higher hour, so the page contradicted itself.
- Say the facts instead of tabulating them (#19), give the hero facts a shape instead of a row (#17), and put the low left and the high right without a rule between them (#18).
- Authenticate the CARTO basemap (#16). CARTO began requiring a key and refuses **silently**: an unkeyed request still answers HTTP 200 with a valid PNG that has "API KEY REQUIRED" printed across the picture. Nothing errors, nothing retries, and a tile-counting check passes. The key is a build-time Vite value, so it is inlined into the shipped bundle by design and an absent key renders the watermarked tiles rather than taking the map out.
- One station on the hero, named once (#15).

## 2026-08-25

- Stop writing an absent dew point into the permanent record as 0°C (#14). A missing reading was being coerced to zero on the way into the hourly observation table, which is a plausible value in Celsius and therefore silently corrupted the accuracy history rather than leaving a gap.

## 2026-08-24

**The station picker became a map, and the tooling stopped assuming one machine.**

- Make the station map a real map you can pick from, and show where the stations are above what they are. Panning finds more stations without lengthening the list, and every pin carries a temperature. It is drawn from raster tiles and a hand-rolled tile grid; there is still no map library in the dependency tree.
- Offer local official stations, not the ring behind them, and always offer enough of them to be a choice. Distance gating stopped the picker presenting stations too far away to mean anything.
- Never offer a station Weather Underground itself flags as bad, and stop a stale discovery index condemning a good one.
- Drop the quality-control verdict and the "showing the N nearest of M" lines. Both were reporting on the picker's own internals rather than helping anyone choose.
- Close the station panel once a station is chosen.
- Give the seven day list the full day-row treatment, and drop the bar-envelope line from it.
- Give the window cards a cold end, so "keep them shut" reads the same whether the binding constraint is heat or cold.
- Start the whole stack with one command on any machine (`npm run dev:all`), proxy `/api` to the backend in Vite dev, load `.env` locally so local matches the deployment, and say which loopback spelling will not work. Vite binds `::1`, so `127.0.0.1` is refused and the error does not say why.
- Make the deploy and the checks survive a different machine. Paths that assumed one developer box were the reason a deploy from the Mac targeted a directory that does not exist on the host.

## 2026-08-23

- Merge the two station controls into one, and name the station on the hero (#12). There had been separate controls for the official station and the personal one, which is an implementation detail the reader should never have been asked to hold.
- Stop both station pickers offering places too far away to matter (#10), and let the operator name a station the picker gets wrong (#11).
- Describe personal stations as neighborhood sensors (#13).
- Point the Threads links at the current handle (#9).

## 2026-08-22

- Let a location read a nearby personal weather station (#7). Weather Underground's PWS network became a source, discovered by proximity rather than configured by hand.
- Fix the source picker layout and the run-together station rows (#8).

## 2026-08-21

**A public identity, a mobile header, and the first security control added after the tunnel.**

- Add HSTS, scoped to the tunnel so the LAN path stays honest (#5). The header is rendered through an nginx `map` on `X-Forwarded-Proto`, so it is set on tunnelled HTTPS traffic and empty on the plain-HTTP LAN path. It went into all four location blocks rather than the server level alone, because `add_header` replaces rather than merges and a server-level-only header would have been live on exactly one path. `max-age` is deliberately five minutes: HSTS is a one-way door for its own lifetime and no server-side change shortens it.
- Reconcile the host with the repository (#3). A deploy on 2026-08-19 built straight to the host without committing, leaving the live public site running code that existed only in a directory with no `.git`. Found and merged back two days later.
- Score the dew point in the accuracy report, which is the number this app is judged on.
- Say what the chance of rain actually measures, name the indoor assumption on the overnight card and offer the setting that changes it, and fade the later forecast days because that is what they are worth.
- Default to the units of the place being viewed rather than a fixed imperial, and carry the reader's unit into the sentences that had hardcoded Fahrenheit.
- One header row on a phone with the controls behind a hamburger, a desktop pill that gets out of the way on the way down, and one row in the tablet band by collapsing the controls rather than the tabs.
- Give DewFront a favicon, a share card and a description that argues for the app rather than describing it.
- Add a footer with Threads, GitHub and LinkedIn, and a colophon naming the data sources.
- Let the details tiles earn their cell, and make dew point a tile like the others so the grid divides evenly.
- Add the GA4 tag.
- Draw the dew point under the 24 hour strip on a shared axis, then revert it. The chart was correct and the screen was busier for it.

## 2026-08-18

**The tunnel went up, and three assumptions became false the same day.**

- Close three HIGH findings opened by the public tunnel (#1). Nothing in the application code changed and no test failed; the app was designed and reviewed against a loopback threat model, and one commit made every one of those assumptions untrue at once.
  - **Server side request forgery with host control.** Webhook registration accepted any http or https URL with no auth and no private range check, and the container runs on the host network, so the request left from the host itself with its own loopback and local subnet in reach. Now gated on the write credential, checked against a blocklist covering loopback, RFC1918, link local, carrier grade NAT and the IPv4-in-IPv6 forms, re-resolved **before every delivery** rather than once at registration because checking once is defeated by DNS rebinding, and refusing redirects because one `302` undoes every check above it.
  - **Unauthenticated station ingest with no time bound.** The endpoint never validated the passkey its own protocol documents and trusted the timestamp in the body, so one request dated four years ahead would pin a forged reading as the latest observation permanently. Now requires the passkey and bounds the timestamp, while still answering `200` either way: consumer station firmware has no error handling, so a 401 would present to the owner as a station that had gone offline.
  - **Full precision coordinates served to anyone.** The location listing returned raw stored coordinates including 15 decimal browser fixes from real visitors, because the summary read path was persisting caller coordinates as a side effect. Now rounded to three decimals on the way out and deduped on the same rounding.
- Force HTTPS for traffic arriving through the tunnel.
- Never request a position without a user gesture. The browser was being asked for geolocation on load, which is a permission prompt nobody invited.
- Expire a cached daily summary after three hours.
- Explain the wait on the windows card with the rule that is actually binding, rather than a generic verdict.
- Collapse the header controls to single-purpose icons, and name the browser tab DewFront.

## 2026-08-17

**Three screens, a backend, and the decision-first redesign that set the thesis.**

- **v0.5.0, the decision-first redesign.** The app stopped presenting a forecast table and started answering the question: one verdict per decision, plus one sentence naming the constraint that bound it.
- Add the backend: `weather-api`, a Fastify service composed from the SPA's own pure functions rather than a reimplementation of them. Bound to loopback, reached through the SPA's nginx `/api/` proxy, never from the LAN.
- Add the api contract, and the inverses the station ingest path needs.
- Add the daily summary, the station card, and forecast versus actual.
- **v0.3.0: three screens, plus Outside and History.**
- Add the insights engine and the three-model comparison, and widen the hourly and daily series.
- **v0.2.0: observed sources, hyperlocal search, the narrative line.**
- Ask the summary model to synthesise rather than restate, resolve a place by proximity rather than by a rounding cell, and headline the night's dew point rather than the afternoon peak.
- Name the app DewFront in the masthead.
- Two test lanes, a server `tsc` target, and a Node 22 floor. The server project compiles the shared `lib` and `api` directories as CommonJS, which is what makes `import.meta` a hard error in any shared module and keeps the two runtimes from drifting.
- CI on Node 22: lint, format check, tests, and both builds.
- Do not say daylight is "left" before the sun has risen.

## 2026-08-16

- **First commit: a weather instrument panel with first-class dew point.** The thesis was in place from the start: dew point does not move with temperature, so it is the reading that answers whether to open the windows, and it belongs in the headline rather than three taps deep.
- Add the host-network override for the deployment, and a deployed-instance end-to-end config.
