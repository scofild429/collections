---
title: "Network"
---

# Network

## The Core Principle

Network communication in Linux strictly follows a sequential, two-stage pipeline. The system must resolve a human-readable name to an IP address before it can determine the physical path to send the data. If Stage 1 fails, the process aborts immediately; it never guesses or forwards unresolved names to a gateway.

### Stage 1: Name Resolution (Finding the IP)

The goal of this stage is exclusively to translate a hostname (e.g., `github.com` or `node1`) into a machine-readable IP address.

1.  Control Switch (`/etc/nsswitch.conf`)

Defines the order of lookups.

Standard configuration: `hosts: files dns` (Check local files first, then query DNS).

1.  Local Phonebook (`/etc/hosts`)

A static, local mapping of IPs to hostnames.

Bypasses network overhead; highly secure and extremely fast.

Ideal for internal cluster nodes (e.g., `10.0.0.5  node1`).

If a match is found, the system skips DNS and immediately proceeds to Stage 2.

1.  Domain Name System (`/etc/resolv.conf`)

Engaged only if the local `hosts` file yields no results.

Queries servers in a strict top-down order defined by the `nameserver` directive.

Typically structured from internal to external (e.g., local cluster DNS -\> `8.8.8.8`).

Utilizes the `search` directive to auto-append domain suffixes for incomplete hostnames (e.g., automatically appending `.cluster.local` to `db-server`).

Critical rule: If DNS returns `NXDOMAIN` (Domain Not Found), the connection fails instantly.

### Stage 2: Routing (Finding the Path)

The system now holds a concrete destination IP address (or the user provided an IP directly). It must now decide which Network Interface Card (NIC) to use to transmit the packet.

1.  The Routing Table (`ip route`)

A dynamic table stored in OS memory acting as a delivery map.

The system scans this table to match the destination IP against defined subnets and interfaces.

1.  Exact Network Matches (Direct Link)

If the IP falls within a defined local subnet (e.g., `10.0.0.x` is mapped to the InfiniBand interface `ib0`), the packet is dispatched directly through that specific hardware interface.

1.  The Default Gateway (`0.0.0.0`)

The ultimate fallback route for unknown, external IPs.

If no specific subnet rule matches the destination IP (e.g., it is a public IP like `140.82.112.4`), the packet is forwarded to the Default Gateway.

The gateway (usually a router) then takes over, often performing Network Address Translation (NAT) to route the traffic out to the wider Internet (WAN).
