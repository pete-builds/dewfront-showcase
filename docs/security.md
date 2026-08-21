# Security

## The threat model changed without the code changing

DewFront was designed and reviewed against a loopback threat model, and the code said so explicitly and correctly. The backend binds to loopback only. The write routes were reasoned about as reachable only from the local network. Comments in the source recorded that reasoning.

Then one commit put the app behind a public tunnel, and every one of those assumptions became false in the same moment. No application code changed. No test failed. Nothing in the deployment looked different from the inside.

**Public ingress is a change to the threat model, not to the deployment.** None of the three findings below required a new bug. All three were written into the design correctly, against a world that stopped existing the day the tunnel came up.

An audit run the same day found three high severity issues, all live on the public internet.

## The three findings

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

## The controls now

| Surface | Control |
|---|---|
| Location registration | Bearer token, fails closed when unconfigured |
| Webhook registration | Bearer token, plus destination blocklist |
| Webhook delivery | Destination re resolved and re checked per delivery, redirects refused |
| Station ingest | Shared passkey in the body, timestamp bounded, constant response |
| Location listing | Coordinates rounded on output |
| Webhook listing | Origins only, the full URL is never returned |
| All reads | Open by design. The whole site is public weather data |
| Errors | `{error, detail}` with no path, no upstream URL and no environment value |

Both credential gates fail closed. With the variable unset, the routes they guard refuse everything rather than falling open, so a deploy that forgets a secret leaves a working read only site rather than an open write surface. The boot log reports whether each gate is configured and never the values.

The gate runs before routing, so an unauthenticated request to a path that does not exist answers 401 rather than 404. That is one fewer way to enumerate the API from outside.

## Verifying from outside, not by reading the config

The cheapest lesson of the same audit: three security headers were defined at the server level in nginx and were reaching **zero** responses, because `add_header` does not inherit into a location block that declares its own headers. The config looked right. One `curl -sSI` from outside the host proved it was not.

Every control in the table above was verified from off the host after deployment. A control confirmed by reading the configuration that declares it has been confirmed to exist, not to work.

## What is deliberately not defended

Stating this is part of the design rather than an omission:

- **Reads are unauthenticated.** The site is public weather data and a login would protect nothing.
- **The personal weather station credential is stored in the browser in clear text.** It is the user's own key, scoped to reading weather data, held on their own device, in an app with no server side session. The alternative is giving a static file container a backend and a secret to protect. The app says so where the key is entered.
- **There is no rate limiting on reads.** The tunnel in front of the app is where that belongs, and putting a second implementation behind it would be a control nobody maintains.

## Related

Longer incident write ups from the same systems, in a fixed five part shape that always includes what was rejected: [field-notes](https://github.com/pete-builds/field-notes).
