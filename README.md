# csc436-hw2-ash-2231360

**Name:** Caitlin Ash

**ID:** 2231360

**Commit SHA or Tag:** `hw2-submission`

**Platform/OS Version:** Windows 11 Pro, 64-bit

# Task Reports

## Task 1 - Trace and explain a public delegation path

```powershell
dig +trace depaul.edu A
dig +trace www.depaul.edu A
```

*See [evidence](./evidence/01-dig-trace.txt) for task 1*

**Query 1:**

**Root (`.`):** `75.75.75.75` answered with a referral, indicated by the list of root nameservers and `NS` records.

**Top-level domain (`.edu`):** `h.root-servers.net` answered with a referral, indicated by the list of `.edu` TLD 
nameservers and `NS` records.

**Second-level domain (`depaul.edu`):** `k.edu-servers.net` answered with a referral, indicated by DePaul's authoritative nameservers (`ns1-ns4`) and `NS` records.

**Authoritative answer (`depaul.edu`):** `ns4.depaul.edu` answered with an answer, indicated by the requested `A` record containing the IP address `64.239.109.1`.

---

**Query 2:**

**Root (`.`):** `75.75.75.75` answered with a referral, indicated by the list of root nameservers and `NS` records.

**Top-level domain (`.edu`):** `i.root-servers.net` answered with a referral, indicated by the list of `.edu` TLD 
nameservers and `NS` records.

**Second-level domain (`depaul.edu`):** `d.edu-servers.net` answered with a referral, indicated by DePaul's authoritative nameservers (`ns1-ns4`) and `NS` records.

**Authoritative answer (`www.depaul.edu`):** `ns2.depaul.edu` answered with an answer, indicated by the `CNAME` record (not the requested `A` record) containing the address `22b8cde06cfddd89.vercel-dns-013.com` (`www.depaul.edu` is mapped to another domain name).

## Task 2 - Read real zone's record choices

```powershell
dig "@1.1.1.1" depaul.edu SOA +noall +answer
dig "@1.1.1.1" www.depaul.edu A +noall +answer
dig "@1.1.1.1" depaul.edu MX +noall +answer
dig "@1.1.1.1" depaul.edu TXT +noall +answer
dig "@1.1.1.1" github.com CAA +noall +answer
dig "@1.1.1.1" depaul.edu CAA
```

*See [evidence](./evidence/02-record-types.txt) for task 2*

**SOA:** The last field of the `SOA` (`900`) is the minimum negative-cache TTL. This tells the operator that there was a failed DNS lookup and it won't keep trying the server again and again.

**CNAME:** `www.depaul.edu` acts as an alias and points to `22b8cde06cfddd89.vercel-dns-013.com`. The `CNAME` has a long TTL because the alias rarely changes, but the target `A` TTLs are shorter because the host servers can change.

**MX:** Mail delivery uses a hostname instead of a raw IP address because `A` records already handle this. `MX` records allow mail routing to adapt to any changes in the `A` record.

**TXT:** I am able to recognize the third party verifications for Zoom, autodesk, google, sitecore, apple, atlassian, etc. Additionally, these all have long TTLs, allowing for fewer future queries. 

**CAA:** `github.com` allows standard certificates from `digicert.com`, `globalsign.com`, `letsencrypt.org`, and `sectigo.com`. It also allows wildcard certificates for `digicert.com`, `letsencrypt.org`, and `sectigo.com`. From my output, it looks like there are no restrictions listed, as they all have a `0` flag. `depaul.edu`'s `NODDATA` result (indicated by `ANSWER: 0` and `STATUS: NOERROR`) means that the domain name exists but no `CAA` records have been configured. There are no restrictions on certificate issuance. 

## Task 3 - Compare authoritative truth with recursive caches

```powershell
$zone = "depaul.edu"
$name = "www.depaul.edu"
$ns = (dig +short NS $zone | Select-Object -First 1).Trim()
Get-Date -UFormat '%Y-%m-%dT%H:%M:%SZ'
dig "@$ns" "$name" A +norecurse
dig "@1.1.1.1" "$name" A
dig "@8.8.8.8" "$name" A
dig "@9.9.9.9" "$name" A
```

*See [evidence](./evidence/03-resolvers.txt) for task 3*

The first query is authoritative, indicated by the `aa` flag that appears in the header. This was answered with a `CNAME` record where `www.depaul.edu` is mapped to `22b8cde06cfddd89.vercel-dns-013.com`.

The rest of the queries were recursive answers. In the second query, we are given the same `CNAME` record from the server but we are also given two `A` records `64.239.109.193` and `64.239.123.193` which are both mapped to the `CNAME` record. These are the two servers that Vercel is using to serve `www.depaul.edu`. The third query has the same `CNAME record` but provided different `A` records `64.239.123.129` and `64.239.109.129`. Finally, the fourth query has the same `CNAME` record and two different `A` records `64.239.109.1` and `64.239.123.1`.

