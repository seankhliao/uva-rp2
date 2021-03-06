# title

## abstract

## intro

SOCKS is a widely supported proxy protocol with a client-server model,
providing a circuit-level gateway with a simple interface useful for traversing
firewalls and Network Address Translation (NAT) \cite{circuit}.
Despite the capitalization, SOCKS does not appear to be short for anything.
SOCKS5 was an update that added support for UDP among other features
to the TCP based protocol \cite{socks}.

Traversal Using Relays around NAT (TURN) is also a proxy protocol,
extending Session Traversal Utilities for NAT (STUN) \cite{stun}.
While STUN provides utilities for clients
to establish peer to peer connections through NAT and firewalls
with techniques such as hole punching,
this is not always successful,
in which case TURN relays with public addresses
can be used to relay data between the clients.
Given its primary usecase in audio/video communications
such as in WebRTC \cite{webrtc}, TURN uses UDP for peer connections.
RFC 6062 \cite{turntcp} specifies an extension to TURN to use TCP connections.

Given the nature of proxies,
operators of TURN relays need to be careful in the design of their network
and in the security policies enforced by the proxy itself.
Failure to do so could result in unwanted connections being made into internal networks.

For users of other networks, TURN relays run by public entities,
such as those used by videoconferencing software,
stand in a privileged position as connections to them
are often allowed to pass through both NAT and firewall due to business needs.
Using these TURN relays as generic proxies could punch through firewalls
for a wider class of applications.

## related work

From early in its design \cite{turn0}
TURN was recognized to stand at a critical juncture between networks.
The latest RFC \cite{turn} expands on the security considerations
when running TURN relays.
Additionally, both an authentication and permissions system is built into the protocol,
as well as recommendations in configuration.

CloudProxy was a research project that combined SOCKS and TURN
for vulnerability scanning with a focus on perfomance \cite{cloudproxy}.
It uses the TURN protocol for NAT traversal,
and a modified client-server pair to deduplicate scanning traffic.
This differs from the approach in this paper,
which uses unmodified and potentially third party TURN relays for NAT traversal.

The only other notable publicised instance of using TURN relays as a proxy
is an April 2020 report by Enable Security
outlining misconfiguration of Slack's TURN relays.
They used an internal tool
to proxy both TCP and UDP connections through Slack's TURN relays \cite{slack}.

## research question

The aim of this research is to implement and test ideas
for using third party TURN relays as proxies,
exposing the functionality as a SOCKS proxy.
There are two main appriaches we will take:
a forward translation layer and a reverse connection through a TURN relay.

The forwarding approach translates a SOCKS connection request
into a TURN connection request,
with the TURN relay making the final connection to the destination.
This acts mainly as a test of TURN relay capabilities and (mis)configuration.
This is very similar to what Enable Security have done,
with the main blocker being the lack of public libraries
that support the full range of TURN specifications
and capabilities implemented by relays.

The reverse connection will be of more interest to red teaming operations.
For practical reasons,
the TURN relays operated by major business software services providers
are often whitelisted in firewalls.
This presents the opportunity to pass connections through these relays,
masking their true destination and opening up protected networks to outside connections.
This is superficially similar to what CloudProxy did,
however, that project only used the protocol and not known endpoints to bypass NATs.

In short:

- Implement and test limitations of SOCKS to TURN translation layer.
- Implement and test limitations of tunnelling connections through TURN relays
  for purposes of bypassing network restrictions.
- Identify methods of preventing abuse for both network and TURN relay operators.

## background

### socks5

SOCKS is a simple protocol,
designed for when the client is behind some NAT or firewall and needs to jump through.

For TCP connections, clients can establish a connection to a proxy
and send request with the intended destination and options inline.
The proxy can then create the connection to the final destination
and reply with a success response to the client.
Any further data on the connections is then relayed between the client and destination
without modification.

For UDP, the client first establishes a TCP connection
to the proxy to act as a control connection.
The proxy replies with the UDP port for the client to connect to.
The TCP connection is then used to control the lifetime of the session.
Each UDP packet from the client contains a SOCKS header containing the destination,
this is removed by the proxy before forwarding to the destination.
Incoming packets have a SOCKS header inserted before being sent to the client.

### stun & turn

STUN and TURN are designed to facilitate establishing peer to peer connections.
STUN encompasses the base wire protocol
and interactions such as discovering the address of a client after NAT.
TURN extends the protocol to support relaying of data
in the cases that direct peer connections between clients cannot be established.

STUN and TURN are most commonly associated with realtime audio and video communications.
These are the cases in which peer connections are advantageous due to lower latency
as well as lower processing and bandwidth costs for the service operator.
It is also for this reason that authentication is mandatory to support TURN.
TURN allocations are time limited (10 minutes by default)
with the option to refresh.

For UDP connections to a destination,
clients can connect to the relay over UDP or TCP, or their secured variants, DTLS or TLS.
Since STUN messages contain a length field,
they can be transmitted over a reliable stream without issue.
Clients can then send an `Allocate` request,
allocating a UDP port on the relay for communications.
The client can then send data with `Send`,
containing both the destination and the data.
Alternatively, clients can request a channel with `ChannelBind`
and send data with `ChannelData`, omitting the STUN header
and using just a 4 byte ID for a lower overhead method of communications.

