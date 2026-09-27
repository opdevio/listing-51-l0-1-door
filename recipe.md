# Listing 51 L0-1 certificate fixture

This recipe is dedicated to the public domain under CC0 1.0 Universal. You may copy, modify, and redistribute it without asking permission. The license text is available at https://creativecommons.org/publicdomain/zero/1.0/legalcode.

## Fixture

Door: https://l0-1-door.opdevio.xyz/door

Declared request: `GET /door`. It returns HTTP 402 with `Content-Type: application/json`. The fixture intentionally presents the currently valid, publicly trusted certificate for `opdevio.xyz` when a client connects with SNI `l0-1-door.opdevio.xyz`. That certificate does not name the door host. Do not use this pattern for a name that serves real users or agents.

The current leaf certificate observed on 2026-09-27 has subject `CN=opdevio.xyz`, SAN `DNS:opdevio.xyz`, issuer Let's Encrypt YE2, validity 2026-09-18 through 2026-12-17, and SHA-256 fingerprint `49:1E:74:9D:0F:BE:82:80:F6:A6:5A:E8:88:47:80:30:5C:C1:64:EF:54:B2:38:F0:94:FB:F8:5F:03:5C:49:98`. Caddy renews this certificate. Renewal changes the leaf fingerprint and validity dates; the required fault is the wrong hostname, not a fixed fingerprint. Re-check the certificate before using the recipe.

## DNS and Caddy configuration

Create a DNS-only A record from `l0-1-door.opdevio.xyz` to the origin address. Do not proxy it through a CDN, since the edge certificate would hide the intended fault.

The global Caddy options must include:

```caddyfile
fallback_sni opdevio.xyz
```

Add this host-scoped handler to the Caddyfile:

```caddyfile
:443 {
	import privacy
	@door host l0-1-door.opdevio.xyz
	handle @door {
		handle /door {
			@declared method GET HEAD
			handle @declared {
				header Content-Type application/json
				respond `{"error":"payment_required","fixture":"L0-1 wrong-name certificate","declared":"GET /door"}` 402
			}
			handle {
				header Allow "GET, HEAD"
				respond 405
			}
		}
		handle /robots.txt {
			respond 404
		}
		handle {
			respond 404
		}
	}
	handle {
		abort
	}
}
```

The global fallback serves the apex certificate for the door SNI. The scoped route permits HTTP responses only for the door host; other unmatched hosts abort. Preserve the existing apex and `www` site blocks. Before deployment, adapt the Caddyfile with the service's Caddy version and confirm that Caddy will not manage or load a matching certificate for the door hostname. Validate and check unrelated host behavior before deploying.

## Checks

These commands make an ordinary verified request and an insecure request. The first must fail specifically with curl exit 60 or 51 for the certificate; the second must return 402.

```sh
curl -sS https://l0-1-door.opdevio.xyz/door
# Expected: exit 60 or 51; certificate verification failure for hostname mismatch.

curl -k -i https://l0-1-door.opdevio.xyz/door
# Expected: HTTP 402, Content-Type application/json, and the declared JSON body.

openssl s_client -connect l0-1-door.opdevio.xyz:443 \
  -servername l0-1-door.opdevio.xyz \
  -verify_hostname l0-1-door.opdevio.xyz </dev/null
# Expected: the trusted opdevio.xyz certificate and hostname mismatch, verify code 62.
```

For the current certificate fingerprint, save the leaf certificate from `s_client` output and run:

```sh
openssl x509 -in leaf.pem -noout -subject -issuer -dates -ext subjectAltName -fingerprint -sha256
```

A certificate may renew. Confirm it remains publicly trusted and valid, and that its SAN still does not include the door hostname.

## Expected verdicts

The expected verdict text is in `expected-verdicts.txt`. It was derived from readiness tester 0.1.2, SHA-256 `123926e3bd8ead4a614ed9f7d355cfe39e885acc4ec9a6a90f187fa82b2e01f3`, and the live endpoint behavior observed on 2026-09-27. The tester source was read but not run. The expected-verdict file's SHA-256 is:

```text
a924deb53568c7ce2e02a947ad53c83674918aeb18633460439c6a9fc16057d9
```

The verdict file is UTF-8 text with LF line endings and a final newline. Its SHA-256 is the SHA-256 of those exact bytes; do not reformat it after publishing. DNS resolves directly to the origin. Public curl exits 60 for the wrong certificate; `curl -k` returns 402. L0-2 through L0-5 are UNOBSERVED because the tester's profiles verify TLS and cannot get an HTTP response.

## Excluded-party statement

OPDEVIO is not the author of any readiness tester. OPDEVIO does not operate a door on Coppice's board or watch, has not asked Coppice to stop sending work, and is not Cairn. Neither `opdevio.xyz` nor the origin address is listed on coppice-ai.com/readiness-l0.html, its watch, or its home page. OPDEVIO sells paid single TLS and x402 checks; that commercial relationship is disclosed here. OPDEVIO does not run the readiness tester. These statements reflect checks recorded on 2026-09-27 and should be rechecked before any submission.
