This is a high-level description of the states the replicator goes through.

## The States

| State | Description |
|----|----|
| Offline | Unable to reach server |
| Connecting | Opening TCP / WebSocket connection |
| Busy | Connected and transferring data |
| Idle | Continuous replication has nothing to do; waiting for changes |
| Stopped | One-shot replication finished; or any replication has a fatal error |

## Flowchart

<img src="https://cloud.githubusercontent.com/assets/43241/25108141/9a95458c-2388-11e7-83d2-8f12da81bf41.png" width=261>

## Notes on "Offline"

**DISCLAIMER:** The Offline state has not been implemented yet. See [issue #98](https://github.com/couchbase/couchbase-lite-core/issues/98).

There are quite a few causes behind the offline state:

1. Device is in "airplane mode", networking switched off
2. Out of range of WiFi and/or cell signal
3. WiFi router is available but itself has no connectivity (DSL is down, cable modem unplugged, bad IP configuration, etc.)
4. DNS can't resolve the hostname (DNS servers down, wrong DNS configuration on device or router, or host is on a private network and hostname is not public)
5. Proxy server is unreachable (DHCP misconfiguration, proxy is down, proxy is up but misconfigured, etc.)
6. Hostname is known, but that IP address is on an unreachable private network like an intranet
7. Other network issues along the route to the server (ISP problems, a backhoe has cut a backbone fiber line, AWS went down again, etc.)
8. Server itself is down
9. Sync Gateway is down

The replicator detects these by the errors they produce, like No such host, No route to host, Connection refused, Connection timeout, 502 Bad Gateway, 504 Gateway Timeout. These cause a transition to the Offline state.

While offline, the replicator has a limited ability to detect when conditions might have improved. Changes to causes 1 and 2 can be detected by OS-specific network change events. The others can't, because they don't involve changes in the device's network interfaces. However, a change in network can mean that the problems are resolved (user may have switched WiFi networks or logged into a VPN, for example.)

While in the Offline state the replicator will listen for network-change events, and attempt to reconnect. It also tries periodically even without an event, using an exponential-backoff schedule.