For TCP connections,
clients first connect to the relay with either TCP or TLS,
establishing a control connection.
As with UDP, an `Allocate` request allocates a (TCP) port.
The client can then send a `Connect` request with the intended destination.
If the relay connects successfully, it will reply with a `CONNECTION-ID`.
The client can then open a separate data connection to the relay,
and associate to the requested connection
with a `ConnectionBind` including the `CONNECTION-ID`.
All further data on the data connection
is then relayed between the client and destination without modification.

Specification wise, RFC 5766 for TURN only specifies support for UDP allocations,
given the intended use case for audio/video communications.
TCP support was added in RFC 6062.

In terms of software, there are various server implementations,
a non exhaustive list includes:
coturn, restund, reTURNServer, rfc5766-turn-server, turnserver.
Of these, only coturn supports the full range of protocols,
others only support the base protocol for UDP connections.
Public client library support is more scarce,
for C/C++ the server implementations can be partially reused,
Erlang has a `processone/stun` library with TURN support,
and Go has `pion/turn` library.
Further support can be found in browsers which expose the functionality under WebRTC.
Of theses client libraries, none support TCP connections,
and the only locateable instance of client code for TCP was in coturn's test client.

## methodology

### TURN TCP

To test the full range of protocols,
it is necessary to implement TCP support (RFC 6062).
From the libraries mentioned in the previous section,
`pion/turn` was selected for extension,
based on the existing framework,
the availability of other protocols for later parts,
and the author's familiarity with the Go language.
As a proof of concept,
the code is a relatively straightforward implementation of the RFC,
reusing large parts of the code already existing for UDP communications,
and is available upstream on the `rfc6062` branch.
However, there remains API design issues,
which are a blocker for merging into the main release.

### forward

This section describes a mode of operation
in which a proxy runs as a SOCKS server and TURN client,
translating SOCKS into TURN, forwarding the request to a TURN relay.
The relay then makes the connection to the final destination.

For UDP, in theory,
the time to establish a TURN session from the proxy to the relay
is when a client first establishes a TCP control connection to the proxy.
However, due to various limitations,
such as poorly behaving client or server library restrictions,
it may not be possible to associate client UDP packets with a TCP session.
For this reason,
a new TURN session is started on demand upon an incoming client UDP packet,
keyed by the client's source address.
However, this may slowly leak sessions
as the aforementioned inability to associate with control channels means
there is no point at which the session can be safely shut down.

For TCP, a TURN session can be started per SOCKS connection,
and the connection oriented nature means it is much more straightforward to
relay the packets.
However, this approach may run into per user session quotas,
if the TURN relay has it configured,
earlier than UDP as there is absolutely no session reuse.

### reverse

This section describes using a TURN relay as a known whitelisted endpoint
to establish a tunnel through a NAT or firewall.
The initiator "proxy" connects to the relay from within a restricted network,
the relay then forwards the connection to a static endpoint "server".
The server can then start a SOCKS server and tunnel all connections to the proxy,
which makes the outgoing connections from within the restricted network.

There is more flexibility within this design,
as we control both ends of the connection with the TURN relay.
We can limit the TURN protocol to just UDP, as this is the most widely available.
To multiplex multiple connections over a single TURN session/connection,
we use QUIC.
QUIC is a transport protocol on top of UDP, originally designed by Google
and currently undergoing standardiztion at the IETF.
With QUIC implementations already running for browsers (Firefox, Chrome),
servers (Nginx), and cloud providers (Google, Cloudlfare) as part of HTTP/3,
QUIC is likely to remain well supported for some time to come.
This gives us multiplexed, bidirectional streams between the proxy and server.

On top of QUIC, a lightweight messaging protocol is used.
A message consists of a length as a 4 byte unsigned integer in big endian,
followed by the message body.
TCP and UDP connections open with the first message containing `udp` or `tcp`.
The second message contains the destination address.
Afterwards, for TCP connections the contents of the stream
are directly relayed to the destination without further interference.
For UDP, the packet boundaries are preserved
by encapsulating them in a message as described above.
In the future, if the datagram extension to QUIC is standardized and implemented,
UDP packets can use that instead.

### testing

To test our implementation,
we selected a handful of popular hosted videoconferencing software
to attempt to connect to.
To do so, we signed up for the various services,
using credentials associated with our own accounts for testing.
Despite being called "long-term credentials",
those used in TURN authentication are in reality short lived and supplied on demand.
There was a draft recommendation on the requesting credentials over HTTP/JSON,
however it was never standardized and each product uses their own way.

To avoid the tedium of reverse engineering
the credential exchange mechanisms of multiple services,
we patched Chromium to output the credentials it received
and used to connect to TURN relays.
Obtaining the credentials was then a matter of making a call through the web browser,
and copying the credentials from the debug log.
It should be noted that reverse engineering the credential exchange mechanisms
would be unavoidable if stable long term credentials are desired.

## results & discussion

### forward & reverse

