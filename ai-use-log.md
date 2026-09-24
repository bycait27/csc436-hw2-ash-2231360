# AI Use Log

**Model:** Gemini

## Task 4

**What was generated:**

- Initial request for a cutover plan outline based on the prompt's scenario parameters (**TTL**: `86400`, temporary **TTL:** `300`, cutover at `Friday 20:00 UTC`).

- Draft phrasing for the safety window calculation.

**What was accepted:**

- The step-by-step structural breakdown of the cutover timeline (pre-lowering TTL, waiting period, cutover window, verification, and retirement).

**What was rejected:**

- The AI initially suggested lowering the TTL 1 hour before cutover (at `19:00 UTC`).

**How it was verified and corrected:**

- **Verification:** I cross-checked this against the assignment rubric and TTL caching mechanics. Lowering the TTL an hour before a 24-hour (`86400`) TTL expires is unsafe because recursive resolvers will still serve the old cached records for up to 24 hours.

- **Correction:** I rejected the AI's 1-hour window and manually rewrote the plan to specify lowering the TTL at least 24 hours in advance so that the old cache fully expires before the cutover window opens.

## Task 5

**What was generated:**

- Explanations and debugging hints for the local fault server logs and HTTP response codes.

- Initial syntax suggestions for executing requests via PowerShell vs. Bash.

- Suggestions that an infinite redirect loop (`301 Moved Permanently` or `302 Found` bouncing endlessly) was caused by a cyclical DNS `CNAME` delegation or a browser cache issue. 

- Explanations regarding SSL termination, reverse proxies, and header forwarding mechanisms.

- Code snippet suggestions for modifying Express.js routing middleware.

**What was accepted:**

- The diagnosis for Fault 1 (missing vhost/edge routing mapping, disproving the "DNS propagation" myth).

- The structure for writing out annotated request/response pairs.

- The step-by-step header inspection strategy using the `.http` response pane to trace the Location header across multiple hops.

- The identification that the backend application forces HTTPS redirection unconditionally, but lacks awareness of the upstream proxy.

**What was rejected:**

- The AI initially suggested that Fault 1's fix involved changing local hosts file entries or updating global DNS records.

- The AI's initial claim that the loop was originating from an external DNS record pointing to itself.

- The AI initially suggested fixing the proxy loop by installing local self-signed SSL certificates directly onto the backend application server.

**How it was verified and corrected:**

**Verification:** 

- I examined the actual response body from the local server ("This shared edge has no site configured for the requested hostname"). Since the request successfully hit `127.0.0.1:8081` and returned a `404` from the proxy layer, DNS was completely out of the equation.

- By examining the local response headers, the server itself was issuing the redirect (`Location: /` or pointing right back to the root path it just served) without ever leaving the local host.

- Testing the direct app route (`http://127.0.0.1:8083`) versus the proxy route (`http://127.0.0.1:8084`) showed that the proxy was terminating TLS and forwarding requests to the app over plain HTTP. Because the app wasn't receiving the `X-Forwarded-Proto:` https header, it assumed the client was unencrypted and issued a redirect to HTTPS, creating an infinite loop at the proxy layer.

**Correction:** 

- I corrected the fix to focus on edge server reverse-proxy configuration rather than DNS changes.

- I ruled out DNS immediately because the request never left `127.0.0.1:8082`. I corrected the diagnosis to focus on internal server routing rules (e.g., a misconfigured www enforcement rule or a root-to-index redirection clash).

- I rejected the certificate installation suggestion and instead verified that the proper fix required configuring the proxy layer to inject the standard forwarding headers (`X-Forwarded-Proto`, `X-Forwarded-For`) so the backend app could respect the original client protocol.

## Challenged AI Claim 

**Prompt:** Why are POST requests to the API endpoint in Fault 2B silently failing to create resources even though the route exists?

**Tool:** Gemini

**The claim:** The AI claimed that a standard `302 Found` redirect issued by the server preserves the original `POST` method and request body when an HTTP client automatically follows the redirect.

**The artifact that rejects the claim:** The [.http](./evidence/05-http-faults.txt) file response capture and server logs, cross-referenced with HTTP specification (RFC 7231 / RFC 9110). The logs and response artifact prove that a classic `302` (and `303 See Other`) forces compliant user agents to change the follow-up request method from `POST` to `GET`, causing the backend to receive a bodyless `GET` request instead of the expected `POST` payload.