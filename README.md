# DNS-Server

Project 6 for CS 4700. Recursive DNS server + authoritative server for a local zone, written in Python.

## Run

```
./4700dns [--port <port>] <root_ip> <zone_file>
./run configs/<test>.conf
```

Needs `dnslib` and `PyYAML`: `pip3 install dnslib PyYAML`.

## Approach

Single UDP socket bound to 127.0.0.1. Each incoming query is handed to a worker thread so recursion for one client doesn't block others. Upstream queries go to port 60053 on whatever NS IP we're talking to.

Zone file is parsed once at startup with `RR.fromZone`. Domain is taken from the SOA. Queries under the domain are answered directly from the zone (AA=1); CNAMEs are chased locally when the target is in-zone, NS queries include glue A records, and names that don't exist return NXDOMAIN with the SOA in authority. Subdomains with an NS record are returned as referrals (AA=0).

For external queries the resolver starts at the deepest NS it has cached (falling back to the root) and walks down through referrals. After each response it filters records out of the responder's bailiwick before caching or using them. CNAMEs in answers are followed by re-resolving the target. NS-for-zone queries stop at the parent's referral instead of asking the child (avoids issues with truncating servers).

The cache is keyed by (name, type) with per-record expiry. TTLs are decremented on read and expired entries are dropped. Error responses aren't cached. On a cache hit for an NS query, matching A/AAAA records for the NS names are pulled from the cache and included as additional, so cached referrals still come with glue.

## Testing

Ran all 13 provided configs (levels 1-18) against the Khoury reference server — all pass. Also tested auth-zone responses directly in a Python REPL against the sample `example.com.zone` before wiring up recursion.