Given TURN relays that support the protocols,
forwarding connections works as expected for both TCP and UDP.

Reverse connections work even better,
as it only relies on well tested library code.

### limitations

There are serveral issues with the implementation,
such as the session reuse and allocation leakage mentioned earlier,
as well as a lack of graceful error handling.

Another limitation is with domain names.
In the forwarding configuration,
this is limited by both the choice of SOCKS library
and the TURN protocol,
which does not support addressing hosts through domain names.
In the reverse configuration, this is only limited by the SOCKS library,
and a different implementation would not have this restriction.
As a result of the above, only raw IP addresses
or names resolvable from the public network can be used to address the destination.

As a practical issue for red teaming operations,
the statically linked final binary weighs in at 16MiB,
this can be reduced to 4.1MiB by stripping out debug symbols and packing with `upx`,
but it is still a large executabe.
This is a general limitation with Go programs in general.

### third party

#### No TURN relays

In the course of our testing, both Zoom and Google Meet do not appear to use TURN relays
for their videoconferencing products.
As such, there is nothing to test.

#### Restricted TURN relays

Cisco provides an online test instance for their Webex videoconferencing solution.
From this we were able to extract a hardcoded set of TURN credentials:
`ciscoThinClient / 1234abcd`.
Connecting to and allocating a port completed successfully for both TCP and UDP,
however, any further requests would be dropped,
resulting in timeouts and no usable connection.

Citrix offers GoToMeeting as their web conferencing product.
This also uses a set of hardcoded credentials: `citrixturnuser / turnpassword`.
Connecting succeeds, but allocations fail with with an error
stating `Wrong Transport Field` for both TCP and UDP.
More investigation would be needed to determine the transport field it uses,
but even so it would be considerably less useful for proxying arbitrary connections.

Slack has calls within its main product.
Enable Security had previously successfully connected to Slack's TURN relays,
however, since then, they have presumably fixed the issue
and announced a migration to Amazon Chime, Amazon's hosted communications service.
At the time of testing, the TURN servers exposed were Amazon Chime servers,
which only allowed UDP allocations,
but restricted making connections to outside addresses.
Further testing would be needed to identify unrestricted address ranges.

#### UDP TURN Relays

Microsoft Teams, Jitsi Meet, BlueJeans, Facebook Messenger,
and Riot.im (a Matrix client)
all had TURN relays that allowed UDP connections to the public internet.
None of them had TCP support enabled,
and all used credentials generated on demand with short validity periods.
These services used various methods to convey the generated credentials to clients,
such as in cookies for Microsoft Teams
and in XMPP messages over WebSockets for Jitsi Meet.

### defense

For the operator of a secured network,
there is unfortunately not a lot that can be done
with regards to blocking unwanted connections.
Deep packet inspection and network flow analysis may offer some insight,
though self signed and pinned certificates
in many legitimate applications will limit their usefulness.
Previously, RFC ... did specify the possibility of discovering local STUN/TURN relays
through DNS, giving operators control over outgoing connections,
though is is unclear if any clients actually support it.

For the operator of a TURN relay,
there are many options in building a multilayered defense against potential abuse.

First would be hiding the relays.
A simple method is running the relays with non standard ports,
something that may already to done to bypass pesky firewalls.
More involved would be only running the secured variants of the underlying transport,
DTLS or TLS, and using Server Name Indication (SNI) to route traffic.
This would prevent casual scans based on the IP address
from learning the existence of relays,
but would do nothing against more targeted approach,
as used earlier.

The next line of defense is authentication.
Despite its name, the "long-term credentials" in TURN
are often only valid for a short period and generated on demand.
While there were recommendations
on how to relay these generated credentials to clients,
there is no standard and each service does it in their own way,
requiring reverse engineering of the credential exchange mechanism for each service.
In the future, OAUTH tokens may be used, standardizing authentication flows.

A final line of defense rests with authorization,
the TURN operator should limit access to destinations as necessary,
including limiting the protocol, address ranges, and possibly port ranges,
as in normal use clients should only be connecting to and from ephemereal ports.

As a side note, TURN relays are used primarily as a fallback to peer to peer connections.
Due to bandwidth and other concerns,
this typically limits the service to approximately a dozen peers.
Scaling beyond this would require central media servers such as
MCUs (Multipoint Conferencing Unit) or SFUs (Selective Forwarding Unit).
If this approach is taken, a TURN relay should not be necessary.

## conclusion

## future work

Additional points of improvement would be in stability and feature support,
such as for IPv6 or passing domain names.
These are not protocol limitations but implementation limitations.
As a statically linked binary,
our proof of concept sizes up to be 16MiB uncompressed
and 4.1MiB after stripping and packing.
An alternative to shipping a large binary
would be to make use of existing programs such as
browsers or videoconferencing clients
which likely already include most of the code needed.

## references

[circuit]: https://flylib.com/books/en/4.179.1.22/1/
[cloudproxy]: https://www.semanticscholar.org/paper/CloudProxy%3A-A-NAPT-Proxy-for-Vulnerability-Scanners-Wang-Shen/48877c7dccc81fa24a1a7579c46c7eaadbf8e792
