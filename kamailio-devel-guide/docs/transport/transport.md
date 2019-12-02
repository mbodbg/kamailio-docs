As a developer, the interaction with the transport layer is lower
and lower. It is already implemented the support for UDP, TCP, TLS and SCTP.
From the modules, you can use the API exported by *sl*> and **tm** modules to send stateless replies, or to
send stateful requests/replies. Sending stateless requests can be done with the functions
from core, exported in file [**forward.h**](https://github.com/kamailio/kamailio/blob/5.3/src/core/forward.h).

The core takes care of receiving the messages from the network, the basic validation for them,
preparing the environment for a higher level processing of SIP messages. When developing new
extensions, you don't have to care about reading/writing from/to network.

TLS implementation is a module, residing inside modules/tls. It reuses the TCP layer from
the core for the management of the connection, while the code in modules/tls takes care
of TLS negotiation and encryption.

If you want to investigate the implementation of transport layers, you can start with:

* **udp_*.{c,h}** for UDP
* **tcp_*.{c,h}** for TCP
* **modules/tls/*.{c,h}** for TLS
* **sctp_*.{c,h}** for SCTP

# DNS Implementation #
Kamailio follows the requirements specified in RFC3263 for server and service location.
That includes support for **NAPTR** and **SRV** records as well.

To investigate the implementation related to DNS, start with files **resolve.{c,h}**.
There is an internal DNS cache which is necessary for doing DNS-based load
balancing and failover. It can be disable via core parameters in the configuration file.