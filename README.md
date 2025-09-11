Request Policy
==============

Developers wish to have control over the resources loaded into their pages'
contexts and the endpoints to which their pages can make requests. This control
is necessary for several purposes, including limiting the ways in which users'
data can flow through the user agent (mitigating exfiltration attacks) and
ensuring control over a site's architecture and depedencies.

Content Security Policy addresses some of this need, but does so in a way that
is more granular than is necessary for the most critical use cases, and with a
syntax and grammar that's complicated by the other protections CSP is used to
deploy.

If we focus on the single use case of controlling the explicit requests a page
may initiate through Fetch and other web platform APIs (WebRTC, Web Transport,
FedCM, Web Payments, DNS Prefetch, etc), we can likely provide a tool whose
purpose is clear and that's straightforward to use.

Proposal
--------

In short, a server will distribute a list of acceptable endpoints as an HTTP
response header, along with some configuration metadata. Before establishing
a connection on behalf of a page, the user agent will consult this list of
endpoints, and block the connection before it's made if there's a mismatch.

One syntax which seems sufficient would be a Structured Field list of Strings,
each matching the `URLPattern` syntax for absolute URLs, or the token
"response-origin". This list could be annotated with a small number of
parameters as we discover required extension points, though it doesn't seem
that we need any to start with. For example:

```http
Request-Policy: (response-origin "https://*.site.example" "https://cdn.example" "https://*.site.([a-z]+)")
```

This policy would allow requests and connections to the origin of the server
which delivered the response, the specific CDN listed, any subdomain of
`site.example`, and any subdomain of any host whose penultimate DNS label is
`site`. URLPattern allows quite a bit of flexibility that we can reuse
here rather than inventing a new approach (or reusing a more limited grammar
from something like CSP).

To effectuate blocking connections, we'd add a check against this list to
Fetch, and to the more esoteric specifications partially listed above.

That's it.

Threat Model
------------

This proposal is intentionally small, targeting a specific but useful niche of
client-side attacks and/or misconfigurations:

*   Policies for documents and workers will be asserted by servers as HTTP
    response headers. This means that attackers who can manipulate response
    headers will remain out of scope.

*   A document's (or worker's) asserted policy governs only requests initiated
    by _that_ context. If a framed document asserts a distinct policy, so be
    it.

*   The connection and/or request are the threat the proposal aims to defend
    against. To be effective as an exfiltration defense, we must block
    connections before they're made.

*   There are a plethora of side-channels available as unintended effects of
    otherwise-excellent web platform APIs. This proposal does not attempt to
    address them, focusing instead soley on those requests or connections
    explicitly initiated by the user agent on a page's behalf. This certainly
    includes the clear cases of Fetch and XMLHttpRequest, along with resource
    requests generally. It also includes network connections established
    through channels that are less explicitly "requests": manifests fetched
    through the Web Install API, connections to TURN/STUN servers via WebRTC,
    Web Transport channels, navigations, and so on. These are all in scope,
    while more esoteric channels like memory or CPU consumption, socket
    exhaustion, and XSLeaks in general are not.

*   This proposal addresses only communication channels. It does not aim to
    prevent (or even substantially mitigate) threats like content injection
    or cross-site scripting. It only can constrain the impact of such an
    attack _on those specific pages_ where the policy is in place, and should
    be considered only as one layer in a page's defenses; it won't be
    sufficient in itself.

*   It's tempting to attempt to defend against a subset of server-side threats,
    like open redirects. It's possible that we could do so in a proposal like
    this one, similar conceptually to what CSP does today. That said, CSP's
    behavior around redirects is both a leak in itself, and has garnered
    inconsistently favorable feedback from developers and researchers alike.
    For simplicity's sake, this proposal will generally treat redirect
    responses as match failures. It's quite possible we'll shift this one way
    or another as use cases crystalize.
