# Changelog

What changed in DewFront, newest first. Each entry is dated by the weekend its work landed in. Every entry shipped to [dewfront.com](https://dewfront.com).

---

## 2026-09-19

- The Tonight card now knows what tomorrow is doing. Before a hot day it opens hours it used to shut, because cooling the house overnight is the point rather than a side effect: a 52°F night falling to 43°F went from three open hours to nine when tomorrow reaches 90°F.
- It tells you to shut them again in the morning, which is the half of the technique that makes the other half worth doing.
- On the coldest stretch of a pre-cooling night it says to crack the windows rather than close them, so the house keeps flushing without the bedroom going arctic.
- Nights with no forecast for tomorrow, and nights before a mild day, are unchanged.

---

## 2026-09-18

- The Windows card now answers a different question in cold weather: whether the house is too dry, and whether to run a humidifier. It used to say "keep them closed" every hour from late October to April, which is right and tells you nothing you could not read off the thermometer.
- It shows what the outside air becomes once it is heated indoors, what to aim a humidifier at, and the humidity above which your windows start sweating. That ceiling is the published glass guidance for the outdoor temperature, so the advice tightens as it gets colder.
- The dew point scale gained four dry bands where it had one. A 50°F dew point and a 0°F one both used to read "Dry" in the same colour; they are a 49% room and a 6% room.
- Fixed a chilly evening showing a chip that read "cold" in lower case, and the dew point key highlighting four bands at once.

---

## 2026-09-16

- Analytics no longer sets cookies. The site was setting two Google Analytics cookies for two years while the privacy notice said it set none; the counter now stores nothing on your device and cannot recognise a returning visitor.
- The privacy notice now names Google, and says plainly that Cloudflare and Google both see your IP address. It also lists everything the app keeps in your browser, including your own Weather Underground key, and no longer implies API request logging covers ordinary page requests.
- The build now fails if the privacy notice and the analytics code disagree, and `npm run privacy:probe` checks the deployed site against what the notice promises.

---

## 2026-09-13

- Privacy notice at [dewfront.com/privacy](https://dewfront.com/privacy), linked from the footer. No accounts, and it names every service your browser contacts directly. Its claim of no cookies and no trackers was wrong, and was corrected on 2026-09-16.
- The web server no longer logs the query string or the caller's address on API requests. It had been recording coordinates to fifteen decimal places next to the IP that sent them.
- The app now logs which ~11 km grid cell a guest forecast was for, and nothing else, so coverage can be measured without recording anyone.
- Desktop layout. The page used to stop widening at 1088px, so a 1920px screen showed the tablet layout with a third of the screen empty and the same amount of scrolling.
- On a wide screen the hero splits in two: where you are, the temperature, the dew point and feels-like on the left, the day ahead and the forecaster's sentence on the right. The whole 24-hour strip now sits above the fold.
- Next 7 days and Details share a row on a wide screen, and the Details tiles divide evenly instead of wrapping four and then two.
- Summary refreshes every hour in an open tab. It used to keep whatever sentence the tab was opened with.
- Fixed the forecaster's sentence being hours behind. The National Weather Service serves a separately cached copy per content type, and the one this app asked for was a three and a half hour old issuance: at 9pm it still read "showers and thunderstorms before 7pm".
- Forecast and the forecaster's sentence now refresh every hour in an open tab.
- The station name and the station picker moved beside the readings they describe instead of sitting centred under the whole panel.
- Daily summary for a place outside the register is shared per 11 km cell and capped per hour, so a coordinate sweep cannot run up the model bill. Past the cap the summary is the composed narrative, labelled as such.
- A fresh clone now runs on plain Docker: nginx reaches the API by service name instead of a loopback address that only worked on the production host.
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

## 2026-09-12

- Air quality panel on Details: the full scale, four pollutants with concentrations, the day's course, and the driving pollutant.
- Pollen screen on Google's Pollen API, reached from Details. Answers cached six hours on an 11 km grid, failures included, so a town shares one billed call.
- Unconfigured, uncovered and unreachable pollen say three different things. Zero pollen stays distinct from no data.

## 2026-08-29

- Rain chance comes from the National Weather Service where it covers the point, with the forecaster's own sentence quoted. Open-Meteo elsewhere.
- One forecast per page instead of three copies.
- The hero headlines the peak hour's rain chance, so it never contradicts the hourly strip.
- Hero facts read as sentences, low on the left and high on the right.
- CARTO basemap authenticated. Unkeyed tiles were rendering "API KEY REQUIRED" with a 200.
- One station, named once on the hero.

## 2026-08-23

- A missing dew point is no longer recorded as 0°C in the accuracy history.
- The station picker is a map. Pan to find stations; every pin shows a temperature. Still no map library.
- Only nearby official stations are offered, and always enough to choose from.
- Stations Weather Underground flags as bad are never offered.
- Dropped the quality-control verdict and the "N nearest of M" line.
- The station panel closes once a station is chosen.
- Seven day list gets full day rows; window cards get a cold end.
- `npm run dev:all` starts the whole stack on any machine, and the deploy works from any machine.
- Two station controls merged into one, and the station is named on the hero.
- Station pickers stop offering distant places; the operator can name a station by hand.
- Personal stations described as neighborhood sensors.

## 2026-08-22

- A location can read a nearby personal weather station from Weather Underground.
- Fixed the source picker layout and the run-together station rows.
- HSTS on the public site, scoped to the tunnel.
- Host and repository reconciled after a deploy that skipped the commit.
- Dew point is scored in the accuracy report.
- Chance of rain explained, the indoor assumption named and adjustable, later days faded.
- Units default to the place being viewed, and every sentence follows the unit toggle.
- One header row on phones with the controls behind a menu.
- Favicon, share card and a description.
- Footer with Threads, GitHub, LinkedIn and the data source credits.
- Dew point is a tile like the others.

## 2026-08-16

- Three high findings closed after the public tunnel went up: webhook request forgery, unauthenticated station uploads, full-precision visitor coordinates.
- HTTPS forced through the tunnel.
- Location is never requested without a tap.
- Cached daily summary expires after three hours.
- The windows card names the rule that is binding.
- Header controls collapsed to icons; the browser tab is named DewFront.
- v0.5.0, the decision-first redesign: one verdict per decision, and one sentence naming the constraint.
- Backend added, built from the app's own functions: daily summary, station card, forecast versus actual.
- v0.3.0: three screens plus Outside and History, the insights engine, and the three-model comparison.
- v0.2.0: observed sources, hyperlocal search, the narrative line.
- Named DewFront.
- Two test lanes, a server build, and CI on Node 22.
- Daylight is not "left" before sunrise.
- First commit: a weather instrument panel with dew point in the headline.
