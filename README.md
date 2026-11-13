# HTTP Cache Freshness Calculator

Paste a raw HTTP response header block and see, for a browser cache, a shared cache and a CDN cache side by side, whether it may be stored, its RFC 9111 freshness lifetime and which rule produced it, its current age, and whether it is fresh or stale.

**Live demo:** https://0xelitesystem.github.io/http-cache-freshness-calculator/

## Use

1. Paste the response headers, from `curl -I`, a browser's raw header view or a log. A status line such as `HTTP/1.1 200 OK` or `HTTP/2 200` is read if present; otherwise pick the status from the dropdown. Lines from `curl -v` (`< ` and `> ` prefixes) are handled, and request lines are skipped with a warning.
2. Set the clock if it matters. `now`, `request_time` and `response_time` all default to the response's `Date` value. Type an HTTP-date or a UTC time such as `2026-10-01T12:15:00Z` to see the response age.
3. Tick "request carried Authorization" if it did, because that changes what a shared cache may store.
4. Read the three cards. Each says whether that cache may store the response and why, the freshness lifetime and the rule that produced it, the current age, and the verdict: fresh, stale, must revalidate, cache-specific or not stored. Below them, the age calculation shows every intermediate term, and a warnings strip lists duplicates, invalid values and ignored fields.

The page opens on a split verdict. With `Date` and `now` equal and these headers:

```
Age: 900
Cache-Control: max-age=300, s-maxage=1200
CDN-Cache-Control: max-age=7200
```

the private cache uses `max-age` (300 s) and the response is stale; a shared cache uses `s-maxage` (1200 s) and it is fresh with 300 s left; a CDN cache with `CDN-Cache-Control` on its target list ignores `Cache-Control` and uses 7200 s. Nobody is misbehaving: each cache is following its own rule.

### When this is useful

- A page stays stale in the browser while the CDN keeps serving it, or the other way round, and you want to see which directive each layer is reading.
- You are adding `CDN-Cache-Control` or `s-maxage` and want to check the lifetime each cache ends up with before deploying.
- A response with `Expires: 0`, a duplicate `max-age` or a quoted value behaves oddly and you want the RFC's reading of it.
- An authenticated API response is or is not being cached by a proxy and you need the storage rule that decides it.

## What it models

Each rule is labelled on the page with its section.

