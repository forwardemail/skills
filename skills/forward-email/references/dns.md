# DNS for Forward Email

Take every name and value from the Forward Email domain object (`getDomain` / `GET /v1/domains/:domain`):

- `verification_record` is the token for the TXT value `forward-email-site-verification=<token>`.
- `smtp_dns_records.dkim` is `{ name, value }` and becomes a TXT record.
- `smtp_dns_records.return_path` is `{ name, value }` and becomes a CNAME, usually `fe-bounces` → `forwardemail.net`.
- `smtp_dns_records.dmarc` is `{ name, value }` and becomes a TXT record, usually at `_dmarc`.

Names are relative to the zone. For a subdomain such as `mail.example.com` in the `example.com` zone, the API returns names like `_dmarc.mail`, and the MX, verification TXT and SPF records go on the `mail` host.

Fixed records:

- MX `@` → `mx1.forwardemail.net` (priority 0)
- MX `@` → `mx2.forwardemail.net` (priority 0)
- TXT `@` → `v=spf1 include:spf.forwardemail.net -all` (merge into an existing SPF record rather than adding a second one)

Optional: CNAME `autoconfig` → `autoconfig.forwardemail.net` and CNAME `autodiscover` → `autodiscover.forwardemail.net`. These let email clients configure themselves.

## Before writing anything

1. Read the current records for `@`, `_dmarc` and the DKIM and Return-Path names.
2. If MX already points somewhere else, the domain receives mail today. Confirm with the user before replacing it. To keep it and only send, skip the MX records and set `ignore_mx_check: true` on the domain.
3. If there is already an SPF record, add `include:spf.forwardemail.net` to it rather than creating a second one.
4. If there is already a DMARC record, or other services send as this domain, show the values and ask which policy to use. `p=none` and `p=quarantine` also pass verification.

## Writing records automatically

Use whatever access the environment already has: a DNS provider MCP server, the provider's official CLI, or its REST API with a token from an environment variable. Look up the exact syntax in the provider's own docs. Never guess at flags.

### Cloudflare REST API example

```bash
# Needs CLOUDFLARE_API_TOKEN with Zone → Zone → Read and Zone → DNS → Edit,
# or set CLOUDFLARE_ZONE_ID yourself and skip the lookup.
DOMAIN=example.com
CF="https://api.cloudflare.com/client/v4"
AUTH="Authorization: Bearer $CLOUDFLARE_API_TOKEN"
ZONE_ID=${CLOUDFLARE_ZONE_ID:-$(curl -fsS "$CF/zones?name=$DOMAIN" -H "$AUTH" | jq -r '.result[0].id')}
[ -n "$ZONE_ID" ] && [ "$ZONE_ID" != null ] || { echo "zone not found for $DOMAIN" >&2; exit 1; }

add() { curl -fsS -X POST "$CF/zones/$ZONE_ID/dns_records" -H "$AUTH" \
  -H "Content-Type: application/json" --data "$1" >/dev/null && echo "added: $1"; }

add '{"type":"MX","name":"'$DOMAIN'","content":"mx1.forwardemail.net","priority":0,"ttl":3600}'
add '{"type":"MX","name":"'$DOMAIN'","content":"mx2.forwardemail.net","priority":0,"ttl":3600}'
add '{"type":"TXT","name":"'$DOMAIN'","content":"forward-email-site-verification=TOKEN","ttl":3600}'
# The Return-Path CNAME must be DNS-only:
add '{"type":"CNAME","name":"fe-bounces.'$DOMAIN'","content":"forwardemail.net","proxied":false,"ttl":3600}'
```

Cloudflare expects full record names that include the zone (for example `fe-bounces.example.com`). Add the SPF, DKIM and DMARC TXT records the same way, using the values from the domain object.

## Verify with backoff

Run both checks; verify-smtp doesn't depend on verify-records.

```bash
check() {  # retries after 15 s, 30 s, 60 s, then every 2 minutes; about 15 minutes in total
  for wait in 15 30 60 120 120 120 120 120 120 0; do
    curl --fail-with-body -sS -u "$FORWARD_EMAIL_API_KEY:" \
      "https://api.forwardemail.net/v1/domains/$DOMAIN/$1" && return 0
    echo; sleep "$wait"
  done
  return 1
}
check verify-records; check verify-smtp
```

- When a check fails, the response text says which record is missing or wrong. Fix it, then run the check again.
- When `verify-smtp` passes, domains that pass Forward Email's automatic checks get outbound SMTP right away. `getDomain` then shows `has_smtp: true`.
- Other domains get a manual review: most within 1–2 hours, typically under 24. The domain admins are emailed, and Forward Email may ask for details.
