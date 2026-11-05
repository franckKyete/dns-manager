# Local Names Daemon - Architecture and Inner Workings

Draft v0.1 | 30 October 2026 | Companion to the Software Specification | Working name: devnamed (placeholder)

## 1. Purpose and Reading Guide

The Software Specification says what the system must do. This document explains how it is built and what happens inside it, step by step, in the situations that matter most. The central question it answers is: when a machine's IP address changes, how does the daemon find the new routes to the apps, and what does it update?

There is no source code here. Component names match section 6.1 of the specification, and requirement identifiers (for example FR-14 or MM-5) point back to it. Timings and thresholds are starting values taken from the specification and may be tuned.

Suggested reading order: sections 2 to 4 for structure, sections 5 to 7 for the core mechanics (the address-change walkthroughs are in 6 and 7), and the remaining sections for subsystem detail, peer-group behavior and failure handling.

## 2. Architecture at a Glance

The system has two planes. The **control plane** is the daemon: it decides what should exist, renders configuration, applies it and checks the result. The **data plane** is dnsmasq (answers DNS), Caddy (proxies HTTP and HTTPS) and the kernel network stack. All user traffic flows through the data plane. The daemon never relays a packet, so if it stops, DNS and proxying keep working with the last applied configuration.

```
      scripts / CLI                         other machines (peers)
            |                                       |
    [ control socket ]                    [ peer channel, mTLS ]
            |                                       |
  +---------+---------------------------------------+---------+
  |                    DAEMON (control plane)                 |
  |                                                           |
  |  control server --> CORE (single-writer loop) <-- peers   |
  |                       |  registry + state store           |
  |  network watcher -----+----- liveness monitor             |
  |                       |                                   |
  |                  RECONCILER                               |
  |      DNS manager    proxy manager    host resolver mgr    |
  +----------+---------------+----------------+---------------+
             |               |                |
          dnsmasq          Caddy        systemd-resolved
        (DNS, port 53)  (ports 80/443)  (this machine's DNS client)
             ^               ^
             |               |
         LAN clients     LAN clients          (data plane)
```

What the daemon talks to on one machine:

| Piece | Role | How the daemon interacts |
| --- | --- | --- |
| dnsmasq | DNS server for the LAN | Writes its drop-in files, restarts or reloads it through systemd, queries it to verify answers |
| Caddy | Reverse proxy and local certificate authority | Writes per-route files, asks it to validate and reload through its local admin interface, sends test requests |
| Kernel | Source of network facts | Subscribes to netlink events for links, addresses and routes; reads the neighbor table |
| systemd | Service manager | Watches unit state over D-Bus, starts units, receives sleep and resume signals via logind |
| systemd-resolved | This machine's own DNS client | Sets per-domain routing so managed zones go to the local dnsmasq |
| NetworkManager (optional) | Connection manager | Listens to connection events over D-Bus; never required |
| Peers | Daemons on other machines | Mutually authenticated channel for heartbeats and replication |

## 3. Core Design Decisions

1. **Desired state and applied state.** The daemon never edits dnsmasq or Caddy ad hoc. It keeps a description of what should exist (desired state), and a reconciler makes dnsmasq and Caddy match it. Every trigger, whether a new route, an address change or a crash recovery, goes through the same path, so there is one thing to test and nothing to forget.
2. **A single writer.** All changes to desired state are handled one at a time by one loop. There are no locks around the registry, ordering is well defined, and a registration racing with an address change cannot corrupt anything.
3. **Routes hold targets, not addresses.** A route records where an app lives in a stable form (this machine and a port, a named peer and a port, a hostname, a MAC address). It never stores a resolved IP. IPs are derived when configuration is rendered, which is why an address change is cheap.
4. **Derived facts are tracked with dependencies.** Whenever the daemon derives something, such as the address of machine B or the IPs behind a hostname, it records which routes depend on it. When that fact changes, exactly those routes are re-rendered.
5. **The daemon stays out of the data path.** A daemon crash, restart or upgrade does not interrupt DNS answers or in-flight requests.
6. **Atomic, reversible changes.** Every change to dnsmasq, Caddy or the host resolver follows the same steps: render to a staging area, validate with the tool's own checker, swap in, apply, verify, and roll back to the last known good version on any failure.
7. **Peers are equals, and every route has one owner.** Each machine runs the full stack. Only a route's owner changes it, so replication between machines needs no consensus and keeps working when machines cannot see each other.

