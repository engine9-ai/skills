---
name: e9-delegate-domain
description: >-
  Add (register, allow, enable), disable, or hand over a Domain on Delegate,
  engine9's common sign-in, so a site can receive Identity Tokens. Covers the
  request (exact hosts, required contact email at any address, organization
  name, integration), DNS TXT proof of control (_delegate.<host>), registration
  and the authorize smoke test, per-request sign-in branding (brand_name,
  brand_color, brand_logo, brand_wordmark, brand_background, brand_text,
  brand_card, brand_card_text, brand_font, brand_google), per-request sign-in
  providers (login_providers: google, facebook), disabling, and
  revoking Grants after a compromise or change of owner. Use when someone asks
  to add or remove a domain on Delegate, to brand the Delegate sign-in page or
  email, or when a site gets 400 invalid_domain.
---

# Add a Domain to Delegate

Delegate is engine9's common sign-in. It issues Identity Tokens only to
**Domains** it allows: an exact `host` or `host:port` such as
`www.example.org`. This skill covers the request a site owner sends, how an
engine9 operator verifies and registers it, how the site brands the sign-in
page and email, and how to disable a Domain safely.

## Quick reference

| Item | Value |
| --- | --- |
| Delegate host | `https://delegate.network` is the default. `https://delegate.engine9.ai` still answers and signs tokens with `iss: https://delegate.engine9.ai` |
| Domain | Exact `host` or `host:port`. `example.org` and `www.example.org` are two Domains. No wildcards |
| Required from the requester | Exact host(s), **contact email** (any address), organization name, integration |
| Ownership proof | DNS TXT `_delegate.<host>` = `delegate-verification=<random>`; kept in place afterward |
| Already allowed, no request needed | Hosts under `.engine9.ai` and `.engine9.io`; `localhost:3000` through `localhost:3003` |
| Branding | Sent by the site on every request (`brand_*`). Delegate stores none of it |
| Registration (operator) | `scripts/addDomain.sh --email <contact> <host> "<name>"` in the Delegate repository |
| Disable (operator) | `scripts/addDomain.sh --disable [--revoke-grants] <host>` |
| Not registered yet | `GET /identity/authorize` answers `400 {"error":"invalid_domain"}` |

## Concepts

| Term | Meaning |
| --- | --- |
| Domain | The host a site signs people in from. It is the JWT `aud`. Tokens are delivered only to a `return_to` on that exact host, over `https` (`http` only on `localhost`) |
| Contact email | The person responsible for the Domain. Delegate uses it for disables, revoked Grants, and security notices. It does not need to be at the Domain; a personal or role address is fine |
| Grant | A User's permission for one Domain to receive certain fields (email, name, and so on). While a Grant covers a request, Delegate signs the User in to that Domain with no page |
| Domain UNID | The token's `sub`. Different for every Domain, so Domains cannot link one person across sites |
| Branding | `brand_*` query parameters on `/identity/authorize` and `/identity/bridge`. Validated per request; invalid values are ignored |

Rule: A Domain is an exact host. Register each host the site signs people in from, including both the apex and `www` when both are used.

Rule: A Domain is registered once and works on every Delegate host. Registering a Domain never depends on which Delegate host the site uses.

### Which Delegate host

Delegate answers on two hosts. The Identity Token's `iss` is the host that issued it.

| Host | Status | `iss` |
| --- | --- | --- |
| `https://delegate.network` | Default for `@engine9/id` and `@engine9/core` | `https://delegate.network` |
| `https://delegate.engine9.ai` | Still works; keep it for sites that set it | `https://delegate.engine9.ai` |

Rule: A site verifies `iss` against the single Delegate URL it configured (`delegateUrl` in `@engine9/id`, `DELEGATE_URL` in `@engine9/core`). Do not tell a site to accept either issuer. A site that sets `delegateUrl` to the old host keeps working and keeps getting old-host tokens until it changes the setting.

Rule: Send the same host for the authorize request, the JWKS, and the issuer check. Mixing hosts fails the `iss` check.

Rule: The contact email is required. Do not register a Domain without one. It does not have to be at the Domain.

## Workflow

### 1. Collect the request

