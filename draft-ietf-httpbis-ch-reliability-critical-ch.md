---
title: Critical-CH for Client Hint Reliability
docname: draft-ietf-httpbis-ch-reliability-critical-ch-latest
submissiontype: IETF
category: exp
updates: ietf-httpbis-client-hints

ipr: trust200902
area: General
workgroup: HTTP

pi: [toc, sortrefs, symrefs]

author:
 -
       ins: V. Tan
       name: Victor Tan
       organization: Google LLC
       email: victortan@google.com

normative:
  RFC2119:
  RFC5234:
  RFC7231:
  RFC8941:
  RFC8942:
  RFC9000:
  RFC3864:

  I-D.davidben-http-client-hint-reliability:

informative:
  DEVICE-MEMORY:
    target: https://w3c.github.io/device-memory/
    title: Device Memory
    author:
      ins: S. Panicker
      name: Shubhie Panicker
      organization: Google LLC


--- abstract

This document defines the Critical-CH HTTP response header to allow HTTP servers
to reliably specify their Client Hint preferences.

--- middle

# Introduction

{{RFC8942}} defines a response header, Accept-CH, for servers to advertise a
set of request headers used for proactive content negotiation. This allows user
agents to send request headers only when used, improving their performance
overhead as well as reducing passive fingerprinting surface.

However, on the first HTTP request to a server, the user agent will not have
received the Accept-CH header and may not take the server preferences into
account. More generally, the server's configuration may have changed since the
most recent HTTP request to the server. This document addresses this issue by
introducing an HTTP response header, Critical-CH, for the server to instruct
the user agent to retry the request if client hints were not initially included
but would have been sent based on the received Accept-CH header.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document uses the Augmented Backus-Naur Form (ABNF) notation of {{RFC5234}}.

This document uses the variable-length integer encoding and frame diagram format
from {{RFC9000}}.


# The Critical-CH Response Header Field {#critical-ch}

When a user agent requests a resource based on a missing or outdated Accept-CH
value, it may not send a desired request header field. Neither user agent nor server
has enough information to reliably and efficiently recover from this situation.
The server can observe that the header is missing, but the user agent may not have
supported the header, or may have chosen not to send it. Triggering a new
request in these cases would risk an infinite loop or an unnecessary round-trip.

Conversely, the user agent can observe that a request header appears in the
Accept-CH ({{Section 3.1 of RFC8942}}) and Vary ({{Section 7.1.4 of RFC7231}})
response header fields. However, retrying based on this information would waste
resources if the resource only used the Client Hint as an optional
optimization.

This document introduces critical Client Hints. These are the Client Hints which
meaningfully change the resulting resource. For example, a server may use the
Device-Memory Client Hint {{DEVICE-MEMORY}} to select simple and complex variants
of a resource to different user agents. Such a resource should be fetched
consistently across page loads to avoid jarring user-visible switches.

The server specifies critical Client Hints with the Critical-CH response header
field. It is a Structured Header {{RFC8941}} whose value MUST be an sf-list
({{Section 3.1 of RFC8941}}) whose members are tokens ({{Section 3.3.4 of
RFC8941}}). Its ABNF is:

~~~
  Critical-CH = sf-list
~~~

For example:

~~~
  Critical-CH: Sec-CH-Example, Sec-CH-Example-2
~~~

Each token listed in the Critical-CH header SHOULD additionally be present in
the Accept-CH and Vary response headers.

When a user agent receives an HTTP response containing a Critical-CH header, it
first processes the Accept-CH header as described in {{Section 3.1 of
RFC8942}}. It then performs the following steps:

1. If the request did not use a safe method ({{Section 4.2.1 of RFC7231}}), ignore the Critical-CH header and continue processing the response as usual.

2. If the response was already the result of a retry, ignore the Critical-CH header and continue processing the response as usual.

3. Determine the Client Hints that would have been sent given the updated Accept-CH value, incorporating the user agent's local policy and user preferences. See also {{Section 2.1 of RFC8942}}.

4. Compare this result to the Client Hints which were sent. If any Client Hint listed in the Critical-CH header was not previously sent and would now have been sent, retry the request with the new preferences. Otherwise, continue processing the response as usual.

Note this procedure does not cause the user agent to send Client Hints it would
not otherwise send.

## Example

For example, if the user agent loads https://example.com with no knowledge of
the server's Accept-CH preferences, it may send the following response:

~~~
  GET / HTTP/1.1
  Host: example.com

  HTTP/1.1 200 OK
  Content-Type: text/html
  Accept-CH: Sec-CH-Example, Sec-CH-Example-2
  Vary: Sec-CH-Example
  Critical-CH: Sec-CH-Example
~~~

In this example, the server, across the whole origin, uses both Sec-CH-Example
and Sec-CH-Example-2 Client Hints. However, this resource only uses
Sec-CH-Example, which it considers critical.

The user agent now processes the Accept-CH header and determines it would have
sent both headers. Sec-CH-Example is listed in Critical-CH, so the user agent
retries the request, and receives a more specific response.

~~~
  GET / HTTP/1.1
  Host: example.com
  Sec-CH-Example: 1
  Sec-CH-Example-2: 2

  HTTP/1.1 200 OK
  Content-Type: text/html
  Accept-CH: Sec-CH-Example, Sec-CH-Example-2
  Vary: Sec-CH-Example
  Critical-CH: Sec-CH-Example
~~~


# Security Considerations

Request header fields may expose sensitive information about the user's
environment. {{Section 4.1 of RFC8942}} discusses some of these considerations.
The document augments the capabilities of Client Hints, but does not change
these considerations. The procedure described in {{critical-ch}} does not
result in the user agent sending request headers it otherwise would not have.

# IANA Considerations

Features relying on this document are expected to register added request header
fields in the Permanent Message Header Fields registry ({{RFC3864}}).

This document defines the "Critical-CH" HTTP response header field, and
registers it in the same registry.

## Critical-CH Header Field

Header field name:
: Critical-CH

Applicable protocol:
: HTTP

Status:
: Standard

Author/Change controller:
: IETF

Specification document:
: this specification ({{critical-ch}})

Related information:
: for Client Hints

--- back

# Acknowledgments
{:numbered="false"}

This document has benefited from the work of David Benjamin in {{I-D.davidben-http-client-hint-reliability}},
contributions and suggestions from Ilya Grigorik, Nick Harper, Matt Menke, Aaron Tagliaboschi,
Victor Vasiliev, Yoav Weiss, and others.
