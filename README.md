# unitysvc-services-resp

Direct-response (`resp://`) catalog for UnitySVC. These services return a fixed
HTTP status code directly from the gateway — **no real upstream is ever
contacted.** They are the gateway's null/sink targets and the test fixtures for
the routing primitives.

## Why this exists

Every UnitySVC service requires an `upstream_access_interface`, so historically
there was no way to model a "do nothing, just close the loop" endpoint. The
reserved `resp://<status>` upstream scheme (unitysvc/unitysvc#1188) fills that
gap: `base_url = "resp://200"` is a valid, resolvable upstream that means
"the gateway itself produces a 200 response." The synthetic response still runs
through the normal pipeline, so it composes with the wrapper primitives.

## Services

| Service   | Returns | Typical use                                   |
| --------- | ------- | --------------------------------------------- |
| `resp200` | 200     | success sink / no-upstream logging            |
| `resp400` | 400     | client-error negative-path testing            |
| `resp404` | 404     | client-error negative-path testing            |
| `resp429` | 429     | rate-limit / retry-backoff testing            |
| `resp500` | 500     | failing primary for failover testing          |
| `resp503` | 503     | failing primary for failover testing          |

## How they are used

Reach a direct-response service like any other gateway service, optionally
wrapped by a primitive:

```bash
# Direct: the gateway returns 200 with no upstream hop.
curl -H "Authorization: Bearer $UNITYSVC_API_KEY" \
     "$API_GATEWAY_BASE_URL/resp200"

# Logging-as-a-service: log the request, then close the loop with a 200.
# The /l/ wrapper records the request + the synthesized 200 even though
# there is no upstream to forward to — the headline use case (#1129).
curl -H "Authorization: Bearer $UNITYSVC_API_KEY" \
     "$API_GATEWAY_BASE_URL/l/resp200"

# Failover: use resp503 as a guaranteed-failing primary to exercise the
# fall-over path — the gateway returns 503, /f/ treats that as a failure
# and serves the _else target instead.
curl -H "Authorization: Bearer $UNITYSVC_API_KEY" \
     "$API_GATEWAY_BASE_URL/f/resp503?_else=p/openai"

# Tee: mirror a copy of a real request to a resp200 sink (the sink just
# swallows the mirrored call and returns 200; _to names the tee target).
curl -H "Authorization: Bearer $UNITYSVC_API_KEY" \
     "$API_GATEWAY_BASE_URL/t/p/openai?_to=resp200"
```

Because the response is produced inside the normal gateway path, `/l/` logs it,
`/f/` can fall over from it, and `/t/` can mirror to it.

Direct-response services compose naturally with the primitives: `/l/resp200`
(log a request with no real upstream) is the headline case, `resp5xx` makes a
guaranteed-failing `/f/` primary, and `resp200` makes a `/t/` sink. A bare
`/resp200` is useful too — as a gateway placeholder while a real upstream is
being wired up, and as a lightweight health check that exercises the full
auth + routing path without touching any upstream.

## Repository layout

```
data/
  README.md                       # per-service detail
  unitysvc/
    provider.toml                 # provider metadata
    docs/
      connectivity.sh.j2          # shared connectivity test (local + gateway)
    services/
      resp200/ { offering.json, listing.json }
      resp400/ ...
      resp404/ ...
      resp429/ ...
      resp500/ ...
      resp503/ ...
```

## Local development

```bash
pip install unitysvc-sellers

# Validate schemas + file references
usvc_seller data validate

# Check / apply formatting (alphabetical keys, 2-space indent)
usvc_seller data format --check
usvc_seller data format

# Run the connectivity tests against the upstream (resp:// → passes directly)
usvc_seller data run-tests
```

Mirrors the structure of the other `unitysvc-services-*` data repos (e.g.
`unitysvc-services-demo`); see that repo for richer authoring examples.