| Field | Required | Notes |
| --- | --- | --- |
| Domain(s) | Yes | Each exact host. Subdomains are not covered by the parent |
| Contact email | **Yes** | Any address the responsible person reads |
| Organization name | Yes | An internal label on the registry row. Users never see it; the site sets what users see with `brand_name` |
| Integration | Yes | `@engine9/id` in the browser, `@engine9/core` on a server, or something else. Decides which branding instructions to send |
| Fields and Identity Level | No | Configured by the site on each request. Nothing is stored per Domain |
| Branding values | No | See [Branding](#branding). Check them before replying |

Reply when something is missing:

```text
To add your site to Delegate we need: the exact host name(s) people will
sign in from (for example www.example.org), a contact email for the person
responsible (it does not need to be at that domain), your organization name,
and whether the site uses @engine9/id, @engine9/core, or something else.
We will also ask you to add one DNS TXT record to prove you control the domain.
```

Decline, or pass to a person, when:

- The host is already allowed (`.engine9.ai`, `.engine9.io`, `localhost:3000`–`3003`).
- The host is `localhost` on another port, an IP address, or a public host over `http`.
- The name imitates another organization (`examp1e-login.org`), or the requester has no visible connection to the Domain.
- The request is for a session-bridge secret or a wildcard suffix. Those are for engine9 API hosts only.

Rule: Never grant a customer Domain a wildcard suffix, a session-bridge suffix, or a session-bridge secret.

### 2. Verify control with a DNS TXT record

Generate one value per host and keep it with the request:

```bash
echo "delegate-verification=$(openssl rand -hex 16)"
```

Send it to the **contact email** (not only to the thread the request came
from), so a reply also proves the address works:

```text
Add a TXT record named _delegate.<host> (for www.example.org the name is
_delegate.www.example.org) with the value <token>. Leave it in place: we may
check it again, and removing it can disable sign-in.
```

Continue only when the exact value comes back:

```bash
dig +short TXT _delegate.www.example.org @1.1.1.1
```

For `host:port`, verify the host.

Rule: The TXT record is on the site's own host (`_delegate.<host>`). It is the same record whichever Delegate host the site signs in through; it is not on `delegate.network` or `delegate.engine9.ai`.

Rule: Accept only the DNS TXT record as proof of control. An email from an address at the Domain, or a file on the website, does not count.

### 3. Register (engine9 operator)

From the Delegate repository, logged in to the production Cloudflare account:

```bash
./scripts/addDomain.sh --email alex@gmail.com www.example.org "Example Org"
```

The script refuses to allow a Domain without `--email`. It saves the contact
on the registry row (`contact_email`) and writes the allow cache. Run it once
for each host. Record the TXT value and verification date with the request.

Smoke test. With no cookies, `min_level=1` sends the browser to sign in:

```bash
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' \
  "https://delegate.network/identity/authorize?domain=www.example.org&return_to=https%3A%2F%2Fwww.example.org%2F&min_level=1"
```

Expected: `302` to `https://delegate.network/login?...`.

A site that still uses the old host answers the same way from `https://delegate.engine9.ai`, with a redirect to `https://delegate.engine9.ai/login?...`. Run the same command with that host to confirm.

### 4. Send integration instructions

Tell the site:

- Verify every Identity Token: ES256 against `https://delegate.network/.well-known/jwks.json`, plus `iss`, `aud` equal to their Domain, and `exp`. `@engine9/id` and `@engine9/core` already do this.
- Add a **Change your Delegate information** button (an authorize request with `prompt=select`) wherever a signed-in person could get stuck: next to Log out, on access-denied messages, and on the login page.
- Keep the `_delegate` TXT record in place.
- Send branding on each request, as described below.
- To offer Facebook as well as Google, send `login_providers` on each request (see Sign-in providers below).

## Branding

Rule: Branding is per request, not per Domain. Delegate stores none of it, and always shows the Domain under the name. Invalid values are dropped silently.

The sign-in email uses the name, color, and logo. Every other option styles
the `/login` page only.

| `@engine9/id` key | Wire parameter | Rule |
| --- | --- | --- |
| `name` | `brand_name` | 1–60 characters, no `://` or `www.`. Default: the Domain |
| `color` | `brand_color` | `#rgb` or `#rrggbb`, for buttons and the monogram. Text on it is white or near-black. Default `#1a73e8` |
| `logo` | `brand_logo` | `https` URL on the Domain's host or a subdomain of it, up to 512 characters, shown about 48px tall. Without it, a monogram of the name's first letter |
| `wordmark` | `brand_wordmark` | Wide logo, same URL rule. Up to 240×120 at the top of `/login`, in place of the logo |
| `background` | `brand_background` | Page color. The card is tinted from it unless `card` is set |
| `text` | `brand_text` | Text on the background. Ignored below 4.5:1 contrast |
| `card` | `brand_card` | Card color, in place of the tint |
| `cardText` | `brand_card_text` | Text on the card. Ignored below 4.5:1 contrast, and ignored without `card` |
| `font` | `brand_font` | `system`, `sans`, `serif`, `rounded`, `mono`, or a Google Fonts family (letters, digits, spaces, up to 40 characters) |
| `google` | `brand_google` | `standard` (Google's white button), `brand` (brand color, "Continue with Google"), or `skip` (go straight to Google when it is the only choice, with no Facebook in `login_providers`; acts as `brand` in a popup) |

Before replying, check the requester's values the way Delegate will:

- Is the logo or wordmark `https`, on their own host or a subdomain of it? A logo on a CDN at another host is dropped.
- Do `text` and `cardText` reach 4.5:1 contrast against their backgrounds?
- Is the name 60 characters or fewer, without `www.` or a URL?

### Set branding

`@engine9/id`, for every request:

```js
import { createEngine9Id } from '@engine9/id';

const id = createEngine9Id({
  domain: 'www.example.org',
  brand: {
    name: 'Example Org',
    color: '#0b6e4f',
    logo: 'https://www.example.org/logo.png',
    font: 'IBM Plex Sans',
    google: 'brand',
  },
});
```

For one request only, pass `brand` to `requestIdentity({ ... })`.

`@engine9/core` on a server:

```js
const url = auth.identityUrl({ returnTo, brand: { name: 'Example Org', color: '#0b6e4f' } });
```

Anything else: add the parameters to the authorize URL:

```text
https://delegate.network/identity/authorize?domain=www.example.org
  &return_to=https%3A%2F%2Fwww.example.org%2Fauth%2Fdelegate
  &brand_name=Example%20Org&brand_color=%230b6e4f
  &brand_logo=https%3A%2F%2Fwww.example.org%2Flogo.png
```

## Disable, hand over, or respond to a compromise

```bash
./scripts/addDomain.sh --disable www.example.org
```

New sign-ins fail within about a minute. The stored name and contact email
are kept.

When the Domain changed hands or was compromised, also revoke its Grants:

```bash
./scripts/addDomain.sh --disable --revoke-grants www.example.org
```

Revoking ends silent sign-in for returning Users. Each User sees the consent
page again the next time any owner of the Domain asks. `--revoke-grants`
also works without `--disable`, for a verified new owner whose Domain stays
enabled.

Rule: Delegate cannot recall Identity Tokens it has already issued. They stay valid until `exp`: 8 hours by default, and up to 30 days when the site asked for a longer `expires_in`. The site and any server that accepts its tokens must stop trusting them.

Email the contact on file about every disable or revoke.

## Sign-in providers

Rule: Providers are chosen per request, not per Domain. Delegate stores none of it. A request can choose among the providers Delegate's operator has enabled (`ENABLED_PROVIDERS`); it cannot turn one on. Unknown values are dropped, and the default is Google only.

| `@engine9/id` key | Wire parameter | Rule |
| --- | --- | --- |
| `loginProviders` | `login_providers` | Comma list in button order from `google` and `facebook`. Default `google` |

Rule: A Facebook sign-in counts as Level 3, but its email is never marked verified, and Facebook may return no email. Sites that check the token's `auth.provider` must accept `google.com` and `facebook.com`.

Rule: One User per email address. If a Facebook email already belongs to a Google User, Delegate asks the person to continue with Google once, then adds Facebook to that User. Both buttons then reach the same Domain UNID.

`@engine9/id`, for every request or one request:

```js
const id = createEngine9Id({ domain: 'www.example.org', loginProviders: ['google', 'facebook'] });
await id.requestIdentity({ minLevel: 1, loginProviders: ['google', 'facebook'] });
```

`@engine9/core` on a server:

```js
const url = auth.identityUrl({ returnTo, loginProviders: ['google', 'facebook'] });
```

Anything else: add `&login_providers=google,facebook` to the authorize URL.

If Facebook does not appear, Delegate's operator has not added `facebook.com` to `ENABLED_PROVIDERS`, or the Facebook provider is not set up in Firebase and Meta.

## Google sign-in redirect URI (engine9 operator)

Delegate signs in with Google through Firebase on the host the page loaded from. Each Delegate host is a Firebase authorized domain and needs its own redirect URI on the Google OAuth client.

| Delegate host | Authorized redirect URI |
| --- | --- |
| `delegate.network` | `https://delegate.network/__/auth/handler` |
| `delegate.engine9.ai` | `https://delegate.engine9.ai/__/auth/handler` |

Rule: Add the `delegate.network` redirect URI and authorized domain before a site moves to it. A missing URI fails Google sign-in on that host with `redirect_uri_mismatch`.

Rule: Keep the `delegate.engine9.ai` redirect URI in place until the old host is retired. Removing it breaks every site that still uses it.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `400 {"error":"invalid_domain"}` | The host is not registered, the request used a different host (`www` versus apex), or the Domain is disabled | Register the exact host. A deny can be cached for up to 10 minutes after an earlier failed try |
| `return_to` on another host fails | `return_to` must be on the `domain` host | Use a callback on the same host |
| Plain `http` page | Delegate refuses `http` except on `localhost` | Serve the site over `https` |
| Facebook button missing | `facebook.com` is not in Delegate's `ENABLED_PROVIDERS`, the request lacks `login_providers=…facebook`, or the Domain is not registered (unregistered hosts get Google only) | Ask the operator to enable it; send `login_providers` |
| Logo or font missing | The value failed validation and was dropped | Host the logo on the Domain over `https`; use a plain Google Fonts family name |
| Google sign-in shows `redirect_uri_mismatch` on one host | That host's `/__/auth/handler` URI or authorized domain is missing | Add it to the Google OAuth client and Firebase (see Google sign-in redirect URI) |
| Token rejected with an issuer error | The site's configured Delegate URL differs from the host that issued the token | Make `delegateUrl` / `DELEGATE_URL` the host the browser was sent to |
| Returning Users get no consent page after the Domain changed hands | Their Grants still cover the request | Run `--revoke-grants` |

## Related documentation

- [Identity Levels](../e9-identity-levels/README.md)
- [`@engine9/id` protocol, Branding](https://github.com/engine9-ai/id/blob/main/docs/protocol.md#branding)