There is only a `CNAME` record in the authoritative answer, so the RRset consists of this one record. However, in the second query, the host `22b8cde06cfddd89.vercel-dns-013.com` has two entries with an `IN A` record to two different IP addresses, making it an RRset.

**Apparent Cache Age Calculations:**

Authoritative TTL (`86,400`) - Resolver TTL `1.1.1.1` (`53,870`) = `32,530` apparent seconds since `1.1.1.1` cache last refreshed

Authoritative TTL (`86,400`) - Resolver TTL `8.8.8.8` (`21,600`) = `64,800` apparent seconds since `8.8.8.8` cache last refreshed

Authoritative TTL (`86,400`) - Resolver TTL `9.9.9.9` (`43,200`) = `43,200` apparent seconds since `9.9.9.9` cache last refreshed

## Task 4 - Cutover plan from public evidence 

```
Production name     : www.campuspulse.example
Current target      : old-edge.vendor.example
New target          : new-edge.vendor.example
Current TTL         : 86400
Temporary TTL       : 300
Cutover window      : Friday 20:00 UTC
Rollback target     : old-edge.vendor.example
Success criteria    : authority serves the new target, then two selected resolvers
                      show the new target after their cached TTLs expire
```

**1. When to lower the TTL and why the old TTL controls the waiting period.**

The current TTL is `86400`, which equates to 24 hours. The TTL should be lowered to `300` (5 minutes) at least 24 hours before the cutover window of `Friday 20:00 UTC`. The old TTL of `86400` still controls the old record. Lowering the TTL after the cutover does nothing for the caches already holding the old record at the old TTL. 

**2. The earliest safe cutover time.**

The earliest safe cutover time is `Friday at 20:00 UTC` because the TTL was successfully lowered 24 hours prior. This ensures no resolver is holding a cache longer than 5 minutes once the cutover occurs. 

**3. What exact dig commands you would run against authority, 1.1.1.1, and 8.8.8.8.**

I would run `dig` commands to check the authroity and resolver behavior during and after the cutover.

**Check Authority:** `dig "@ns1.campuspulse.example' www.campuspulse.example CNAME`

**Check `1.1.1.1` Resolver:** `dig "@1.1.1.1" www.campuspulse.example CNAME`

**Check `8.8.8.8` Resolver:** `dig "@8.8.8.8" www.campuspulse.example CNAME`

**4. What evidence would trigger rollback.**

If the authoritative nameserver responds with an error (`SERVFAIL`, `NXDOMAIN`) or points to an unintended target after cutover, or if the new-edge deployment fails its application-layer health checks.

**5. What evidence is not a rollback trigger because it is expected cache lag.**

Seeing `old-edge.vendor.example` on public resolvers (`1.1.1.1` or `8.8.8.8`) immediately after `20:00 UTC`. This is normal resolver cache lag; as seen in task 3, public resolvers track their own expiration countdowns based on when they last fetched the record.

**6. When it is safe to retire the old target.**

It is safe to retire `old-edge.vendor.example` only after the temporary TTL (5 minutes) plus a generous safety buffer (e.g., 24 to 48 hours) has passed, ensuring all long-tail or misbehaving resolvers have completely flushed their caches.

**7. Then add a two-sentence reflection connecting the plan to your Task 3 public DNS evidence.**

As observed in the [evidence](./evidence/03-resolvers.txt) for task 3 analysis, where resolvers like `1.1.1.1` and `8.8.8.8` displayed heavily staggered TTLs ranging from `21600` to `53870`, failing to pre-lower a 24-hour TTL before a cutover would leave critical web traffic routed to stale infrastructure for hours. Proactively shortening the TTL forces recursive caches to expire quickly, guaranteeing a predictable and synchronized migration window.

## Task 5 - Diagnose supplied HTTP failures using VSCode `.http` requests

*See [evidence](./evidence/05-http-faults.txt) for task 5*

### Fault 1A and 1B

**Annotated Request/response pair**

For 1A, it is a `GET` request, indicating that we are trying to fetch something from `http://127.0.0.1:8081/healthz`, with the `Host` being the virtual host `status.campuspulse.example`. This tells the server which host should handle the request. The response returns a `200 OK` status code, meaning there was a successful match and execution of the request. 

For 1B, it is also a `GET` request to `http://127.0.0.1:8081/healthz` with a different unknown `Host` of `www.campuspulse.example`. The response returns a `404 Not Found` status code, indicating that the unknown virtual host was rejected. `X-Served-By` shows which node handled the request. 

**The diagnosis**

The problem is that the unknown host is missing an edge configuration. `www.campuspulse.example` isn't binded to the shared edge proxy's site configurations. The problem is not DNS because the request successfully reached the server socket.

**The confirming request**

`curl.exe --noproxy '*' -i http://127.0.0.1:8081/healthz -H "Host: www.campuspulse.example"`

