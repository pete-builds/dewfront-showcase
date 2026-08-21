# Decisions

Each entry states the decision, the option on the other side of the fork, and what would reverse it. Entries with no reversal condition say so explicitly, because "nothing would change my mind" is a claim that should be made out loud rather than left implied.

---

## Dew point is the headline and relative humidity is a footnote

**Decision.** Dew point appears in the current panel with its comfort word, on every hourly column, on every day of the ten day list, and as the lead panel on the details screen. Relative humidity sits underneath it as a footnote and does not appear on the windows decision card at all.

**The other side.** Every hosted weather app leads with relative humidity, and users have been trained on it for thirty years. Leading with a number most people cannot interpret is a real adoption cost.

**Why this way.** Relative humidity cannot answer the question the app exists to answer, because it moves with temperature. 90 percent at 50°F is dry air and 60 percent at 85°F is not. The teaching burden is a one time cost and the comfort word carries most of it: "dew point 52, comfortable" needs no training.

**Enforcement.** A test asserts relative humidity is absent from the windows card. Layout arguments decay under maintenance unless something fails when they are violated.

**Reverses if:** usage shows people reading the dew point pill and then going to look for relative humidity anyway, which would mean the comfort word is not carrying the load.

---

## Source precedence is per field, not per source

**Decision.** Each field independently takes the best available source: the configured personal weather station if fresh within 60 minutes, then the nearest NWS station if fresh within 90 minutes, then the model.

**The other side.** Pick one source per render and use it for everything. Simpler to implement, simpler to explain, one provenance label instead of one per row.

**Why this way.** A consumer weather station typically has no barometer. Under per source precedence, a station with a thermometer and a hygrometer would drag pressure down to a modelled value along with everything else, or would have to be rejected wholesale. Per field, that station contributes exactly what it measures.

**The cost, paid deliberately.** Provenance becomes a per row property, so the UI has to carry an accent mark on measured values and a label naming the active source with its distance and age. That is more UI than one badge, and it is the honest amount.

**Reverses if:** mixing sources ever produces an internally inconsistent set of readings that a user could catch, for example a measured temperature and a modelled dew point implying a relative humidity neither source reported.

---

## A stale station is shown struck through, not hidden

**Decision.** A station that answered but whose reading has aged past the cutoff is displayed struck through, and a pinned station stays pinned once stale rather than silently failing over to a different one.

**The other side.** Hide it and fail over. The user gets a number either way and never sees an error state.

**Why this way.** A missing number should never be silent, and a pin the app quietly overrides is not a pin. If a user chose a station, the app either uses it or tells them it cannot.

**Reverses if:** nothing.

---

## Compute the NWS heat index instead of reusing the model's apparent temperature

**Decision.** Implement NOAA's Rothfusz regression with both published corrections, rather than reading the upstream's `apparent_temperature` field.

**The other side.** The field is right there in the response, costs nothing, and is a defensible feels like number.

**Why this way.** `apparent_temperature` is a Steadman style figure that folds in wind chill and radiation. It is not the NWS heat index, and every published hydration and exertion guideline is written against the NWS number. An app that shows a heat figure next to activity advice and means a different quantity than the guidelines is quietly wrong in the exact place it matters most.

**The subtle part.** The result is deliberately not floored at the air temperature. At 100°F and 10 percent humidity the heat index is genuinely about 94°F, because sweat evaporates freely in very dry air. An earlier draft clamped it to the air temperature, which silently cancelled the dry air correction. A test caught it and now guards it.

**Reverses if:** the guidelines the app aligns with move to a Steadman basis.

---

## Share domain logic between frontend and backend by compile, not by package

**Decision.** The backend's TypeScript project includes the shared logic and upstream client directories in its own compile. Both sides import the same functions from the same files.

**The other side.** Publish an internal package, or duplicate the handful of derived functions the API needs.

**Why this way.** Duplication here means the browser and the server can disagree about the same forecast, and nothing fails when they start to. A published package means a version skew window and a release step on every threshold change. The compile include gives one definition of every threshold and breaks both builds simultaneously when it changes.

**The cost, paid deliberately.** Everything in the shared tree must run in both a browser and Node. The clock is an argument rather than an import, and the upstream clients set no headers a browser cannot set.

