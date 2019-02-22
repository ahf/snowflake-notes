# Snowflake Broker

This document tries to describe the current Snowflake broker design as of
February 2018. The goal with this document is to make it easier for new
contributors to navigate the source code quickly, but also to help people who
are interested in design decisions to participate without having to have a deep
understanding of the current implementation of the broker.

The Broker is written in Google's Go programming language and the source code
can be found at https://gitweb.torproject.org/pluggable-transports/snowflake.git/tree/broker

## Introduction

The Snowflake Broker is responsible for handing out proxies to clients that
request them. The client connects using "domain fronted HTTP" and
exchanges WebRTC Session Descriptor documents. The WebRTC Session Descriptor
documents are small text documents describing how two or more WebRTC peers can
reach each other. The Snowflake client will then use the Session Descriptor to
connect to the Snowflake proxy which will then connect to the Snowflake Bridge
via WebSocket and relay traffic between the Bridge and the Client.

The process of negotiation and "finding each other" is often described as
signaling in WebRTC documentation.

You can see some example WebRTC Session Descriptor offers/answers in
[RFC4317](https://www.rfc-editor.org/rfc/rfc4317.txt). A useful website for
getting a better understanding of the lines found in the Session Descriptor
document can be found at https://webrtchacks.com/sdp-anatomy/

## Go Dependencies

The broker only has one external dependency and does not depend on any PT
related libraries such as `goptlib` since it's a standalone application and is
not executed via, for example, Tor like PT client and bridges are.

The current external dependency of the Snowflake Broker is:

- `autocert` (https://godoc.org/golang.org/x/crypto/acme/autocert) - autocert
  is an implementation of the ACME protocol used to get Let's Encrypt validated
  x509 TLS certificates. The broker uses this to generate a valid TLS certificate
  using the hostname of the broker instance as well as an email address of the
  administrator.

## Implementation Details

This section tries to outline some implementation details of the Snowflake
Broker codebase.

### The HTTP interface

When the Broker is started it exposes an HTTP API available on port 443 via
HTTPS. The Broker is responsible for fetching a TLS certificate from Let's
Encrypt using the hostname and email address given on the command line
parameters. You can optionally start the Broker where it listens on plain text
HTTP (port 80) if one is interested in running the Broker behind another
webserver such as nginx or Apache's httpd.

The broker exposes the following handlers:

#### The `/debug` handler

Dumps some debug information about the current state of the Broker. This
includes the number of currently available Snowflakes (proxies), their ID's,
and the round-trip average (see the metrics section of this document).

Example response:

    current snowflakes available: 2

    snowflake 1: AAAAAAAAAAAAAAAAAAAAAA
    snowflake 0: BBBBBBBBBBBBBBBBBBBBBB

    roundtrip avg: 123

#### The `/proxy` handler

The `/proxy` handler is used by Snowflake proxies to "poll" for a Snowflake
client. 

If there is a client available the Proxy receives the WebRTC Session Descriptor
Offer from the client, which is used to help the proxy to establish
connectivity between the client and the proxy. Snowflake clients uses the
`/client` HTTP handler to submit their Session Descriptor Offer.

If no clients becomes available to the Broker while the proxy is waiting a
timeout will be reached and the broker responds with a `504 Gateway Timeout`
HTTP error code. The proxy can reconnect again until a Snowflake client is
available.

#### The `/client` handler

The `/client` handler is used by Snowflake clients to send their WebRTC Session
Descriptor Offer in the hope of receiving a Session Descriptor Answer from the
proxy (the proxy calls the `/answer` HTTP handler). The available Snowflake
proxies (see the description of the Snowflake Heap) is the set of proxies that
are currently polling the `/proxy` handler.

If the proxy responds (to the `/answer` handler) with a WebRTC Session
Descriptor Answer the content is passed to the client allowing the client and
proxy to establish connectivity between each other using WebRTC.

If the Snowflake proxy does not respond a timeout is reached and the client
will receive a `504 Gateway Timeout` HTTP error code.

#### The `/answer` handler

The `/answer` handler is used by Snowflake proxies that have already received a
WebRTC Session Descriptor Offer from a Snowflake client to respond with its
WebRTC Session Descriptor Answer. The broker will then pass the answer to the
client that is currently polling for the answer in the `/client` handler.

If the client was too slow to send its offer the broker will respond with a
`410 Gone` HTTP error code. If the proxy responds with an empty or otherwise
bogus answer the `400 Bad Request` HTTP error code is returned.

#### The `/robots.txt` handler

This handler is used to serve a `robots.txt` file with the following content:

    User-agent: *
    Disallow:

Which allow web crawlers to crawl everything on the site. For more information
about this please have a look at https://moz.com/learn/seo/robotstxt

FIXME(ahf): Is this really correct? Shouldn't we disallow crawling?

### The Heap

The Snowflake Heap (`type SnowflakeHeap`) is an implementation of Go's
[container/heap](https://golang.org/pkg/container/heap/) data structure and is
used as a priority queue of Snowflake proxies that are available for acting as
a proxy for clients. The method `Pop()` is used to take the "most available"
Snowflake Proxy and pass the Session Descriptor Offer to it.

### Metrics

The Snowflake broker has an interface for collecting some internal metrics.
Currently only the `clientRoundtripEstimate` metric is implemented.

- `clientRoundtripEstimate`: stores the round trip time between the Session
  Descriptor Offer is given to the client and an answer is received.
