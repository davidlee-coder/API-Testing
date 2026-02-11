# Finding and exploiting an unused API endpoint
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security Research](https://img.shields.io/badge/Security-Research-blue.svg)](https://github.com/yourusername/web-shell-race-condition)

**Level** — Practitioner<p align="center"<p align="center"></i></p>
<br>
**Category** — API Testing / Business Logic Vulnerabilities<p align="center"<p align="center"></i></p>
<br>
**PortSwigger Link** — https://portswigger.net/web-security/api-testing/lab-exploiting-unused-api-endpoint<p align="center"<p align="center"></i></p>
<br>
**Completed** — February 11 2026<p align="center"<p align="center"></i></p>
<br>
**Tools** — Burp Suite Proxy & Repeater, normal browser interaction<p align="center"<p align="center"></i></p>
<br><br>

# Table of Contents

- [Overview](#overview)
- [My Aha Moment](#my-aha-moment)
- [Exploitation](#exploitation)
- [Root Cause](#root-cause)
- [Impact](#impact)
- [Mitigation](#mitigation)

# Overview: Finding and exploiting an unused API endpoint

This vulnerability exploits a common architectural oversight: Broken Function Level Authorization (BFLA). While the e-commerce front-end is designed to only perform read-only GET requests for price displays, the underlying API improperly exposes high-privilege modification methods (POST/PUT/PATCH) to standard authenticated users.
The Hidden Attack Surface:
The flaw isn't a lack of authentication—the server correctly returns a 401 Unauthorized for anonymous users. Instead, the failure is in the Authorization logic. Because the UI never triggers these 'hidden' methods, they often go untested and unhardened.
By manually intercepting a standard request and pivoting the HTTP method, an attacker can bypass all client-side logic to inject arbitrary prices or discounts directly into the database. This turns a standard user session into a functional 'Admin' session for price manipulation.


# Aha Moment

The breakthrough came when I realized the endpoint wasn't truly 'hidden'—it was a core part of the shopping experience (item details, cart updates), but it was only ever invoked via GET requests. Because the front-end never used POST or PUT, the dangerous modification logic remained invisible during standard use.
I initially mistook the 401 Unauthorized response on anonymous requests as proof of a 'secure' endpoint. However, once authenticated, the server's guard dropped completely. It blindly accepted HTTP Method Alteration, allowing me to inject a new price directly into the application state without further authorization checks.
Seeing the price drop to $0.01 in the cart after a simple method swap was a stark reminder of how fragile business logic becomes when the API and UI are not tightly coupled. This experience shifted my mindset: I no longer assume a 'read-only' endpoint is safe. Now, every authenticated GET request I encounter is a candidate for Method Brute-Forcing to uncover hidden administrative or modification capabilities..

# Exploitation 

I started by just browsing the site like any other customer. Every time I clicked a product priced at $1330, Burp caught a standard GET request to /api/products/1 with the price in the JSON response. At first, I tried a blind PATCH and PUT in Repeater, but I got a 401 Unauthorized. It felt like a dead end.
<img width="1356" height="692" alt="image" src="https://github.com/user-attachments/assets/31627265-be1b-498c-b0f0-034572266a63" />
<img width="1361" height="684" alt="image" src="https://github.com/user-attachments/assets/95cfa9d6-6f5b-4a72-b0ff-38c9ad785c8a" />
<img width="1337" height="677" alt="image" src="https://github.com/user-attachments/assets/70aee258-9564-4678-b424-7d82fb6e81bb" />
<img width="1329" height="680" alt="image" src="https://github.com/user-attachments/assets/ba110d96-a446-4688-84be-a4909bfd8539" />
<img width="1021" height="643" alt="image" src="https://github.com/user-attachments/assets/8c720048-1db7-432d-a64e-81ec72b01c3b" />
<img width="1024" height="673" alt="image" src="https://github.com/user-attachments/assets/f803d905-4f46-4592-839f-53cf538383d4" />
<p align="center"></i></p>
<br><br>

I realized I was testing as an anonymous guest. I logged in, added an item to my cart, and saw that same endpoint pop up again—still a GET, but now I had an authentication session. I grabbed that authenticated request, flipped the method to PATCH, and started a 'conversation' with the server to see what it would let me get away with:
<img width="1296" height="656" alt="image" src="https://github.com/user-attachments/assets/f26e4f31-3ef7-46ff-8416-f211ee089fef" />
<img width="1336" height="637" alt="image" src="https://github.com/user-attachments/assets/5b60dc12-f257-4f55-9b34-44f63c2f42af" />
<p align="center"></i></p>
<br><br>

The server barked back with a 400 error, basically saying, "I only talk in application/json." Fine. I added the header.
<img width="1024" height="678" alt="image" src="https://github.com/user-attachments/assets/a5323a1a-d4f1-4bbf-99de-646a530b97fe" />
<p align="center"></i></p>
<br><br>

I sent a PATCH with an empty body and hit a 500 error. Usually, a 500 is a bad sign, but here it was a win—it meant the server wasn't blocking my request anymore; it was actually trying to process it and crashing because it didn't know what to do with the empty input. I sent a pair of empty brackets {} to see if I could trigger a better error message. It worked! The API leaked exactly what it wanted "Missing price parameter."
<img width="1024" height="678" alt="image" src="https://github.com/user-attachments/assets/40541c45-94a2-486d-8bc3-681eb0cbfd4e" />
<img width="1028" height="669" alt="image" src="https://github.com/user-attachments/assets/ff2cb1c7-e510-4f24-8937-d5bf81c1897c" />
<p align="center"></i></p>
<br><br>

I had the key. I sent {"price": 0} in the body and held my breath and the response returned 200 OK. "price": $0.00"
<img width="1027" height="641" alt="image" src="https://github.com/user-attachments/assets/863df472-2a5f-461e-bfe9-0eb84aa5ed48" />
<p align="center"></i></p>
<br><br>

I hopped back over to my browser, refreshed the cart, and there it was the item was now $0.00. I checked out for basically free. It was one of those moments where you just sit back and realize: the vulnerability wasn't hidden behind some complex exploit; it was sitting right there in plain sight, just waiting for someone to change the HTTP method.
<img width="1356" height="689" alt="image" src="https://github.com/user-attachments/assets/0c1e6087-0c29-476e-9111-ad3593b380e0" />
<img width="1307" height="651" alt="image" src="https://github.com/user-attachments/assets/f143a904-9123-4bf9-a286-f149cebf9eef" />
<p align="center"></i></p>
<br><br>

# Root Cause

- API endpoint accepts dangerous methods (POST/PUT/PATCH) for price updates when authenticated, without additional authorization checks  
- Front-end UI only ever sends GET requests → no client-side trigger for modification logic  
- No server-side validation that the method matches expected use-case (read vs write)  
- Lack of method restriction or role-based access control on price-modifying actions  
- No integrity check on total price during checkout (server trusts the manipulated price)

# Impact

- Financial Loss & Profit Abuse: Attackers can manipulate prices to $0.01 or even negative values. In some cases, this allows for "profitable" purchases where an attacker exploits refund policies to receive more money back than they originally spent.

- Inventory & Resource Drainage: By lowering prices to near-zero, an attacker can purchase high-value inventory in bulk. This leads to stockouts for legitimate customers and massive physical asset loss for the company.

- Vulnerability Chaining: This flaw rarely exists in a vacuum. It can be paired with IDOR (Insecure Direct Object Reference) to change prices in other users' carts or Race Conditions to bypass stock limits during a mass exploitation event.

- Business Logic Collapse: Beyond just "free stuff," this represents a total failure of the Trust Boundary between the API and the storefront, rendering standard promotional and pricing controls useless.

# Mitigations

- Lock Down the Methods: If an endpoint is meant for browsing, it should only respond to GET. Use route handlers to explicitly reject POST, PUT, or PATCH with a 405 Method Not Allowed response.

- Recalculate at Checkout: This is the big one. Never trust a price passed in a request body. The server should ignore any client-supplied totals and recalculate the cart by looking up the SKU in the Product Database the second the 'Buy' button is hit.

- Role-Based Access (RBAC): Just because a user is 'logged in' doesn't mean they can change prices. Restrict any price-modifying actions to an Admin-only role. If a standard user tries to PATCH a price, they should hit a 403 Forbidden.

- Clean Up the 'Dark' API: If a write method (like POST) isn't needed for the public-facing site, remove it. Unused code is just an unmonitored attack surface waiting to be found.

- Audit Your Errors: My exploit worked because the API was too helpful with its error messages (e.g., "Missing price parameter"). Use Generic Error Responses in production to avoid leaking your internal API schema to an attacker.

Happy (ethical) Hacking!