## 4. Data Model

Some data is persisted and survives restarts; other data is derived and rebuilt on start.

| Entity | Key contents | Persisted |
| --- | --- | --- |
| Node | node identity, name, advertised address, status, last seen | Yes (this machine and known peers) |
| Zone | name, TLS mode, owner-offline policy, zone settings | Yes |
| Route | identifier, hostname, zone, owner node, target, options, bindings, state, revision | Yes |
| Binding | kind (lease, process, path) and parameters, expiry | Yes; processes and paths are re-validated at start |
| Port allocation | port and route identifier | Yes (this machine only) |
| Peer record | identity key, last known address, last known MAC address, last sequence number seen per owner | Yes |
| Network snapshot | interfaces, addresses, default route, interface classes | No; re-read at start |
| Resolution cache | hostname to addresses with expiry; MAC to address | No |
| Health state | per route: healthy or unhealthy, last probe time | No |

### 4.1 Targets

A target says where an app lives. The daemon turns it into a dial address (what the proxy connects to) only at render time.

| Kind | Meaning | Resolved to | Changes when |
| --- | --- | --- | --- |
| local | App on this machine, with a port | Loopback address | Never |
| node | App on a named peer machine, with a port | That peer's current advertised address | The peer announces a new address |
| host | A hostname and a port | Addresses from DNS lookups, re-resolved on expiry and after probe failures | The hostname's DNS answer changes |
| mac | A device identified by MAC address, with a port | IP from the neighbor table | The device's address changes and the daemon observes it |
| address | A literal IP and a port | Itself | Never; flagged as fixed, with a warning if probes fail |

### 4.2 Two addresses per route

A route involves two different addresses, and they change for different reasons.

- **Serving address**: the address DNS answers for the route's hostname. It is the advertised address of the machine whose proxy serves the name, normally the route's owner.
- **Dial address**: the address that proxy connects to in order to reach the app. For a local target it is loopback and never changes. For other target kinds it is derived as in the table above.

Example: an app on machine B, registered on B with a local target. The serving address is B's advertised address; the dial address is loopback on B. When B's IP changes, only the serving address changes, and DNS is the only thing that needs updating.

### 4.3 Versions and ordering

- Each change to desired state increments a local **revision**. The reconciler records which revision each subsystem has applied, so lag is visible in status.
- Each owner stamps its own route changes with a **per-owner sequence number**, which is how peers know what they are missing (section 10).
- **Hybrid logical clocks** order events across machines without depending on wall-clock accuracy.

## 5. Inside the Daemon

### 5.1 Event sources

Everything that can cause work is normalized into a typed event and placed on one queue.

| Source | What it reports | Mechanism |
| --- | --- | --- |
| Control socket | Register, unregister, pin, renew, status requests | Local Unix socket, caller identified by kernel credentials |
| Kernel | Link up or down, address added or removed, default route changed, neighbor entries | Netlink subscription |
| Sleep and resume | System about to sleep, system resumed | logind signals over D-Bus |
| Service state | dnsmasq, Caddy or systemd-resolved started, stopped or restarted | systemd and D-Bus signals |
| Connection manager | Connection activated or deactivated | NetworkManager D-Bus signals, if present |
| Filesystem | A bound path disappeared; configuration file changed | inotify watches |
| Timers | Lease expiry, health probes, heartbeats, safety poll, audit tick | Internal scheduler |
| Peers | Heartbeats, address announcements, replicated changes | Peer channel |
| Signals | Terminate, reload configuration | Unix signals |