- **Storage**, [RFC 9111 section 3](https://www.rfc-editor.org/rfc/rfc9111.html#section-3): `no-store`; "The unqualified private response directive indicates that a shared cache MUST NOT store the response"; a request with `Authorization` and no `must-revalidate`, `public` or `s-maxage` (section 3.5); and the list of things the response must contain at least one of, including a heuristically cacheable status. Those statuses come from [RFC 9110 section 15.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.1): 200, 203, 204, 206, 300, 301, 308, 404, 405, 410, 414 and 501.
- **Freshness lifetime**, [section 4.2.1](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2.1), first match: "If the cache is shared and the s-maxage response directive (Section 5.2.2.10) is present, use its value, or If the max-age response directive (Section 5.2.2.1) is present, use its value, or If the Expires response header field (Section 5.3) is present, use its value minus the value of the Date response header field". For a repeated directive, "either the first occurrence should be used or the response should be considered stale", and the page shows both readings.
- **Expires**, [section 5.3](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.3): "If a response includes a Cache-Control header field with the max-age directive (Section 5.2.2.1), a recipient MUST ignore the Expires header field", a shared cache also ignores it under `s-maxage`, and "A cache recipient MUST interpret invalid date formats, especially the value "0", as representing a time in the past (i.e., "already expired")."
- **Heuristic freshness**, [section 4.2.2](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2.2): "A cache MUST NOT use heuristics to determine freshness when an explicit expiration time is present in the stored response." With `Last-Modified`, the page multiplies the fraction you choose by the interval since then. The default is 10 percent because the RFC says "A typical setting of this fraction might be 10%."; it is a setting, not a rule, and "This specification does not provide specific algorithms, but it does impose worst-case constraints on their results." Without `Last-Modified` the lifetime is shown as cache-specific and nothing is computed.
- **Age**, [section 4.2.3](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2.3): `apparent_age`, `response_delay`, `corrected_age_value`, `corrected_initial_age` and `resident_time`, then "current_age = corrected_initial_age + resident_time;". The default is the conservative "corrected_initial_age = max(apparent_age, corrected_age_value);"; the alternative the RFC allows, using `corrected_age_value` alone, is one click away. Freshness is "response_is_fresh = (freshness_lifetime > current_age)", so equal means stale.
- **no-cache**, [section 5.2.2.4](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.2.4): the response may be stored, but it "MUST NOT be used to satisfy any other request without forwarding it for validation and receiving a successful response". It never means do not store.
- **Overflow**, [section 1.2.2](https://www.rfc-editor.org/rfc/rfc9111.html#section-1.2.2): "the cache MUST consider the value to be 2147483648 (2^31) or the greatest positive integer it can conveniently represent."
- **CDN column**, [RFC 9213](https://www.rfc-editor.org/rfc/rfc9213.html): the targeted field is parsed as a Structured Fields Dictionary ([RFC 8941 section 3.2](https://www.rfc-editor.org/rfc/rfc8941.html#section-3.2)), so `Max-Age=600` (uppercase key) is a parse error. "If a targeted field in a given response is empty, or a parsing error is encountered, that field MUST be ignored by the cache". Otherwise the cache "MUST select the first (in target-list order) field with a valid, non-empty value and use its value to determine the caching policy for the response, and it MUST ignore the Cache-Control and Expires header fields in that response". "In particular, no-store and no-cache make max-age inoperative, and unrecognized extension directives are ignored." A decimal `max-age` is not coerced to an integer. The target list is editable, so a vendor field can be put ahead of `CDN-Cache-Control`. The parser follows RFC 8941, the version RFC 9213 cites; the rfc-editor lists RFC 8941 as obsoleted by RFC 9651, which adds Date and Display String values that this parser reports as a parse error.
- **Dates**, [section 4.2](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2): matched case-insensitively, all three HTTP-date formats accepted, a zone other than GMT treated as invalid for expiration, and everything computed in UTC.

Recognised but not computed: `stale-while-revalidate` and `stale-if-error`, `must-understand`, `Vary` (listed with its fields), 304 responses, request directives and `Pragma`. Only GET and HEAD are treated as cacheable. The page models the RFCs. It makes no claim about how any particular browser, proxy or CDN product behaves.

RFC 9111 is STD 98, published June 2022, and obsoletes RFC 7234. RFC 9213 is Standards Track, published June 2022.

## Self test

The page runs its self test on load and prints a live pass and fail count. It covers each rule above with at least one fixture, the five worked examples (the split verdict, the heuristic with status 200 and with 302, `Expires: 0` with and without `max-age`, `no-store` next to `CDN-Cache-Control`, and a lifetime equal to the age), and a property test that generates random `max-age`, `s-maxage`, `Expires` and `Age` combinations from a fixed seed and checks the evaluator against a separately written reference of sections 4.2.1 and 4.2.3. The last row is a positive control written to fail.

## Why this exists

One response, three caches, three correct answers. RFC 9111 gives a shared cache `s-maxage` first and a private cache skips it; RFC 9213 tells a CDN cache with `CDN-Cache-Control` on its target list to ignore `Cache-Control` and `Expires` entirely. "The CDN served it for an hour, the browser for ten minutes" is every party following its own rule, and the only way to see it is to compute all three from the same headers.

Single file, no build step, no network call, no tracking, MIT licensed. Paste production headers without wondering where they go.

## Privacy

Everything runs in your browser. The page makes no network call of any kind: no code, font, analytics or telemetry request, and nothing you paste is transmitted. A Content-Security-Policy in the page forbids network connections. The only thing it stores is your light or dark theme choice, in your browser's local storage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/http-cache-freshness-calculator.git
cd http-cache-freshness-calculator
```

Open `index.html` in a browser. Or serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence and provenance

This is an independent tool. It is not affiliated with or endorsed by the IETF or any CDN vendor. Behaviour was implemented from the RFC text; no code is reproduced here.

## Third-party notices

**RFC 9111, HTTP Caching** (https://www.rfc-editor.org/rfc/rfc9111.html) and **RFC 9213, Targeted HTTP Cache Control** (https://www.rfc-editor.org/rfc/rfc9213.html). Holder line, copied from the Copyright Notice of each: "Copyright (c) 2022 IETF Trust and the persons identified as the document authors. All rights reserved." Both are subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info). What was taken: the short rule sentences and formulas quoted on the page and in this README.

**RFC 9110, HTTP Semantics** (https://www.rfc-editor.org/rfc/rfc9110.html) and **RFC 8941, Structured Field Values for HTTP** (https://www.rfc-editor.org/rfc/rfc8941.html). Behaviour reimplemented from the text (the heuristically cacheable status list, the Date rules, the HTTP-date formats and the Dictionary parsing algorithm); no text is quoted.

## Related

- https://github.com/0xelitesystem/http-header-analyzer
- https://github.com/0xelitesystem/caching-basics-reference
- https://github.com/0xelitesystem/http-status-codes-reference

## License

MIT. Copyright (c) 2026 RBL Industries LLC. See [LICENSE](LICENSE).