**Reverses if:** a third consumer appears that cannot take a source dependency, at which point a package is the right shape.

---

## Grade forecasts against the earliest snapshot held

**Decision.** For each target date, the accuracy report uses the earliest snapshot it holds, which is the longest lead available.

**The other side.** Use the most recent forecast before the day, which is the one a user would actually have read.

**Why this way.** A forecast issued at noon for that same afternoon is barely a forecast. Grading against it flatters the model and produces a number that looks like accuracy and measures nothing.

**Reverses if:** the report needs to answer "how good is the short lead forecast", which is a genuinely different question and deserves its own column rather than a redefinition of this one.

---

## Leave the rain accuracy column null

**Decision.** Actual precipitation stays null and the panel says why, rather than being filled from the model's own estimate.

**The other side.** The estimate is right there, the column would be full, and the panel would look complete.

**Why this way.** The model's estimate is not an actual. Comparing a forecast to a reanalysis of the same model is a measurement of nothing, presented with the authority of a measurement of something. The upstream station payload does carry an hourly precipitation field, and it was probed null at the reference station, as it routinely is at ASOS sites, so trusting it would have produced a column that was empty for ever without saying so.

**Reverses if:** a gauge is wired in, which is precisely what the station ingest path exists for.

---

## Return the sample size so the UI can refuse to conclude

**Decision.** The accuracy endpoint returns `n`, counting only days where both a forecast and an actual existed, alongside the signed bias.

**The other side.** Return the bias. The client can draw a number.

**Why this way.** A bias computed from one day is not a bias. Returning the count is what lets the panel decline to say anything yet, and declining is the correct output for most of the first fortnight of any deployment.

**Reverses if:** nothing.

---

## Turn the LLM daily summary off rather than let it classify

**Decision.** The gateway backed daily summary is disabled. The app serves a narrative composed from the data by code.

**The other side.** Leave it on. It was working, it read well, and a bad sentence once in a while is a small cost.

**Why this way.** A live sample described the air as dry at a 59°F dew point, which the app's own tested classifier calls slightly humid. That is untested prose contradicting a tested classifier, on the app's thesis metric, in the one place a reader is most likely to trust it. It was also a second rendering of a sentence the app already composes, already displays and already tests.

**Reverses if:** the classifier's verdict is fed into the prompt as a constraint, so the model rephrases a decision it is not allowed to make. That is the design for turning it back on, and it is written down rather than remembered.

---

## Both write gates fail closed

**Decision.** With the credential environment variable unset, the routes it guards refuse everything. The boot log reports whether each gate is configured, and never the values.

**The other side.** Fall open when unconfigured, so a fresh deployment is usable before secrets are in place.

**Why this way.** A deploy that forgets a secret should produce a working read only site with writes turned off, not an open write surface on the public internet. The direction a mistake takes you is a design decision, and it should be made before the mistake.

**A consequence worth knowing.** The gate runs before routing, so an unauthenticated request to a path that does not exist answers 401 rather than 404. That is one fewer way to enumerate the API from outside.

**Reverses if:** nothing.

---

## A rejected station upload still answers 200

**Decision.** The station ingest endpoint answers `200 ok` whether or not the passkey matched. The refusal is a log line.

**The other side.** Answer 401. It is the correct status code, and silently accepting a rejected request looks like a bug to anyone reading the route.

**Why this way.** Consumer weather station firmware has no error handling and no retry. A 401 would be invisible to the device and would present to its owner as a station that had gone offline, which is a worse failure than a log line they can look up. It also means guessing the passkey against that endpoint returns no signal to the guesser.

**Reverses if:** the ingest path is fronted by something that can surface an error to a human.

---

## Webhook targets are re resolved before every delivery

**Decision.** The destination is resolved and checked against the private range blocklist before each delivery, not once at registration, and the delivery refuses redirects outright.

**The other side.** Validate at registration. It is one check instead of one per send, and the URL has not changed.

**Why this way.** The URL has not changed but what it resolves to can. Checking once is defeated by DNS rebinding: register a name that answers publicly, then answer loopback afterwards. And one `302` to a private address undoes every check above it, so redirects are an error rather than a hop.

**Reverses if:** nothing.
