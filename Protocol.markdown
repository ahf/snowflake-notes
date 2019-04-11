Snowflake Protocol
==================

At the Brussels meeting we were discussing whether the current very simple
protocol between the various Snowflake components should be extended to at
least be extensible for future protocol changes, but also should allow us to
collect various metrics.

## Client to Snowflake Proxy protocol

Notes:

1. Should a client be able to connect to multiple proxies? Would it make any
   sense with Tor's current PT design where we have one output stream from the
   Tor process?
2. Should the client be able to specify which bridge (from a closed set) the
   proxy should use?
3. Should there be some kind of heartbeat protocol to find out whether the connection is still alive?

Related tickets:
- https://trac.torproject.org/projects/tor/ticket/25429

Relevant code:
- https://gitweb.torproject.org/pluggable-transports/snowflake.git/tree/proxy-go/snowflake.go#n261

## The layer between the Client to Proxy and Proxy to Broker

1. Does the "token bucket" algorithm in Snowflake right now do what we think?

## Snowflake Proxy to Broker protocol

Notes:

1. It would be useful for us to have information about which countries clients
   are connecting from. If this information could be relayed to the Broker we
   would be able to detect if there is a sudden drop in traffic from a specific
   country. (Medium priority?)
2. Tor relays have an identity key, which allows us to reason about their
   performance and stability. Should we (optionally?) allow Snowflake proxies
   to have an identity key such that we later can start doing things like having
   flags like with Tor relays where some Snowflake proxies are considered more
   stable than others? (Low priority?)
3. Should the broker be able to handle connections from multiple proxies for
   load balancing/stability? (??? priority)
4. For extensibility: allow snowflakes to identify which transport (if we later
   decide to use something other than WebRTC) they are using, and which version
   of the transport
5. For extensibility: related to (4), make sure proxy and broker can
   communicate necessary connection information for transport types other than
   WebRTC
6. Should we switch up how proxies poll or "register" with the broker? Maybe
   WebSocket?
7. Maybe have proxy advertize a capacity value? Either for bandwidth or number
   of connections?

Related tickets:
- https://trac.torproject.org/projects/tor/ticket/25598 (feature to let broker inform proxies how often to poll, could depend on number of snowflakes and number of clients)
- https://trac.torproject.org/projects/tor/ticket/25681 (detect and prevent floods of low-bandwidth snowflakes)
- https://trac.torproject.org/projects/tor/ticket/29260 (identify snowflakes to broker for reputation-based system)
- https://trac.torproject.org/projects/tor/ticket/21315 (stats about snowflake usage)

Relevant code:
- https://gitweb.torproject.org/pluggable-transports/snowflake.git/tree/proxy-go/snowflake.go#n135
- https://gitweb.torproject.org/pluggable-transports/snowflake.git/tree/proxy-go/snowflake.go#n171

## Client to Broker protocol

This step is used to discover the WebRTC ID of a Snowflake Proxy. All actual
communication happens via the Snowflake Proxy which relays traffic to the
Broker.

Notes:
1. Should be kept at a very bare minimum since this protocol is "domain fronted".
2. Collect stats about where people are trying to connect from to get a proxy
   ID. This would allow us to check if more people are requesting proxy ID's
   than we serve via the proxies, which would allow us to detect if a country
   allows the domain fronting technique, but rejects (for example) WebRTC.
3. Extensibility: allow clients to request different transport types or version numbers

Related tickets:
- https://trac.torproject.org/projects/tor/ticket/21305 (bug)
- https://trac.torproject.org/projects/tor/ticket/21315 (stats about client usage)

Relevant code:
- https://gitweb.torproject.org/pluggable-transports/snowflake.git/tree/client/snowflake.go#n112

## Broker to Broker protocol

1. Let's ignore this one for now, but if we want to have a highly available
   distributed system we should probably support some kind of gossip protocol
   between two brokers so they can share state about available proxies (and their
   load).

## Useful metrics to know

1. Which countries does people connect to Proxies from? GeoIP style like with Tor?
2. Which countries does people request proxies from? GeoIP style like with Tor?
3. How much BW do we see at the bridges? and the snowflakes?
4. How many times does a Snowflake proxy fail to connect to a bridge?
5. Where do people run Snowflake proxies from? How reliable/persistant are these proxies?
