# Changelog

What changed in DewFront, newest first. Dates are UTC. Every entry shipped to [dewfront.com](https://dewfront.com).

---

## 2026-09-13

- Public changelog at [dewfront.com/changelog](https://dewfront.com/changelog), linked from the footer.
- Installs as an app. The service worker caches the app shell and never a reading. Offline, the page says the forecast cannot be reached instead of showing an old number.
- Worker kill switch: `?sw=off` on any URL, or a build flag that removes it from every browser.
- Forecast days run 7 AM to 7 AM, with a separate rain chance and dew point band for day and night. The calendar day was filing 1 AM storms under the next morning.
- Day rows colour by the mean dew point of each half, not the overnight minimum. The legend now says bar length is temperature and colour is dew point.
- Wind and UV moved into the day's expansion; an expanded day lists 7 AM through 4 AM.
- Stale forecast banner removed. The last good forecast still shows during a provider outage, and the forecast revalidates on every load.
- Fixed run-on verdict sentences like "Wait until dew point reaches 61°F eases off" in the summary, webhooks and API.
- Daily summary is computed once a day, not once a request.
- Rapid temperature drop alert no longer fires on ordinary evening cooling.
- Home screen icon on iOS and Android, a web manifest, and launch without browser chrome.
- Fixed the manifest being served as a binary file.
- Forecaster's sentence no longer repeats the period name.
- Dew point sentence reports a spell held for three hours, not a single hour, and speaks on steady days too.
- Heat alert reports the level held for three hours, not one spike hour. Miami no longer reads 123°F from one bad cell.
- Dew point colour key sits under the forecast rows, not only on the welcome screen.
- Air quality tile names the pollutant driving the index.
- "Opens after 2 AM" reads "Open after 2am".

## 2026-09-10

- Air quality panel on Details: the full scale, four pollutants with concentrations, the day's course, and the driving pollutant.
- Pollen screen on Google's Pollen API, reached from Details. Answers cached six hours on an 11 km grid, failures included, so a town shares one billed call.
- Unconfigured, uncovered and unreachable pollen say three different things. Zero pollen stays distinct from no data.

## 2026-08-27

- Rain chance comes from the National Weather Service where it covers the point, with the forecaster's own sentence quoted. Open-Meteo elsewhere.
- One forecast per page instead of three copies.
- The hero headlines the peak hour's rain chance, so it never contradicts the hourly strip.
- Hero facts read as sentences, low on the left and high on the right.
- CARTO basemap authenticated. Unkeyed tiles were rendering "API KEY REQUIRED" with a 200.
- One station, named once on the hero.

## 2026-08-25

- A missing dew point is no longer recorded as 0°C in the accuracy history.

## 2026-08-24

- The station picker is a map. Pan to find stations; every pin shows a temperature. Still no map library.
- Only nearby official stations are offered, and always enough to choose from.
- Stations Weather Underground flags as bad are never offered.
- Dropped the quality-control verdict and the "N nearest of M" line.
- The station panel closes once a station is chosen.
- Seven day list gets full day rows; window cards get a cold end.
- `npm run dev:all` starts the whole stack on any machine, and the deploy works from any machine.

## 2026-08-23

- Two station controls merged into one, and the station is named on the hero.
- Station pickers stop offering distant places; the operator can name a station by hand.
- Personal stations described as neighborhood sensors.

## 2026-08-22

- A location can read a nearby personal weather station from Weather Underground.
- Fixed the source picker layout and the run-together station rows.

## 2026-08-21

- HSTS on the public site, scoped to the tunnel.
- Host and repository reconciled after a deploy that skipped the commit.
- Dew point is scored in the accuracy report.
- Chance of rain explained, the indoor assumption named and adjustable, later days faded.
- Units default to the place being viewed, and every sentence follows the unit toggle.
- One header row on phones with the controls behind a menu.
- Favicon, share card and a description.
- Footer with Threads, GitHub, LinkedIn and the data source credits.
- Dew point is a tile like the others.

## 2026-08-18

- Three high findings closed after the public tunnel went up: webhook request forgery, unauthenticated station uploads, full-precision visitor coordinates.
- HTTPS forced through the tunnel.
- Location is never requested without a tap.
- Cached daily summary expires after three hours.
- The windows card names the rule that is binding.
- Header controls collapsed to icons; the browser tab is named DewFront.

## 2026-08-17

- v0.5.0, the decision-first redesign: one verdict per decision, and one sentence naming the constraint.
- Backend added, built from the app's own functions: daily summary, station card, forecast versus actual.
- v0.3.0: three screens plus Outside and History, the insights engine, and the three-model comparison.
- v0.2.0: observed sources, hyperlocal search, the narrative line.
- Named DewFront.
- Two test lanes, a server build, and CI on Node 22.
- Daylight is not "left" before sunrise.

## 2026-08-16

- First commit: a weather instrument panel with dew point in the headline.