```
X-Served-By: shared-edge-7

404 Not Found

This shared edge has no site configured for the requested hostname.
Requested Host: www.campuspulse.example
```

**The fix**

I would configure the edge server to bind `www.campuspulse.example` to the appropriate site.

### Fault 2A

**Annotated Request/response pair**

For 2A, it is a `GET` request, indicating we are trying to fetch something from `http://127.0.0.1:8082/`, with the `Host` configured as the site root `campuspulse.example` on socket `8082`. The response returns a `301 Moved Permanently` status code, indicating that the redirection triggered. The resource has been moved for good.

**The diagnosis**

The problem is that a cyclic routing misconfiguration at the site root `http://127.0.0.1:8082/` issues a `301 Moved Permanently` redirect pointing right back to the requested URI.

**The confirming request**

`curl.exe --noproxy '*' -i http://127.0.0.1:8082/ -H "Host: campuspulse.example"`

```
HTTP/1.1 301 Moved Permanently

Location: http://www.campuspulse.example:8082/
```

**The fix**

I would correct the URL rewrite and routing rules in the application or proxy configuration. 

### Fault 2B and 2C

**Annotated Request/response pair**

For 2B, it is a `POST` request, indicating that we are trying to write something to the endpoint `http://127.0.0.1:8082/legacy/incidents`. The `Host` is `campuspulse.example`. The response returns with a `302 Found` status code, indicating that redirection was triggered. The resource is only somewhere else temprorarily. 

For 2C, it is another `POST` request to the endpoint `http://127.0.0.1:8082/api/incidents`. The `Host` is also `campuspulse.example`. The response returns with a `201 Created`, indicating that the request was successful and led to the creation of a new resource on the server.

**The diagnosis**

The problem is that the legacy endpoint `http://127.0.0.1:8082/legacy/incidents` is configured to intercept `POST` requests and issue a `302 Found` redirect instead of accepting payload directly. A `302` redirect is not a safe method-preservation mechanism for `POST`. 

**The confirming request**

`curl.exe --noproxy '*' -i http://127.0.0.1:8082/legacy/incidents -H "Host: campuspulse.example"`

```
HTTP/1.1 302 Found

Location: http://campuspulse.example/api/incidents
```

This differs from the `201 Created` status code returned from the request endpoint `http://127.0.0.1:8082/api/incidents`. Additionally, the new location is shown as the working endpoint from 2C. 

**The fix**

I would update the routing configuration in the application or proxy to correctly handle or proxy legacy payload routes.

### Fault 3A, 3B, and 3C

**Annotated Request/response pair**

For 3A, it is a `GET` request, indicating we are trying to fetch something from `http://127.0.0.1:8084/healthz`. The `Host` is configured as `campuspulse.example`. The response returns a `301 Moved Permanently` status code, indicating that the redirection triggered. The resource has been moved for good. `x-app-saw-forwarded-headers: x-forwarded-protocol, x-forwarded-for, x-forwarded-host` also indicates that there was multiple redirects.

For 3B, it is also a `GET` request to fetch something from `http://127.0.0.1:8083/healthz`. The `Host` is also configured as `campuspulse.example`. The response returns a `301 Moved Permanently` status code, indicating that the redirection triggered. The resource has been moved for good.

For 3C, it is also a `GET` request to fetch something from `http://127.0.0.1:8083/healthz`. The `Host` is also configured as `campuspulse.example`. The response returns a `200 OK` status code, indicating that there was a successful match and execution of the request. 

**The diagnosis**

The reverse proxy terminates TLS but fails to inject the `X-Forwarded-Proto` header, causing the backend application to assume an insecure request and trigger an infinite HTTPS redirect loop.

**The confirming request**

`curl.exe --noproxy '*' -i http://127.0.0.1:8083/healthz -H "Host: campuspulse.example" -H "X-Forwarded-Proto: https"`

```
HTTP/1.1 200 OK

X-App-Saw-XFP: https
X-App-Saw-Forwarded-Headers: x-forwarded-proto
```

Different from `301 Moved Permanently` status code returned by 3A because of redirect loop caused by missing header.

**The fix**

I would configure the reverse proxy to explicitly inject `X-Forwarded-Proto: https` into downstream requests sent to the backend app. 

# Deliberate False Lead and Rejection

**False Lead:** "The application is returning a `404 NotFound` error on `www.campuspulse.example` because DNS has not propagated yet or the domain name resolution is misconfigured."

**How it was rejected:** I ran a `curl` request hitting the exact same local server IP (`127.0.0.1:8081`) with two different Host headers (`status.campuspulse.example` returned `200 OK` and `www.campuspulse.example` returned `404 Not Found`). Since the request successfully reached the server socket without any DNS lookup errors, and the server explicitly replied with "This shared edge has no site configured for the requested hostname," I proved that DNS worked perfectly and the actual issue was a missing virtual host mapping on the reverse proxy. 

# Redaction Attestation

No live secrets, tokens, cookies, session identifiers, or API keys needed to be redacted.