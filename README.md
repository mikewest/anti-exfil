Connection Allowlists
=====================

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
Connection-Allowlist: (response-origin "https://*.site.example" "https://cdn.example" "https://*.site.([a-z\\-]+)")
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
    it (with the caveat that we'll likely inherit this policy into contexts
    created via local schemes (`data:`, `about:`, etc.), similar to other
    components of a context's
    [policy container](https://html.spec.whatwg.org/multipage/browsers.html#policy-containers))
    which are handled by HTML when creating new document/worker contexts.
    
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
    responses as match failures. That is, we'll start with a draconian policy
    which will block any redirect response. It's quite possible we'll shift
    this one way or another as use cases crystalize.


Questions
---------

### Why build this when we already have Content Security Policy? ###

Three reasons:

1.  CSP's model is too granular: Developers who wish to mitigate the risk that data flows out
    of a sensitive context require a protection that exhaustively covers the possible
    ways in which requests can be made or connections established. CSP's categorization
    of requests into types which can be controlled in isolation is the wrong way to
    approach this problem, as data leaking through a request for a web font is just as
    bad as data leaking through a request for an image or a script. Distinguishing these
    request types complicates the process of designing a reasonable defense with questions
    that are simply irrelevant.

2.  CSP's syntax is not granular enough: The `host-source` grammar CSP supports leads to truly
    verbose headers being delivered with responses. A distinct policy provides the opportunity
    to shift to the URLPattern syntax which will resolve some complaints folks have raised about
    CSP's approach by providing a more modern, malleable, and standardized matching syntax.

3.  CSP's coverage is incomplete: While CSP does a good job covering HTTP requests which run
    through Fetch, it does not exhaustively cover the myriad ways in which web platform APIs
    allow connections to be established. DNS prefetch and WebRTC are good examples to start
    with, but there are many others which have struggled with exactly how they fit into CSP's
    threat model. By creating a new policy with a narrow focus and explicit promise to developers,
    these discussions will have a defensible answer and a clear mandate to specification authors.


### How does this policy interact with other mechanisms of controlling outgoing requests? ###

In short, we should follow CSP's model for resolving multiple policies: each should take effect,
and connections will only be established if they pass all of the applicable policies. That is, a
response containing the following headers:

```http
Content-Security-Policy: default-src https://good.example/ https://sketchy.example/
Connection-Allowlist: ("https://good.example/")
```

requests to `https://good.example/` would pass both policies, while requests to
`https://sketchy.example/` would be blocked, as they wouldn't match the `Connection-Allowlist`.


### We probably need a report-only variant of this policy, don't we? ###

Of course. That's been a critical part of real-world policy deployments, and we'll need it here too.


### What about navigation? ###

Navigation (of the current context or any other to which a handle can be obtained) is clearly in-scope
for exfiltration mitigation. In certain contexts, it will be straightforward to list all the possible
places to which navigation is expected, and the current proposal will handle those situations well.
In other situations, endpoints might be harder to determine. OAuth flows, payment flows, etc. might hop
around in ways which are difficult to predict.

It's potentially necessary for us to split navigation requests out from the rest, possibly by allowing
developers to set broad expectations for them via a property on the list
(e.g. `(...);allow-navigation={same-origin,same-site,cross-origin}`)? Feedback on use cases here would
be quite helpful.


### You're going to have to rethink a ban on redirects. ###

Probably, yes. I want to simplify the initial proposal so we can start talking about it, but I do
anticipate redirects being required one way or the other for real-world deployments. I think we have
a few realistic options:

1.  Apply the allowlist to every hop of a redirect chain. This has the advantage of matching CSP's
    behavior that developers are already familiar with. It _is_ a cross-origin data leak insofar as it
    provides insight about another origin's decisions, which is unfortunate but perhaps unavoidable.

2.  Allow _a specific rule_'s redirect chain to arbitrarily redirect. This narrows the concerns
    above by forcing developers to annotate the allowlist with their expectations. It might be perfectly
    acceptable for `https://trusted.example/` to redirect users to arbitrary locations, while other
    endpoints are expected to remain put. Annotating list items should make this kind of distinction
    possible if necessary (e.g. `("https://trusted.example/";redirection-allowed "https://less-so.example/")`).

3.  Narrow the above by allowing _a specific rule_ to redirect so long as the targets match the allowlist.
    This creates less opportunity for unexpected connection than 1 or 2 by requiring developers to annotate
    the specific rules which can redirect, but would do so in a way that's less broad (e.g.
    `("https://semi-trusted.example/";redirection-allowed=within-allowlist ...)).

We could add more options as well. CSP's earlier `navigate-to` proposal distinguished between intermediate
redirects and the final, non-redirect response. You could imagine adding those kinds of options either to
the entire allowlist or individual rules. Feedback here as well would be much appreciated.
