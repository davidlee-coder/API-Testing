# Finding a Hidden GraphQL Endpoint with Introspection Defenses
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

**Level** — Practitioner<p align="center"></i></p>
<br>

**Category** — GraphQL API Testing / Reconnaissance<p align="center"></i></p>
<br>

**PortSwigger Link** — https://portswigger.net/web-security/graphql/lab-graphql-find-the-endpoint<p align="center"></i></p>
<br>

**Completed** — February 12 2026<p align="center"></i></p>
<br>

**Tools** — Burp Suite Proxy & Repeater, ffuf (path fuzzing), common GraphQL wordlist<p align="center"></i></p>
<br><br>

# Table of Contents

- [Overview](#overview)
- [My Aha Moment](#vulnerability-details)
- [Exploitation](#exploitation)
- [Root Cause](#root-cause)
- [Impact](#impact)
- [Mitigation](#mitigation)
  
# Overview

In production, many organizations disable GraphQL Introspection the feature that lets you query __schema to see the entire API map. The idea is that if an attacker can't see the "blueprint," they can't find the "rooms." However, the endpoint itself is rarely hidden; it’s usually just tucked away at a non-obvious path like /api, /gql, or /v1/graphql. 
The Core Vulnerability:
The danger isn't just about finding the URL; it’s about what the server does once you’re there. Even with introspection blocked, a GraphQL server is often a "leaky" talker. Universal Queries: Sending a simple query{__typename} can confirm an endpoint is GraphQL, even if it returns a 404 or 401 on standard GETs. The "Suggestions" Trap: Many servers (like Apollo) have a "did you mean?" feature enabled by default. If I guess a field like user, and it's actually getUser, the server will helpfully correct me, allowing me to manually reconstruct the schema.
Authorization Gaps: Once I’ve guessed a mutation name like deleteUser, the server might not check if my role is allowed to trigger it, leading to Broken Function Level Authorization (BFLA).

# Exploitation 

I started by proxying all my traffic while using the site normally logging in, browsing products, and checking out. To find the GraphQL entry point, I ran ffuf with a specialized wordlist of common GraphQL suffixes. Most paths hit a dead end, but /api stood out.
<img width="1364" height="680" alt="image" src="https://github.com/user-attachments/assets/d200e516-3e61-47b5-9a38-8d3213ef07d9" /><img width="1360" height="701" alt="image" src="https://github.com/user-attachments/assets/71721142-b1b4-4ec7-8817-ca2753958343" /
><img width="872" height="507" alt="image" src="https://github.com/user-attachments/assets/fa6ce4b5-f170-4eb2-b75f-7cd987c3d105" />


When I sent a GET request to /api, the server barked back with a "Query not present" error. That’s a massive tell—it’s basically the server admitting it’s waiting for a GraphQL query. I confirmed it by sending a Universal Query: GET /api?query=query{__typename}. The response {"data": {"__typename": "query"}} was my green light. I’d found the front door.
<img width="1022" height="610" alt="image" src="https://github.com/user-attachments/assets/dd4833d8-142b-46eb-85ac-d55e2922d205" />
<img width="1021" height="608" alt="image" src="https://github.com/user-attachments/assets/ca1a34d6-d636-4a41-aade-47bdb6397e8e" />

I fired up the Burp GraphQL extension to pull the full schema, but the server hit me with a 'Defense-in-Depth' error: 'GraphQL introspection not allowed, contains __schema or type'. The developers had clearly tried to block mapping tools.
<img width="1028" height="688" alt="image" src="https://github.com/user-attachments/assets/6bb26208-a56b-4f72-96da-4bd220bb70aa" />

I decided to try a classic obfuscation bypass. I injected a newline character (%0a) immediately after the __schema keyword in my query. That tiny bit of whitespace was enough to trip up the server’s regex filter. The floodgates opened, and the full schema poured into my Site Map including a very dangerous-looking mutation called deleteOrganizationUser. Looking through the leaked schema, I found two critical pieces of the puzzle:

    A Query: getUser(id: $id) which leaked usernames based on an integer ID.
    A Mutation: deleteOrganizationUser(input: $input) which took a user ID as a parameter.
<img width="1027" height="690" alt="image" src="https://github.com/user-attachments/assets/b7c212db-d741-49b4-a955-77e81a21401e" />
<img width="1025" height="686" alt="image" src="https://github.com/user-attachments/assets/a117d9f1-0651-4025-afa3-0daa3c6e3907" />
<img width="1365" height="704" alt="image" src="https://github.com/user-attachments/assets/feccfd13-e735-4724-9329-b153cfd4254f" />
<img width="1361" height="499" alt="image" src="https://github.com/user-attachments/assets/c5326b75-a20a-46d6-93b9-413306d9dade" />

I used the getUser query to hunt for my target, 'Carlos'. By incrementing the ID variable, I quickly found that Carlos was User ID 3.
<img width="1366" height="653" alt="image" src="https://github.com/user-attachments/assets/d35c83fa-dd77-4c0c-9f6d-9ab23dc21cae" />
<img width="1365" height="706" alt="image" src="https://github.com/user-attachments/assets/ddccd83f-d37d-4ac7-97b4-bb1a22ca5255" />

Now for the final move. I pivoted to the deleteOrganizationUser mutation in Burp Repeater, set the input ID to 3, and hit send. The server didn’t check if my user had the right to delete anyone else—it just executed the command. This wasn't just a GraphQL misconfiguration; it was a full Insecure Direct Object Reference (IDOR). Carlos was gone, and I’d proven that 'hidden' doesn't mean 'secure'.
<img width="1189" height="566" alt="image" src="https://github.com/user-attachments/assets/746b007b-8c8d-44cd-b794-aebc0130f8bf" />
<img width="1180" height="547" alt="image" src="https://github.com/user-attachments/assets/ff6ac971-04ee-41fa-8be0-37a7ca4c95c3" />
<img width="1365" height="685" alt="image" src="https://github.com/user-attachments/assets/7a2e43eb-3bc1-4952-8cfc-f84e4e407514" />

# Aha Moment

The real breakthrough didn't come from a fancy tool; it came from a failed checkout request. When /api/v2 spit back a structured GraphQL-style error instead of a generic 400, it was like the server accidentally whispered its true identity. Even with Introspection blocked, that tiny leak told me exactly what I was dealing with.I’d been assuming I’d need heavy brute-forcing or deep JavaScript Analysis to find a hidden entry point. But the truth was much simpler: the app was already talking to the GraphQL API in plain sight during normal use; it just wasn't advertising it.The moment that deleteUser mutation actually worked—without me ever having seen the official schema, it hit me, disabled Introspection is a speed bump, not a wall. GraphQL’s greatest strength is its flexibility, but that same flexibility makes it incredibly 'leaky.' If you can guess the query, the server will often just hand you the keys. It was a stark reminder that 'security through obscurity' is just an invitation for a curious attacker to start guessing.

# Root Cause

- GraphQL endpoint exposed at a non-obvious but predictable path (`/api/v2`)  
- Introspection disabled (good practice), but no additional protection (auth, rate limiting, allow-list of operations)  
- Dangerous mutations (`deleteUser`, role changes, etc.) accessible to authenticated users without fine-grained authorization  
- Lack of operation/method allow-listing or depth limiting  
- Helpful error messages or response shapes leaking GraphQL nature

# Impact

- **Arbitrary user deletion** — remove any account (including admin) via guessed mutation  
- **Privilege escalation** — change roles, reset passwords, or grant admin access  
- **Data manipulation** — create, update, or delete records via undocumented mutations  
- **DoS potential** — complex guessed queries could cause heavy backend load (if no query cost limiting)  
- **Schema reconstruction** — chain error messages, leaked types, and common naming conventions to rebuild schema over time  
In real GraphQL apps (especially with legacy or merged services), this can lead to mass account compromise or full data destruction.

# Mitigations

- Enforce Query Whitelisting: In production, only allow a pre-approved list of queries and mutations. Anything else should be dropped immediately.
- Disable Field Suggestions: Turn off the "did you mean?" functionality in production environments (e.g., set debug: false in Apollo) to prevent schema reconstruction.
- Generic Error Responses: Ensure error messages don't leak type names, field names, or stack traces. A simple "Invalid Request" is all an end-user needs.
- Strict Authorization: Never assume a mutation is safe just because it's 'hidden.' Every resolver must have Field-Level Authorization checks to verify the user's role before executing.


**Happy (ethical) Hacking!**
