The WebSocket implementation used by the client should support HTTP proxy servers; here are some notes about how to implement this.

## What's an HTTP proxy server?

The proxy server is part of the LAN the client is on. Such a LAN will have no connectivity to the outside Internet, so normal HTTP connections will fail. The only way to reach the Internet is to connect to an HTTP proxy on the LAN and send it requests. The proxy server has Internet connectivity and relays the requests.

(This is unrelated to the proxy servers used as middleware in front of servers. Those live in the data center, not the client LAN, and are usually _reverse_ proxies.)

The client device needs to know what proxy to connect to. Most of the time this is configured automatically using settings received from the DHCP server, but sometimes it needs to be configured manually via the network settings UI. The OS has an API to let apps know the proxy settings. (For example, on Apple platforms it's in `<CFNetwork/CFProxySettings.h>`.)

**Note:** There are other types of client-side proxy servers, but as far as I know, all the others operate at the TCP level (like SOCKS). As long as you're using a system networking API that's above the basic BSD-socket layer, it should know about these proxies and connect to them transparently.

## Algorithm

WebSocket connections need to use the HTTP `CONNECT` method to open a tunnel to the server. So the procedure for opening a WebSocket to `example.com` is:

1. Get system proxy settings
2. Determine whether a proxy should be used to connect to `example.com`. If so:
    1. Open a socket to the proxy
    2. Send `CONNECT example.com:4984 HTTP/1.1`. Also send a `Host:` header with value `example.com:4984`. You may need to send a `Proxy-Authorization:` header if the proxy requires credentials.
    3. Read the entire HTTP response.
    4. If the status is 407, the credentials are wrong/missing; if you can find credentials, open a new socket and go back to step (i).
    4. If the status is not 200, give up.
3. Or if _not_ using a proxy:
    1. Open a socket to `example.com`.
4. Send the WebSocket `GET` request as usual.

## References

* [How HTML5 Web Sockets Interact With Proxy Servers](https://www.infoq.com/articles/Web-Sockets-Proxy-Servers)
* [HTTP/1.1 specification of the CONNECT method](https://tools.ietf.org/html/rfc7231#section-4.3.6)
* [tinyproxy](https://tinyproxy.github.io) is an easy proxy server to set up for debugging/testing. You will just need to edit its config file to allow CONNECT requests to port 4984.