### 5.2 The core loop

The core takes one event at a time. For each it validates the input, changes desired state, increments the revision, writes the change to the state store before doing anything else (so a crash cannot lose an acknowledged change), and wakes the reconciler. Slow work, such as probes, rendering, file writes and service restarts, never runs inside the core. Workers do it and report back as new events.

### 5.3 The reconciler

The DNS manager, proxy manager and host resolver manager each run the same pipeline independently, with their own backoff:

1. **Plan**: compute what the output should be from desired state and derived facts; skip everything if the content hash is unchanged.
2. **Render**: produce the files into a staging area, with stable ordering so identical input gives identical output.
3. **Validate**: run the tool's own checker (dnsmasq's syntax test, Caddy's validation), per route first where possible.
4. **Swap**: atomically replace the live files, keeping the previous ones as last known good.
5. **Apply**: reload or restart the service.
6. **Verify**: ask the service a real question (a DNS query for a sample name, a request with the route's Host header) and compare the answer to the expectation.
7. **Commit or roll back**: record the applied revision on success; on failure restore the last known good files, re-apply them and mark the affected routes failed.

Changes are coalesced before running: route changes wait up to 200 ms (never more than 500 ms in total) so a burst of registrations becomes one reload, and network changes wait for the quiet window described in section 6. A registration request that asked for a synchronous answer waits until the relevant subsystems report an applied revision at or beyond the request's revision, then responds.

### 5.4 The dependency index

This is how an address change reaches the right routes without touching the rest. When the renderer produces output for a route, it records the facts that output depended on as keys, for example the advertised address of node B, the lookups of a hostname, or the neighbor entry for a MAC address. The core keeps an index from each key to the set of routes that depend on it.

When a fact changes, the core looks up the key and marks only those routes dirty. The reconciler then re-renders them. A periodic full audit (every 60 seconds) re-renders everything and compares hashes, which catches any change that slipped past an event.

## 6. Detecting That This Machine's Address Changed

The daemon does not poll for this. The kernel announces it.

1. **Kernel event.** The kernel sends a netlink message when an interface goes up or down, an address is added or removed, or the default route changes. The network watcher receives it within milliseconds.
2. **Relevance filter.** The watcher classifies the interface (physical, Docker or libvirt bridge, virtual pair, VPN tunnel) using the configured include and exclude lists. Events on ignored interfaces are dropped. Tentative addresses (still undergoing duplicate address checks) are ignored until they settle.
3. **Debounce.** DHCP renewals, Wi-Fi reconnects and resume from sleep produce bursts of events, often removing an address and adding it back. The watcher waits for a quiet window of 2 seconds, with a hard cap of 10 seconds so a flapping interface cannot postpone a decision forever.
4. **Snapshot and selection.** After the window, the watcher reads the full current state: interfaces, addresses and routes. The address selector applies the policy from the specification (configured override, then the default-route interface's primary IPv4 address excluding virtual interfaces, then any private-range address on a physical interface, then loopback).
5. **Compare.** If the selected address equals the previous one, the process ends here. This is the outcome for most events, such as a lease renewal that kept the same address.
6. **Commit.** If the address differs, the core updates this machine's node record with the new address, increments the revision and persists it.
7. **Fan-out.** The new fact triggers, in parallel: the DNS manager re-render (section 7), an announcement to every peer, an `address-changed` event and hooks, and a re-check of the host resolver settings.
8. **Warn about stale clients.** Other devices that were told this machine's old address as their DNS server will keep using it until reconfigured. Status and doctor say so explicitly, and mention that a DHCP reservation would prevent it.

After a suspend and resume, the watcher forces a snapshot once links have settled, and a 30 second safety poll compares snapshots in case any event was missed.

Typical timeline for one machine:

| Time | What happens |
| --- | --- |
| 0 s | Kernel reports the address change |
| 0 to 2 s | Debounce window; more events may arrive |
| about 2.1 s | Snapshot read, address selected, change detected |
| about 2.2 s | Node record committed; peers notified |
| about 2.3 s | DNS render and validation |
| about 2.5 s | New configuration swapped in and applied |
| about 3.5 s | Verified by querying the local DNS server |
| up to 30 s later | Clients' cached answers expire (short record lifetime) |

## 7. How Routes Follow an Address Change

### 7.1 Principle

Because routes store targets and not IPs, an address change is a change to one fact, the advertised address of a node. The dependency index finds every route whose output depended on that fact, and only those are re-rendered. Nothing is looked up per route at request time, and no route is edited by hand.

### 7.2 Walkthrough A: the app is on the machine whose address changed

Machine B runs an app and registered it with a local target. B's address changes from the old one to a new one.

1. B's watcher detects the change (section 6) and B's node record gets the new address.
2. The dependency index lists every route whose serving address depends on node B. Each is marked dirty.
3. B's DNS manager re-renders the answers for those routes with the new address, validates, applies and verifies.
4. B's Caddy configuration is untouched: it listens on all interfaces, and its upstream is loopback, which did not change.
5. B announces the new address to its peers, which update their own DNS answers for B's routes (walkthrough B).
6. Clients refresh their cached answer within the record lifetime, and new connections go to the new address. Open connections to the old address are lost, as with any address change.

### 7.3 Walkthrough B: a peer's address changes (two machines)

Machine A and machine B are peers. B's address changes.

| Time | On machine B | On machine A |
| --- | --- | --- |
| 0 s | Kernel reports the change | Still using B's old address |
| about 2 s | Address selected; node record updated | Heartbeats to B may start failing |
| about 2.2 s | Announcement sent to A (to A's address, which did not change) | Receives announcement, verifies it came over the authenticated channel, updates B's node record |
| about 2.3 s | B's DNS and own routes re-rendered | Dependency index finds routes served by B; A's DNS records for them are re-rendered |
| about 3.5 s | Verified | Verified; exact-name records are re-read without a restart, wildcard routes need a short coalesced restart |

A also records the address the announcement arrived from, because that is the address that demonstrably works, and compares it with the announced one. If they disagree, the observed one wins and a warning is raised.

### 7.4 Walkthrough C: a route whose target is a peer

A route served by machine A has a `node` target pointing at machine B (A's proxy forwards to B). When B's address changes, two things depend on it: nothing in DNS (the serving address is A's), but A's Caddy upstream line for that route. The dependency index marks the route dirty, A re-renders its Caddy file with B's new address, validates it, reloads Caddy gracefully and verifies the route end to end. Existing connections through the old upstream finish normally.

### 7.5 Walkthrough D: other target kinds

- **host target**: the daemon resolves the hostname, caches the answer for its lifetime and records the dependency. It re-resolves when the answer expires and also immediately after a health probe failure. If the addresses differ, the dependent routes are re-rendered.
- **mac target**: the daemon looks the MAC address up in the neighbor table, which the kernel keeps current as devices talk on the network, and subscribes to neighbor changes for that MAC. When the entry shows a new IP, the dependent routes are re-rendered. If the entry has expired, the daemon sends a targeted neighbor probe for the last known address before declaring the target unreachable.
- **address target**: the IP is taken as fixed. If probes fail, the route is marked unhealthy with a message explaining that the address may have changed, since the daemon cannot discover this by itself.

### 7.6 When the announcement is lost

If B changes address while A is unreachable, or both change at once, B's announcement goes to an address that no longer works. Peers then run a rediscovery ladder, stopping at the first success:

1. Reconnect to the peer's last known address and any additional addresses it advertised.
2. Look for the peer by multicast service discovery on the LAN, which needs no DNS.
3. Resolve the peer's name as the router's DHCP or local DNS knows it, if configured.
4. Look up the peer's recorded MAC address in the neighbor table, and probe the subnet for that MAC with a rate-limited neighbor sweep (optional, off by default).
5. Otherwise the peer stays unreachable and the owner-offline policy applies to its routes. Rediscovery keeps retrying with backoff, and any later successful contact, from either side, repairs the record.

Authentication does not depend on addresses, so a peer found at a new address is accepted only if it proves the identity key recorded at pairing.

### 7.7 What the daemon cannot fix

- Devices configured with this machine's old address as their DNS server. The daemon detects and reports the situation, and a DHCP reservation prevents it.
- Apps bound to one specific old IP. They need to listen on loopback or on all interfaces; the daemon reports the failing probe.
- Third-party hosts registered by literal IP. See the address target above.

## 8. Subsystem Internals

### 8.1 Network watcher and address selector

The watcher holds netlink subscriptions for links, IPv4 and IPv6 addresses, routes and neighbor entries. It keeps a current network snapshot in memory and rebuilds it from scratch after any wake-up, so it never depends on having processed every individual event. The selector is a pure function from a snapshot and the configuration to an advertised address and a reason string ('default-route interface wlan0'), which status shows. A minimum interval of 5 seconds between DNS applications limits the damage from an interface that flaps, and a flapping warning is logged. When no usable LAN address exists, the selector returns loopback and the daemon enters degraded mode (FR-19).

### 8.2 DNS manager

- **Output on a single machine:** one generated drop-in file containing, per zone, a wildcard answer for the advertised address, a declaration that the zone is local (unknown names get a negative answer instead of leaking upstream), and suppression of IPv6 answers.
- **Output in a peer group:** exact-name routes are written to a hosts-style file that dnsmasq re-reads when signalled, so ordinary route changes need no restart. Wildcard routes need address lines in the drop-in, which dnsmasq only reads at start, so they trigger a short restart. Each name is answered with the advertised address of the machine serving it (section 4.2).
- **Listening:** dnsmasq is configured to bind to interface addresses dynamically, so a new local address is picked up by dnsmasq without any action from the daemon. What changes on an address change is the content of the answers, not where dnsmasq listens.
- **Record lifetime:** answers carry a short lifetime (30 seconds by default) so clients converge quickly after a change.
- **Apply and verify:** render to staging, run dnsmasq's syntax check, swap, signal or restart, then query the local server for a sample name per zone and compare with the expected address. A restart takes well under a second; clients retry automatically.
- **Ownership:** the daemon only ever writes files it marked as generated. It reports, but does not fix, conflicts such as another process holding port 53.

### 8.3 Proxy manager

- **One file per route** in a directory that the main Caddy configuration imports. A bad route can therefore be quarantined without affecting others.
- **Validation** runs per route first and then for the combined configuration, using Caddy's own validator.
- **Reload** goes through Caddy's local admin interface, which swaps configuration gracefully without dropping connections. If the interface is unavailable the daemon falls back to restarting the unit and records the loss of graceful reloads.
- **Verification** sends a request with the route's Host header to the local listener and checks that the response comes from the proxy and not from a daemon-generated error page.
- **Built-in pages:** unknown names inside a managed zone get a plain 'no such route' page; routes whose upstream is not answering get a page naming the hostname and upstream.
- **Certificates:** Caddy's internal authority issues them, either per host on first request or as one wildcard per zone. In a peer group all machines share one root (section 10.8).

### 8.4 Host resolver manager

On machines using systemd-resolved, the daemon registers each managed zone as a routing-only domain pointing at the local dnsmasq. Other lookups are untouched. It re-applies the setting when systemd-resolved restarts or links change. On other setups it only reports what to configure. For `.local` zones it also checks which multicast DNS components would intercept the lookup.

### 8.5 Liveness monitor

- **Health probes:** a TCP connection attempt to each route's dial address every 10 seconds, with a 2 second timeout. A route changes state after two consecutive results, to avoid flapping. A failed probe also triggers an immediate re-resolution of host and mac targets (section 7.5).
- **Leases:** timers per route; a missed renewal removes the route after its time-to-live.
- **Process bindings:** the daemon holds a handle to the bound process and is notified when it exits.
- **Path bindings:** inotify watches on the parent directory, plus a check during each audit, remove the route when the path disappears.
- **Garbage collection:** routes unhealthy longer than the stale period (default 7 days), with no live binding and not pinned, are removed and reported.

### 8.6 Port allocator

When asked for a port, it derives a starting point from a hash of the hostname so a name tends to get the same port each time, then probes forward within the configured range until it finds a port that nobody is listening on and no other route holds. The allocation is persisted and released on removal. It reserves the number only; the app still has to bind it.

### 8.7 Service supervisor

The supervisor checks the state of the dnsmasq, Caddy and systemd-resolved units over D-Bus, starts them if they are inactive and the administrator has allowed it, and reports their state changes to the core as events. A restart of any of them causes the reconciler to re-verify and re-apply within 5 seconds.

### 8.8 Control server and request lifecycle

1. Accept a connection on the local socket and read the caller's identity (uid, pid) from the kernel.
2. Parse and validate the request against strict rules: hostname syntax, zone membership, target kind and parameters, allowed upstream ranges, reserved names.
3. Authorize: socket group members may manage routes in allowed zones; administrators may do more.
4. Rate-limit per caller.
5. Submit to the core as an event and receive the new revision number.
6. For a synchronous request, wait until the reconciler reports that the relevant subsystems applied that revision, up to the timeout (10 seconds by default); for an asynchronous request, return a request identifier at once.
7. Respond with the route's state, any warnings and the allocated port if any. Repeating the same request from the same owner is a no-op that returns the current state.
8. Record the call in the audit log.

### 8.9 State store

The store is transactional. The core writes each accepted change before acknowledging it. The store carries a schema version, migrates forward on upgrade after taking a backup, and keeps the last 5 known-good snapshots. Rendered files are never the source of truth: they can always be regenerated from the store and the configuration.

### 8.10 Events, logs and status

Every significant transition produces an event on an in-process bus, which feeds the event stream for clients, the hooks directory and the structured journal logs. The status view is computed from the same internal state: daemon state (ready, reconciling, degraded), advertised address and the reason, subsystem applied revisions, route counts by state, peer states and replication lag, and warnings. The doctor command runs the checks listed in the specification and ties each failure to a fix hint.

## 9. Key Flows

### 9.1 Registering a route (single machine)

1. A script runs the register command with a hostname, a target and optional bindings.
2. The control server validates and authorizes it (8.8) and submits it to the core.
3. The core checks for a name collision, allocates a port if requested, writes the route to the state store and increments the revision.
4. The reconciler renders the Caddy file for the route, validates it, swaps it in, reloads Caddy and verifies the route. DNS needs no change on a single machine because the zone wildcard already covers the name; in a peer group the DNS manager adds the record for the name (and signals dnsmasq).
5. The liveness monitor starts probing the dial address. The route is reported pending until the app accepts connections.
6. The control server answers the script. Elapsed time is typically under 2 seconds.

### 9.2 Removing a route

Explicit removal and binding lapse follow the same path. The core deletes the route, releases its port and, in a peer group, writes a tombstone. The reconciler removes the route's Caddy file (and DNS record, in a peer group), reloads and verifies that the name now returns the 'no such route' page.

### 9.3 Start-up

1. Take the exclusive instance lock.
2. Load the state store; apply migrations; drop expired leases and re-validate process and path bindings.
3. Open the control socket immediately; status says 'reconciling'.
4. Read the network snapshot and select the advertised address.
5. Verify dnsmasq, Caddy and the host resolver; start units if needed and allowed.
6. Run a full reconciliation of DNS, proxy and host resolver; start probes and heartbeats; connect to peers.
7. Status becomes 'ready' once the applied revisions match the desired revision.

### 9.4 Daemon crash and restart

dnsmasq and Caddy keep serving with the last applied configuration. systemd restarts the daemon, which follows the start-up flow. Because acknowledged changes were persisted before being acknowledged and rendered files are reproducible, nothing is lost and no manual repair is needed.

### 9.5 dnsmasq or Caddy dies

The supervisor sees the unit state change and reports it. The reconciler starts the unit (or waits for systemd's restart policy), re-verifies the files by hash, re-applies if needed and runs the verification queries. Target time to full service is under 10 seconds.

### 9.6 Network disappears and returns

The selector finds no usable LAN address and returns loopback; the daemon is degraded but local names keep working on this machine. Peers see missed heartbeats and apply the owner-offline policy to this machine's routes. When an address returns, the normal address-change flow (section 6) runs and peers are told the new address.

## 10. Peer Group Internals

### 10.1 Pairing and channel

The first machine creates the group and prints a single-use join token that expires in 10 minutes. The second machine redeems it; both exchange identity keys and record each other. From then on they talk over a dedicated port using mutual TLS with those pinned keys, on LAN interfaces only. Identity is the key, not the address, so a peer can move to any address and still be recognized.

### 10.2 Liveness

Heartbeats every 2 seconds carry the sender's current advertised address and its highest sequence numbers. Three missed heartbeats (6 seconds) mark a peer unreachable and raise an event. Recovery of a peer clears the state automatically.

### 10.3 Replication

Each owner numbers its own route changes. Peers push every change as it happens (typically under 1 second) and every 30 seconds compare the highest sequence number held per owner, pulling anything missing. Deletions are kept as tombstones for 7 days and then compacted; a machine that was away longer takes a full snapshot instead. Because each record has one writer, merging is just 'take the newer version from the owner', with no conflicts.

### 10.4 Validation on receipt

A receiving daemon applies the same rules as for a local registration: the hostname must be inside a managed zone, the target must be permitted, reserved names are refused. It also checks that the sender owns the route being changed. A peer can therefore never alter another machine's routes or publish names outside the zones, even if it is buggy.

### 10.5 DNS and proxy in a peer group

Every machine runs dnsmasq and Caddy and holds the complete record set. Each route's DNS answer is the advertised address of the machine serving it, by default its owner, and each machine's Caddy serves the routes it owns. Clients should be offered at least two DNS servers so that one failing does not stop resolution.

### 10.6 Walkthrough: machine A goes offline while B keeps working

1. A is switched off. B notices missed heartbeats after 6 seconds and marks A unreachable.
2. B's DNS server and Caddy continue to serve everything from their local copies, including B's own routes.
3. Clients that used A as primary DNS fail over to B after their resolver timeout, typically 1 to 5 seconds.
4. A script registers a new route on B. B accepts it, applies it and queues it for A.
5. A comes back. Heartbeats resume, A compares sequence numbers, pulls B's new route, and renders it into its own DNS and proxy within seconds.

If A's routes had been affected, B would have applied the owner-offline policy: by default `keep` (leave the records, so clients simply fail to connect to A's apps), or `fallback` after a grace period (point those names at B's friendly 'machine A is offline' page).

### 10.7 Conflicts

If two machines registered the same name while disconnected, then on reconnection the earlier registration (by hybrid logical clock, ties broken by machine identity) wins. The other route is marked conflicted, is not answered by DNS and raises an event so its owner can rename it.

### 10.8 Shared certificate authority

The first machine creates the root and intermediate certificates. The signing material is sent to the other machines over the authenticated channel and stored where only the Caddy service can read it. Each machine's Caddy issues certificates for the names it serves, so issuance works even when machines cannot see each other, and any device that trusts the one root works with all of them.

## 11. Concurrency and Consistency Model

- **One writer per machine:** all changes to local state pass through the core loop in a total order.
- **Read-your-writes:** a synchronous registration returns only after the route has been applied and verified on this machine.
- **Crash consistency:** a change is persisted before it is acknowledged; external files are derived from persisted state.
- **Idempotence:** repeating any request or any reconciliation step has the same result as doing it once.
- **Across machines:** eventual consistency with one writer per route. Registrations are accepted while disconnected, and merges after reconnection are deterministic.
- **Time:** leases and timers use monotonic clocks; ordering across machines uses hybrid logical clocks; clock skew above 5 seconds raises a warning because it affects certificates.

## 12. Failure Handling Mechanisms

- **Isolation:** each subsystem (DNS, proxy, host resolver, peers) has its own pipeline, backoff and status, so one failing subsystem does not block the others.
- **Last known good:** every external file has a previous version kept; any failed apply restores it and re-applies.
- **Backoff:** retries after failures start at 1 second and double to a 60 second cap. After repeated failures the subsystem is marked failed and surfaced in status, with the underlying error.
- **Quarantine:** an invalid route is set aside and marked failed without affecting other routes.
- **Degraded modes:** no network (loopback only), peer unreachable (local operation continues), a missing dependency (registrations accepted and applied once it appears).
- **Audit:** a 60 second full audit compares every managed artifact and service state with desired state and repairs differences, which covers any event that was missed.

## 13. Security Architecture

- **Trust boundaries:** local callers (group members and administrators), paired peers, LAN devices that only send DNS and HTTP requests, and the rest of the network, which gets nothing.
- **Privilege separation:** the daemon runs unprivileged and can only restart or reload the dnsmasq and Caddy units through a narrow policy rule. Caddy's certificate authority key stays under Caddy's own account.
- **Input pipeline:** every value from the control socket or a peer passes strict validation before it can reach a rendered file, and is escaped on output. Upstream targets are limited to loopback, private and link-local-excluded ranges unless an administrator allows more.
- **Network exposure:** DNS and proxy listen only where intended and are limited to the LAN by firewall guidance and doctor checks; the peer port is LAN-only; Caddy's admin interface is never reachable from the network.
- **Peer trust:** mutual TLS with pinned identity keys, single-use short-lived join tokens, receiver-side validation and ownership enforcement.

## 14. Deployment Layout (proposed)

| Item | Proposed location or value |
| --- | --- |
| Daemon unit | systemd system service, starts at boot |
| Configuration | system configuration directory, one file |
| State store and backups | daemon's state directory |
| Control socket | runtime directory, group-readable |
| dnsmasq drop-in and hosts-style file | a directory included by dnsmasq's main configuration |
| Caddy per-route files | a sites directory imported by the main Caddyfile |
| Hooks | a hooks directory of executables |
| Ports | 53 (DNS), 80 and 443 (proxy), one cluster port (peers) |

## 15. Risks and Open Design Questions

1. **Rediscovery of a moved peer** relies on multicast discovery, router-provided names or neighbor probing. Which of these are reliable on the user's network needs testing; the neighbor sweep is optional because it is noisy.
2. **Client-side caching:** browsers and operating systems may cache DNS answers longer than the advertised lifetime, so convergence after an address change can take longer than 30 seconds for some clients.
3. **dnsmasq restarts** for wildcard changes cause a brief cache flush. An embedded DNS responder inside the daemon would remove the restart and the hosts-style file, and is attractive in a peer group where the daemon already holds all records.
4. **Unprivileged operation** must be prototyped to confirm that everything the daemon needs works with a narrow policy rule instead of root.
5. **Network manager variety:** NetworkManager, systemd-networkd and manual setups emit different higher-level events; the design relies on kernel events so that behavior is the same, but resume and Wi-Fi roaming need real-world testing.
6. **Neighbor table accuracy** for mac targets depends on the device having recently sent traffic; the targeted probe mitigates this but is not guaranteed on networks that block it.
7. **Shared CA key distribution** puts the signing key on every machine. The alternative of one signing machine reduces exposure but makes that machine necessary for rotation.
8. **Failover time for clients** is set by their resolvers. A floating virtual DNS address would make it seamless at the cost of another moving part.
