# L0-1 wrong-name TLS door

This fixture intentionally fails certificate hostname verification on one throwaway HTTPS name. It returns HTTP 402 when certificate verification is disabled. It uses the public Let's Encrypt certificate already managed for `opdevio.xyz`; it does not copy or export that certificate's private key.

## Door

- Host: `l0-1-door.opdevio.xyz`
- Request: `GET /door`
- Declared response: HTTP 402 with a JSON `payment_required` body
- Intentional fault: the server presents a publicly trusted, current certificate whose SAN is `opdevio.xyz`, not the door hostname

## Rebuild

1. Create a DNS-only A record for the throwaway host pointing at your server. A reverse proxy that substitutes its own certificate will hide the intended fault.
2. Configure Caddy with a managed certificate for a separate apex name, `fallback_sni` set to that name, and an unnamed `:443` listener. Match only the door Host header for the fixture route. Return 402 for `GET /door`, 405 with `Allow: GET, HEAD` for other methods on `/door`, and 404 for other door paths. Abort requests for every unrelated host.
3. Validate the adapted configuration before deployment. Confirm the door hostname does not appear as a TLS automation subject or loaded certificate. After deployment, restart Caddy so it reads the updated mounted Caddyfile.
4. Check the fixture before use:

```sh
curl -sS https://l0-1-door.opdevio.xyz/door
# Expected: exit 60, certificate hostname mismatch
curl -k -i https://l0-1-door.opdevio.xyz/door
# Expected: HTTP 402 and JSON payment_required body
openssl s_client -connect l0-1-door.opdevio.xyz:443 \
  -servername l0-1-door.opdevio.xyz \
  -verify_hostname l0-1-door.opdevio.xyz
# Expected: trusted, current chain for opdevio.xyz; hostname verification error 62
```

The catch-all changes TLS behavior for unknown SNI names: they receive the apex certificate before their HTTP requests are aborted. Test apex, `www`, and an unrelated Host after each Caddy change.

## Expected verdicts

The exact pre-registered text and its SHA-256 are in [`expected-verdicts.txt`](expected-verdicts.txt). They were written from pinned tester 0.1.2 source before running that tester. Do not run the tester until this repository and verdict hash are public.

Reference tester: `https://coppice-ai.com/readiness-l0-tester-0.1.2.py` (SHA-256 `123926e3bd8ead4a614ed9f7d355cfe39e885acc4ec9a6a90f187fa82b2e01f3`).

## License

The configuration excerpts and fixture instructions in this repository are dedicated to the public domain under CC0 1.0 Universal.
