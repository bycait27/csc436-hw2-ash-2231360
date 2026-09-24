# csc436-hw2-ash-2231360

**Name:** Caitlin Ash

**ID:** 2231360

**Commit SHA or Tag:** `hw2-submission`

**Platform/OS Version:** Windows 11 Pro, 64-bit

## Task Reports

### Task 1 - Trace and explain a public delegation path

`$ dig +trace depaul.edu A`

**Root (`.`):** `75.75.75.75` answered with a referral, indicated by the list of root nameservers and `NS` records.

**Top-level domain (`.edu`):** `h.root-servers.net` answered with a referral, indicated by the list of `.edu` TLD 
nameservers and `NS` records.

**Second-level domain (`depaul.edu`):** `k.edu-servers.net` answered with a referral, indicated by DePaul's authoritative nameservers (`ns1-ns4`) and `NS` records.

**Authoritative answer (`depaul.edu`):** `ns4.depaul.edu` answered with an answer, indicated by the requested `A` record containing the IP address `64.239.109.1`.

`$ dig +trace www.depaul.edu A`

**Root (`.`):** `75.75.75.75` answered with a referral, indicated by the list of root nameservers and `NS` records.

**Top-level domain (`.edu`):** `i.root-servers.net` answered with a referral, indicated by the list of `.edu` TLD 
nameservers and `NS` records.

**Second-level domain (`depaul.edu`):** `d.edu-servers.net` answered with a referral, indicated by DePaul's authoritative nameservers (`ns1-ns4`) and `NS` records.

**Authoritative answer (`www.depaul.edu`):** `ns2.depaul.edu` answered with an answer, indicated by the `CNAME` record (not the requested `A` record) containing the address `22b8cde06cfddd89.vercel-dns-013.com` (`www.depaul.edu` is mapped to another domain name).

### Task 2 - Read real zone's record choices

`$ dig "@1.1.1.1" depaul.edu SOA +noall +answer`

The last field of the `SOA` (`900`) is the minimum negative-cache TTL. This tells the operator that there was a failed DNS lookup and it won't keep trying the server again and again.

`$ dig "@1.1.1.1" www.depaul.edu A +noall +answer`

`www.depaul.edu` acts as an alias and points to `22b8cde06cfddd89.vercel-dns-013.com`. The `CNAME` has a long TTL because the alias rarely changes, but the target `A` TTLs are shorter because the host servers can change.

`$ dig "@1.1.1.1" depaul.edu MX +noall +answer`

Mail delivery uses a hostname instead of a raw IP address because `A` records already handle this. `MX` records allow mail routing to adapt to any changes in the `A` record.

`$ dig "@1.1.1.1" depaul.edu TXT +noall +answer`

I am able to recognize the third party verifications for Zoom, autodesk, google, sitecore, apple, atlassian, etc. Additionally, these all have long TTLs, allowing for fewer future queries. 

`$ dig "@1.1.1.1" github.com CAA +noall +answer`

`github.com` allows standard certificates from `digicert.com`, `globalsign.com`, `letsencrypt.org`, and `sectigo.com`. It also allows wildcard certificates for `digicert.com`, `letsencrypt.org`, and `sectigo.com`. From my output, it looks like there are no restrictions listed, as they all have a `0` flag. 

`$ dig "@1.1.1.1" depaul.edu CAA`

`depaul.edu`'s `NODDATA` result (indicated by `ANSWER: 0` and `STATUS: NOERROR`) means that the domain name exists but no `CAA` records have been configured. There are no restrictions on certificate issuance. 

### Task 3 - Compare authoritative truthw ith recursive caches

The first query is authoritative, indicated by the `aa` flag that appears in the header. This was answered with a `CNAME` record where `www.depaul.edu` is mapped to `22b8cde06cfddd89.vercel-dns-013.com`.

The rest of the queries were recursive answers. In the second query, we are given the same `CNAME` record from the server but we are also given two `A` records `64.239.109.193` and `64.239.123.193` which are both mapped to the `CNAME` record. These are the two servers that Vercel is using to serve `www.depaul.edu`. The third query has the same `CNAME record` but provided different `A` records `64.239.123.129` and `64.239.109.129`. Finally, the fourth query has the same `CNAME` record and two different `A` records `64.239.109.1` and `64.239.123.1`.

There is only a `CNAME` record in the authoritative answer, so the RRset consists of this one record. However, in the second query, the host `22b8cde06cfddd89.vercel-dns-013.com` has two entries with an `IN A` record to two different IP addresses, making it an RRset.

**Apparent Cache Age Calculations:**

Authoritative TTL (`86,400`) - Resolver TTL `1.1.1.1` (`53,870`) = `32,530` apparent seconds since `1.1.1.1` cache last refreshed

Authoritative TTL (`86,400`) - Resolver TTL `8.8.8.8` (`21,600`) = `64,800` apparent seconds since `8.8.8.8` cache last refreshed

Authoritative TTL (`86,400`) - Resolver TTL `9.9.9.9` (`43,200`) = `43,200` apparent seconds since `9.9.9.9` cache last refreshed

### Task 4 - Cutover plan from public evidence 

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

1. When to lower the TTL and why the old TTL controls the waiting period.
2. The earliest safe cutover time.
3. What exact dig commands you would run against authority, 1.1.1.1, and 8.8.8.8.
4. What evidence would trigger rollback.
5. What evidence is not a rollback trigger because it is expected cache lag.
6. When it is safe to retire the old target.
7. Then add a two-sentence reflection connecting the plan to your Task 3 public DNS evidence.

...

### Task 5 - Diagnose supplied HTTP failures using VSCode `.http` requests

For each fault, four things:

1. The annotated request/response pair — the specific lines carrying the evidence,
   with your annotation on each.

2. The diagnosis — one or two sentences naming the actual cause.

3. The confirming command — the single command that distinguishes your diagnosis from
   the next most plausible one, and the result that would confirm it.
   
4. The fix — what you would change, and where: app, proxy, or DNS.

## Deliberate False Lead and Rejection

## Redaction Attestation

No live secrets, tokens, cookies, session identifiers, or API keys needed to be redacted.