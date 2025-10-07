# redir

`redir` is a tiny command-line tool (Perl) that prints the **full redirect chain** for a URL, including “soft” redirects (Refresh header, HTML meta refresh, simple JavaScript `location=`). It’s useful for seeing where a shortlink really lands or why your browser ends up on a “domain for sale” page.

## Features

* Follows hard redirects: 301/302/303/307/308 (`Location:`)
* Detects common soft redirects:

  * `Refresh:` response header (e.g. `0; url=…`)
  * HTML `<meta http-equiv="refresh" content="…url=…">`
  * Simple JS: `location=`, `location.href=`, `location.assign/replace`
* Options for TLS verification, method (`GET`/`HEAD`), header output, JSON, timeouts, etc.
* Normal output is a numbered list of hop URLs; add `--annotate` for status/method/why.

## Installation

1. Save the script as `redir` and make it executable:

```
chmod +x redir
```

2. Ensure these Perl modules are available (most systems will have them; the HTTPS bits may need installing):

```
cpanm LWP::UserAgent LWP::Protocol::https IO::Socket::SSL Net::SSLeay URI JSON::PP
```

Or use your package manager (e.g., `apt`, `dnf`, `brew`) using their Perl module packages.

## Usage

```
redir [options] URL
```

### Options

* `--insecure`
  Disable TLS certificate/hostname verification. **Only use when necessary** (e.g., misconfigured/expired certs).
* `--annotate`
  Show status code and redirect mechanism per hop (e.g., `[301 http]`, `[200 final]`).
* `--json`
  Emit machine-readable JSON instead of text.
* `--max N`
  Maximum hops to follow (default: 20).
* `--timeout S`
  Per-request timeout in seconds (default: 20).
* `--method M`
  HTTP method to use (`GET` default). **Note:** Soft redirect detection requires `GET` because bodies aren’t returned for `HEAD`.
* `--show-headers`
  Print each hop’s response headers (indented under the hop).

## Examples

### Plain chain (strict TLS)

```
redir "https://example.com"
```

### Annotated, with headers, allowing bad TLS (e.g., broken certs)

```
redir --insecure --annotate --show-headers "http://t.co/bhxnq1IJ"
```

**Sample output:**

```
1. http://t.co/bhxnq1IJ  [301 http]  {GET}
    HTTP/1.1 301 Moved Permanently
    Location: https://t.co/bhxnq1IJ
    ...

2. https://t.co/bhxnq1IJ  [301 http]  {GET}
    HTTP/2 301
    Location: https://gu.com/p/32vnc/tw
    ...

3. https://gu.com/p/32vnc/tw  [301 http]  {GET}
4. https://www.gu.com/p/32vnc/tw  [301 http]  {GET}
5. https://www.gu.com/  [302 http]  {GET}
6. https://www.brandforce.com/domain/GU.COM/  [200 final]  {GET}
```

### JSON for tooling

```
redir --insecure --json "http://t.co/bhxnq1IJ" | jq .
```

### Faster, hard-redirects-only

```
redir --method HEAD --annotate "https://example.com"
```

> Tip: If you don’t see expected soft redirects, rerun with `--method GET`.

## Exit codes

* `0` — Completed (chain printed)
* `2` — Usage / argument error
* Non-zero (other) — Network/HTTP errors encountered (still prints whatever hops were collected)

## Security notes

* `--insecure` turns off TLS verification to handle sites with broken/legacy certificates (e.g., old shortlinks). Prefer strict TLS (the default) and only use `--insecure` for troubleshooting.
* This tool **does not execute arbitrary JavaScript** — it only detects straightforward `location=` patterns in HTML. Complex SPA/router flows will appear as “final” unless the server emits redirects or refreshes.

## Known limitations

* Complex client-side redirects (framework routers, delayed timers, obfuscated JS) won’t be followed.
* Some servers only reveal redirects on `GET`. Use `--method GET` if `HEAD` misses hops.
* Meta refresh parsing is best-effort; extremely malformed HTML may be skipped.

## Development

* Code style: modern Perl (`v5.36`), strict/warnings.
* Tests (future): add fixtures for common patterns (HTTP, Refresh, meta, JS).
* Ideas welcome: `--filter-header Location,Set-Cookie`, HAR export, timing per hop.

## Licence

MIT. See `LICENSE` if included; otherwise treat this file as licensed under MIT.

