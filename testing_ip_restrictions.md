# Testing IP-based restrictions under KTD

Several Koha features restrict or grant access based on the client's IP address.
Testing these under KTD requires understanding the network topology between your
browser, the Traefik reverse proxy, and the Koha container.

## Features that use IP-based logic

| System preference | Behavior |
|---|---|
| `OpacSuppressionByIPRange` | IPs matching the prefix see suppressed records; others don't |
| `RestrictedPageLocalIPs` | Only matching IPs can access the restricted OPAC page |
| `ILS-DI:AuthorizedIPs` | ILS-DI service is restricted to listed IPs |
| `SelfCheckAllowByIPRanges` | Self-checkout only from specified IP ranges |

All of these compare against `$ENV{'REMOTE_ADDR'}` at runtime.

## How REMOTE_ADDR is determined in KTD

KTD instances run behind a **Traefik reverse proxy**. The request path is:

```
Browser (host machine)
  → Traefik (proxy-proxy-1, IP 172.18.0.x on the `proxy` network)
    → Apache in the Koha container
```

Without intervention, `REMOTE_ADDR` inside Koha would be Traefik's container IP
(e.g. `172.18.0.2`), not your machine's IP.

However, Koha includes `Koha::Middleware::RealIP` which reads the
`X-Forwarded-For` header (set by Traefik) and overwrites `REMOTE_ADDR` with the
original client IP. The actual value you'll see depends on your Docker network
configuration.

## Discovering your effective IP

The simplest way to find out what IP Koha sees for your requests:

```bash
# From KTD shell, check the Apache access log after making a request:
ktd --name bug_XXXXX --shell --run "tail -1 /var/log/koha/kohadev/opac-access.log"
```

Or add a temporary debug line:

```perl
warn "REMOTE_ADDR = " . $ENV{'REMOTE_ADDR'};
```

Typical values you'll see:

| Access method | REMOTE_ADDR seen by Koha |
|---|---|
| `curl` from inside the container | `127.0.0.1` |
| Browser via Traefik vhost (e.g. `http://bug_XXXXX.koha`) | Host's IP on the Docker bridge (e.g. `192.168.65.1` on macOS, `172.x.0.1` on Linux) |
| `cy.visit()` in Cypress (running inside KTD) | Container's loopback or bridge IP |

## Strategies for testing IP-restricted features

### Manual testing in the browser (most common)

When testing features like `OpacSuppressionByIPRange` through your browser,
you need to know what IP Koha sees for your requests so you can set the
preference accordingly.

**Step 1: Find your effective IP**

The host machine's IP as seen by Koha is the gateway of the Docker `proxy`
network (the network shared with Traefik):

```bash
docker network inspect proxy --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
```

This typically returns something like `172.18.0.1` (Linux) or `192.168.65.1`
(macOS with Docker Desktop).

You can verify by checking the access log after visiting the OPAC in your
browser (e.g. `http://bug_XXXXX.koha/`):

```bash
ktd --name bug_XXXXX --shell --run "tail -1 /var/log/koha/kohadev/opac-access.log | awk '{print \$1}'"
```

**Step 2: Set the preference to match (or not)**

To test the "IP in range" path (e.g. user CAN see suppressed records):
```
OpacSuppressionByIPRange = 192.168.
```

To test the "IP NOT in range" path (e.g. user cannot see suppressed records):
```
OpacSuppressionByIPRange = 10.99.
```
(any prefix that doesn't match your actual IP)

**Step 3: Clear the cache and reload**

After changing the preference, clear Memcached so the new value takes effect:

```bash
ktd --name bug_XXXXX --shell --run "koha-shell kohadev -c 'memcached-tool localhost:11211 flush_all'"
```

Then reload the OPAC page in your browser.

### Typical IPs seen in KTD

| Host OS | IP Koha sees from browser | Explanation |
|---|---|---|
| macOS (Docker Desktop) | `192.168.65.1` | Docker Desktop's VM gateway |
| Linux (native Docker) | `172.x.0.1` | Docker bridge gateway |
| Inside container (`curl localhost`) | `127.0.0.1` | Loopback |

These can vary depending on Docker version and network configuration. Always
verify with the access log.

### Strategy for unit tests (prove)

For automated tests, mock the IP directly:

```perl
# In a test file
local $ENV{'REMOTE_ADDR'} = '192.168.1.100';
# Now test with an IP that matches the range

local $ENV{'REMOTE_ADDR'} = '10.0.0.1';
# Now test with an IP outside the range
```

When running `prove` inside the KTD container, requests to `localhost` have
`REMOTE_ADDR = 127.0.0.1`. Setting the IP range to `127.` will match.

The most robust test pattern covers both "in range" and "out of range":

```perl
subtest 'OpacSuppressionByIPRange tests' => sub {
    plan tests => 2;

    t::lib::Mocks::mock_preference('OpacSuppression', 1);

    # IP within range — suppressed records ARE visible
    t::lib::Mocks::mock_preference('OpacSuppressionByIPRange', '127.');
    local $ENV{'REMOTE_ADDR'} = '127.0.0.1';
    # ... assert suppressed records are included ...

    # IP outside range — suppressed records are hidden
    local $ENV{'REMOTE_ADDR'} = '10.0.0.1';
    # ... assert suppressed records are excluded ...
};
```

## Common pitfall: regex-based matching

The IP range check in Koha uses a **regex prefix match**:

```perl
my $ip_in_range = ( $ip_address =~ /^$ip_range/ );
```

This means `OpacSuppressionByIPRange = 172.` will match `172.18.0.2`,
`172.26.0.4`, etc. It's not CIDR notation — it's a simple string prefix matched
as a regex. Be careful with dots (`.` matches any character in regex), though in
practice with IP addresses this rarely causes false positives.

## Note on Cypress tests

Cypress integration tests running inside KTD use `cy.intercept()` to mock API
responses. For IP-based behavior testing in Cypress, you typically can't control
`REMOTE_ADDR` from the browser side. Instead:

1. Set the preference to match the IP that Koha sees from Cypress (usually the
   container's bridge gateway)
2. Or test the logic at the unit level with `prove` and mock the IP there
3. For e2e tests that need both paths, use separate `before()` blocks that
   toggle the preference value
