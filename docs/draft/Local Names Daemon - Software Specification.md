# Local Names Daemon - Software Specification

Draft v0.1 | 27 October 2026 | Working name: devnamed (placeholder) | Status: for review

## 1. Purpose and Summary

Developers who run many apps on one workstation want friendly, stable hostnames (for example `myapp.test`, `api.myapp.test`, `feature-login.myapp.test`) that work from the workstation and from other devices on the LAN, without typing ports. Today this is achieved with hand-edited dnsmasq and Caddy configuration plus ad-hoc scripts. That approach breaks when the machine's IP changes, leaves stale entries when apps are removed, and has no single place that knows what is registered.

This document specifies a long-running background service (the daemon) for Arch Linux that owns the problem end to end. It keeps a local DNS server (dnsmasq) and a reverse proxy (Caddy) configured to match a registry of hostnames. It watches the network and re-points DNS when addresses change. It exposes a local control interface and a CLI so any script or tool can register or remove a hostname with one command. It audits itself continuously and repairs drift.

## 2. Goals and Non-Goals

**Goals**

- G1. One command publishes or removes a hostname for a local app, reachable by name from this machine, from other LAN devices, and from every other machine in the same peer group (7.16).
- G2. No manual action when a machine's IP address or network changes. With several machines, each machine's routes keep working when another machine goes offline (7.16).
- G3. A small, stable CLI and API that external scripts use to register names pointing at a host and port, and to remove them again, quickly and repeatedly, without the daemon knowing anything about the workflow behind them.
- G4. Self-healing: the daemon starts at boot and continuously ensures dnsmasq, Caddy and the host resolver are installed, configured, running and consistent with the registry.
- G5. Safe by default: LAN-only, no open resolver, no hijacking of public names, least privilege.
- G6. Transparent: one status view, structured logs, and a diagnostics command that explains any problem and how to fix it.

**Non-goals**

- NG1. Not a public or split-horizon DNS product; it never serves internet domains.
- NG2. Not a general API gateway: no authentication, caching, rate limiting or load balancing beyond what local development needs.
- NG3. Does not configure the router or other devices. Other devices must be told to use this host as their DNS server (router DHCP setting or manual).
- NG4. No Windows or macOS support in v1: Linux with systemd only.
- NG5. No large-scale or wide-area clustering: the peer group (7.16) targets 2 to 5 machines on one LAN segment; WAN links, NAT traversal and orchestration at scale are out of scope.
- NG6. Does not manage application processes and contains no workflow logic: scripts that create per-branch or per-project environments are outside this project and call its CLI and API (7.10).

## 3. Context and Assumptions

- Host: Arch Linux with systemd. Docker is in use on the same machine. NetworkManager is likely present but must not be required.
- Building blocks: dnsmasq (DNS) and Caddy (reverse proxy with an internal certificate authority). The daemon controls these; it does not reimplement them (an embedded-DNS alternative is listed in section 14).
- Zones: chosen by the user; none is built in. Several zones may be managed. Recommended zones live under `.test`, `.home.arpa` or `.lan`. `.local` is permitted but triggers warnings because of mDNS conflicts (Avahi, Apple devices, systemd-resolved).
- How an app runs is not the daemon's concern. It only maps a name to a host and port, whether the app is a local process, a container, a virtual machine or another machine, as long as that host and port are reachable.
- Addressing strategy: each zone is answered by a wildcard record pointing at one advertised address (the host's primary LAN address). On a single machine no per-route DNS entries exist: DNS changes only when the advertised address changes and the proxy picks the destination from the hostname. In a peer group (7.16) this is replaced by per-route records that point at the machine owning each route.
- Other LAN devices learn this host's address from router DHCP settings. If the host's address changes they keep using the old one until reconfigured. The daemon cannot fix that and must make it visible (7.3, 14).
- Integration boundary: workflow automation, for example a script that creates one subdomain per git worktree, is outside this project. The daemon provides the commands and API such scripts call (7.10).

## 4. Key Scenarios

- S1. First-time setup: the user installs the package and runs the bootstrap command. The daemon inspects the system, proposes changes to dnsmasq, Caddy and resolver configuration, applies them after confirmation, and starts at boot.
- S2. Static project: the user registers `myapp.test` pointing at `127.0.0.1:3000`. It resolves and loads over HTTPS from the laptop and from a phone on the same Wi-Fi.
- S3. Scripted registration: an external script (not part of this project) calls the CLI to register a name for a host and port, for example one subdomain per git worktree. The script can ask the daemon to allocate a free port and read it from the command output. The route is live within seconds.
- S4. Scripted removal: the script unregisters the name; if it never does, the route's binding (to a path, a process or a lease) lapses and the daemon removes the route automatically.
- S5. Network change: the machine moves from Wi-Fi to Ethernet, or DHCP hands out a new address. DNS follows within seconds with no user action, and a notification event is emitted.
- S6. Suspend and resume, or offline: after resume the daemon re-evaluates the network once links settle. While no LAN address exists, local development keeps working through loopback (7.3).
- S7. Faults: dnsmasq is killed, Caddy rejects a config, or the registry file is corrupted. The daemon restores service, rolls back bad changes, and reports what happened.
- S8. Diagnosis: the user runs the doctor command and gets a prioritized list of problems with suggested fixes.

## 5. Terminology

| Term | Meaning |
| --- | --- |
| Zone | A base domain managed by the daemon, for example `myapp.test`. All names beneath it are answered by DNS. |
| Route | A mapping from a hostname inside a zone to an upstream, with options. |
| Upstream | The destination of a route: scheme, host and port. |
| Owner | The identity that registered a route: uid, free-form label, and optional binding. |
| Binding | A condition tying a route's life to a process (PID), a filesystem path or a lease (TTL). |
| Advertised address | The IP address a zone's wildcard record answers with. |
| Desired state | Registry, configuration and discovered network facts: the single source of truth. |
| Applied state | What is actually rendered into dnsmasq, Caddy and the host resolver. |
| Reconciliation | The loop that converges applied state to desired state. |
| Managed artifact | A file, unit drop-in or resolver setting created and owned by the daemon, always marked as such. |

## 6. Architecture

### 6.1 Components

1. **Core and state manager**: owns the registry, serializes all mutations, persists state (7.13).
2. **Registry**: zones, routes, owners, bindings, port allocations.
3. **Network watcher**: subscribes to kernel netlink events (addresses, links, routes), systemd-logind sleep and resume signals, and optionally NetworkManager signals. A slow polling loop is a safety net.
4. **Address selector**: turns the set of interfaces and addresses into one advertised address per zone, using the configured policy.
5. **DNS manager**: renders the dnsmasq drop-in, validates it, applies it, and verifies DNS answers.
6. **Proxy manager**: renders Caddy route files, validates them, reloads Caddy, rolls back on failure, and verifies routing.
7. **Host resolver manager**: makes this machine resolve managed zones through the local dnsmasq without altering its global DNS.
8. **Port allocator**: hands out free ports from a configured range.
9. **Liveness monitor**: health probes, lease expiry, PID and path bindings, garbage collection.
10. **Reconciler**: the loop that diffs desired against applied state and repairs differences.
11. **Control server**: local API on a Unix socket.
12. **CLI client**: thin command-line front end to the control server.
13. **Service supervisor**: talks to systemd to check, start and restart dnsmasq and Caddy.
14. **Diagnostics and event bus**: structured logs, status, doctor checks, event stream. In a peer group a fifteenth component, the cluster manager (7.16), replicates routes and shared settings between machines.

### 6.2 Data flow

A client sends a register request. The control server authenticates the caller by socket credentials and validates input. The core records the route in the registry and persists it. The reconciler is triggered, renders Caddy configuration, validates and reloads it, verifies the route end to end, and answers the client. Independently, a network event reaches the network watcher, is debounced, passes through the address selector, and if the advertised address changed, the reconciler re-renders the dnsmasq drop-in and applies it. A periodic audit runs the same reconciler to catch drift not caused by events.

### 6.3 Process and privilege model

- One daemon process, started by systemd at boot, restartable at any time without losing registrations.
- Preferred model: the daemon runs as a dedicated unprivileged system user. It owns only its own directories: a dnsmasq drop-in directory and a Caddy sites directory, both read by the respective services. A narrowly scoped polkit rule lets it restart and reload only `dnsmasq.service` and `caddy.service`. It never needs a general root shell.
- Fallback model: run as root with a restrictive systemd sandbox (no network namespace changes, read-only system, explicit writable paths). The choice is an open decision (section 14).
- The Caddy internal CA private key remains owned by the Caddy service. The daemon never reads it.
- Existing connections survive daemon restarts because the daemon is not in the data path: dnsmasq and Caddy keep serving with the last applied configuration while the daemon is down.

## 7. Functional Requirements

### 7.1 Zones and routes (registry)

- FR-1. The daemon manages one or more zones from configuration. Zones can also be added and removed at runtime by an administrator.
- FR-2. A route has: hostname, upstream (scheme `http` or `https`, host, port), owner (uid, label), optional binding, optional flags (pinned, insecure upstream TLS, websocket pass-through default on), creation and update timestamps, and a state (healthy, unhealthy, pending, failed).
- FR-3. Hostnames must be lowercase RFC 1123 labels, at most 63 characters per label and 253 in total, and must sit inside a managed zone. Anything else is rejected with a specific error. The daemon never creates DNS overrides outside managed zones.
- FR-4. Name derivation: the daemon can turn arbitrary text such as a branch name (`feature/Login_Page`) into a valid label (`feature-login-page`): lowercase, replace invalid runs with a single hyphen, trim hyphens, truncate to 63 characters with a short stable hash suffix when truncated.
- FR-5. Hostname patterns: a registered name covers exactly that host. A route may be flagged as wildcard to also cover all deeper subdomains (for example `*.feature-login.myapp.test` for `api.` and `admin.` under one project). The most specific match wins.
- FR-6. Collisions: re-registering a name with the same owner updates it (idempotent). A different owner is rejected unless the caller has admin rights and passes an explicit override. Reserved names (such as the status and CA-download hosts in 7.14 and 7.6) cannot be registered.
- FR-7. A route with an unreachable upstream is kept, marked unhealthy, and served a friendly error page naming the hostname and upstream. It is not silently dropped.
- FR-8. Upstream safety: by default upstream hosts must be loopback or private-range addresses or resolvable local names. Link-local ranges (for example the cloud metadata address), multicast and public addresses are rejected unless an administrator enables remote upstreams for that route.

### 7.2 Registration lifecycle and bindings

- FR-9. Register, update, unregister, list, get and pin are available through the API and CLI. All are idempotent and safe to repeat.
- FR-10. Bindings: a route may be bound to (a) a lease with a TTL that the client must renew, (b) a PID, removed when the process exits, or (c) a filesystem path, removed when the path no longer exists. A route may have several bindings; it is removed when any one lapses, unless pinned. Pinned routes ignore bindings and expiry.
- FR-11. Garbage collection: unhealthy routes whose owner binding is gone, or that have been unhealthy longer than a configurable stale period (default 7 days) and are not pinned, are removed. Each removal is logged and emitted as an event.
- FR-12. Registration is synchronous by default: the call returns after the route is applied and verified, or fails with a reason, within a configurable timeout (default 10 seconds). An asynchronous mode returns immediately with a request identifier.
- FR-13. Bursts of changes (for example ten routes registered in a loop by a script) are coalesced into a single proxy reload. Maximum added latency from coalescing is 500 ms.

### 7.3 Network monitoring and address selection

- FR-14. The daemon detects within 1 second of occurrence: interface up and down, address added or removed, default route change. It also re-evaluates after system resume and after NetworkManager or systemd-networkd restarts.
- FR-15. Events are debounced: changes arriving within 2 seconds of each other are coalesced into one evaluation. A safety poll runs every 30 seconds to catch missed events.
- FR-16. Advertised address policy, in order: (1) an explicit address or interface set in configuration; (2) the primary IPv4 address of the interface carrying the default route, excluding virtual interface classes (container bridges, veth, libvirt, VPN tunnels); (3) the first private-range IPv4 address on a physical interface; (4) loopback.
- FR-17. Interface classes are configurable with include and exclude lists (names and glob patterns) with sensible defaults for Docker and libvirt bridges, WireGuard and OpenVPN tunnels.
- FR-18. If a VPN becomes the default route, the daemon continues to prefer the physical LAN interface per FR-16 rather than the tunnel address.
- FR-19. Offline mode: with no usable LAN address, the daemon advertises loopback so local development continues from this machine. The status view shows 'degraded: no LAN address', other devices cannot resolve the names, and the daemon returns to normal automatically when a LAN address appears.
- FR-20. On advertised address change the daemon updates DNS (7.4), emits an `address-changed` event with old and new values, and runs user hook scripts (7.12). It also reports, loudly in status and doctor, that other devices configured with the old address must be updated, and whether a DHCP reservation would prevent this.
- FR-21. IPv6: in v1 only A records are served. Queries for AAAA on managed zones receive an empty answer so clients do not prefer an address the proxy does not serve. Dual-stack is future work.
- FR-22. Multiple LAN interfaces: a single advertised address per zone in v1. Optional per-interface answers (split horizon) are future work.

### 7.4 DNS management (dnsmasq)

- FR-23. The daemon owns exactly one dnsmasq drop-in file per instance, clearly marked as generated. It never edits other dnsmasq files except during bootstrap (7.11), which shows the proposed change and keeps a backup.
- FR-24. The drop-in contains, per zone: a wildcard answer for the advertised address (replaced by per-route answers in a peer group, 7.16), a local-domain declaration so unknown names inside the zone get NXDOMAIN instead of leaking upstream, and AAAA suppression per FR-21.
- FR-25. Global dnsmasq settings required by the daemon (dynamic interface binding, drop-in directory inclusion, no use of the system resolver file, upstream servers, local-only service) are verified at startup and by every audit. Missing settings are reported; they are only fixed automatically if bootstrap has been run and the user opted in.
- FR-26. Apply procedure: render to a temporary file, run dnsmasq's own syntax check against the result, atomically replace the drop-in, restart dnsmasq (wildcard address lines are not re-read on a plain reload), then verify by querying the local server for a sample name in each zone and comparing the answer to the advertised address. On any failure, restore the previous drop-in, restart again, and report the error.
- FR-27. Restarts are skipped when the rendered content is byte-identical to what is applied. The expected restart blip is under one second; clients retry automatically.
- FR-28. Upstream resolvers are configurable: an explicit list (default two public resolvers) or 'inherit', which follows the DNS servers learned by the active network connection, useful on corporate or captive networks.
- FR-29. The daemon detects port 53 conflicts (for example the systemd-resolved stub listener or another resolver) and reports the owner of the conflicting socket with remediation steps.
- FR-30. DNS rebinding protection is kept enabled for upstream answers. Managed-zone answers are unaffected.

### 7.5 Reverse proxy management (Caddy)

- FR-31. Strategy: the daemon writes one small file per route into a dedicated sites directory that the user's main Caddy configuration imports. This coexists with hand-written Caddy configuration. Bootstrap adds the import line and the global options needed (internal certificates for managed zones).
- FR-32. Apply procedure: render all route files into a staging directory, validate each route individually and then the whole configuration with Caddy's own validation, atomically swap the directory contents, reload Caddy through its local admin interface, and verify each changed route by sending a request with the route's Host header to the local listener. Existing connections must not be dropped.
- FR-33. Isolation: if one route fails validation it is quarantined and marked failed with the error text, and all other routes are applied normally.
- FR-34. Rollback: if the combined reload fails, the previous known-good directory contents are restored and reloaded, and the new routes are marked failed.
- FR-35. Per-route behavior: reverse proxy to the upstream, pass the original Host header by default (option to rewrite), WebSocket and streaming pass-through enabled, HTTPS upstreams supported with an option to skip verification for self-signed development certificates, and forwarding headers set.
- FR-36. Unknown hostnames inside a managed zone return a friendly 'no such route' page listing nothing sensitive. Unhealthy routes return the page in FR-7.
- FR-37. Caddy admin interface must stay bound to loopback or a Unix socket. The daemon verifies this and reports otherwise.
- FR-38. If the admin interface is unavailable, the daemon falls back to restarting the Caddy unit and reports the loss of zero-downtime reloads.

### 7.6 TLS

- FR-39. Modes per zone: `internal` (Caddy's local CA issues certificates, default), `http-only` (no TLS, for simple cases), or `custom` (the user supplies certificate files, for example from mkcert).
- FR-40. Certificate strategy for `internal`: either per-host certificates issued on first request, or a single wildcard certificate per zone (configurable) to avoid first-request latency across many routes.
- FR-41. Trust tooling: the daemon can print the root certificate path and fingerprint, install it into the system trust store on request (with confirmation), and report which browser stores (Firefox and Chromium NSS databases) are not yet trusting it.
- FR-42. Mobile devices: a reserved `ca.<zone>` host serves the public root certificate over HTTP on the LAN, and the CLI can display a QR code for that address, to ease installing trust on phones. Only the public certificate is ever served.

### 7.7 Host resolver integration

- FR-43. The daemon makes this machine resolve managed zones via the local dnsmasq. With systemd-resolved it sets per-domain routing for each zone (routing-only domains) toward the local dnsmasq address, without changing global DNS. It re-applies the setting when systemd-resolved restarts or when links change.
- FR-44. Without systemd-resolved, the daemon reports how to configure resolution and only edits the system resolver file if the administrator opts in.
- FR-45. For `.local` zones the daemon checks the mDNS interceptors (Avahi name-service entries, resolved multicast setting) and reports exactly which ones will shadow the zone.

### 7.8 Port allocation

- FR-46. A register request may ask the daemon to allocate a port. The allocator picks from a configurable range (default 3000-3999 for apps, with a configurable reserved list for well-known tools), preferring a stable per-name port derived from the hostname, probing forward on conflict.
- FR-47. A port is considered free only if no process listens on it and no other route holds it. The allocation is persisted and released on unregister.
- FR-48. Allocation reserves the number; it cannot force the app to bind it. The response states this, and the route stays pending-unhealthy until the upstream actually accepts connections.
- FR-49. A caller may also supply an explicit port. Conflicts with another live route are rejected.

### 7.9 Reconciliation and self-healing

- FR-50. A single reconciler loop (single writer) converges applied state to desired state. It is idempotent and driven by events (route change, network change, service state change, configuration change, peer change) plus a full audit every 60 seconds.
- FR-51. Each pass: verify dnsmasq and Caddy are installed and active (start them if inactive and bootstrap permits), verify managed files match rendered content by hash, verify listeners on ports 53, 80 and 443 belong to the expected services, verify host resolver settings, probe routes, and fix or report differences.
- FR-52. Failures back off exponentially (1 s doubling to a 60 s cap) per subsystem, so one failing subsystem does not block others.
- FR-53. The reconciler never applies partial output: every external change follows render, validate, swap, apply, verify, with rollback.
- FR-54. Startup: acquire an exclusive instance lock, load state, drop expired bindings, start serving the control socket immediately (status shows 'reconciling'), then run a full reconciliation in the background.
- FR-55. The daemon watches the state of the dnsmasq and Caddy units; when either restarts or dies, it re-verifies and re-applies within 5 seconds.

### 7.10 Scripting and integration surface

- FR-56. Scope: this project provides commands and an API. It contains no workflow logic for any particular tool. Scripts that create per-branch or per-project environments (for example one subdomain per git worktree) are written and owned outside this project and call the CLI described in 7.14. When the machine's address changes, the daemon updates DNS by itself (7.3); scripts never need to.
- FR-57. Everything a script needs is available through the CLI with stable, documented behavior: register (hostname, host, and port or an allocated port), unregister, list and get, renew, pin and unpin, port allocate and release, status and doctor, events, and a helper that turns arbitrary text such as a branch name into a valid hostname label (FR-4).
- FR-58. Machine-friendly behavior: every command can print JSON, writes results to standard output and diagnostics to standard error, and uses stable exit codes (FR-78). Commands are idempotent, so a script can run them repeatedly without checking state first.
- FR-59. Lifetime binding: a script can tie a route's life to a process (PID), a filesystem path or a lease (FR-10), so routes disappear on their own when the script fails to clean up.
- FR-60. Port allocation is available as a standalone command so a script can obtain a free port before starting its app, and as an option on register (7.8).

### 7.11 Service lifecycle, bootstrap and supervision

- FR-61. The daemon ships a systemd unit that starts at boot independent of network availability, restarts on failure, logs to the journal, and carries sandboxing options consistent with 6.3. It handles SIGTERM with graceful shutdown (finish current apply, flush state) and SIGHUP by reloading configuration.
- FR-62. The daemon ensures dnsmasq and Caddy are enabled and running, via systemd, and may install drop-ins for them (for example restart-on-failure policy and ordering) when bootstrap has been run.
- FR-63. Bootstrap is an interactive, idempotent command (with a non-interactive mode). It inspects and reports: installed packages, conflicting services, existing dnsmasq and Caddy configuration, resolver stack, firewall tooling, interface candidates. It shows a diff for every file it would change, backs up originals, applies on confirmation, and records exactly what it changed so uninstall can reverse it.
- FR-64. Uninstall or `teardown` removes all managed artifacts and restores backed-up files, leaving the user's own configuration intact.
- FR-65. Optional firewall integration (ufw, firewalld, nftables): never changes firewall rules silently. It can show the exact rules needed (DNS, HTTP, HTTPS and, in a peer group, the cluster port, from the LAN subnet only) and apply them on explicit request.

### 7.12 Configuration

- FR-66. A single human-edited configuration file in the system configuration directory, validated at load with precise error messages, hot-reloadable by SIGHUP or the reload command. Invalid configuration on reload is rejected and the running configuration is kept.
- FR-67. Precedence: built-in defaults, then configuration file, then per-zone settings, then per-route options. Command-line flags apply only to the invoked command. Settings marked cluster-wide are replicated to all peers (7.16) and edited on any one of them.
- FR-68. Key settings:

| Area | Settings |
| --- | --- |
| Zones | name, TLS mode, certificate strategy, wildcard default, advertised address override |
| Network | interface include and exclude, address policy, debounce interval, poll interval |
| DNS | upstream list or inherit, IPv6 policy, record TTL, extra local records (admin-only) |
| Proxy | admin interface address, sites directory, default error pages |
| Ports | allocation range, reserved list |
| Lifecycle | default lease TTL, stale period, health probe interval and timeout, apply timeout |
| Access | socket group, admin group, allowed zones per group |
| Cluster | node name, cluster port, heartbeat interval, owner-offline policy and grace period, replication tombstone retention |
| Hooks | directory of executable scripts run on events, with timeout |
| Logging | level, format |

- FR-69. Hooks: executables in the hook directory are run on selected events (address changed, route added or removed, degraded entered or left, peer up or down) with event details passed through environment variables, a 10-second timeout, and output captured to the log. A failing hook never affects reconciliation.

### 7.13 Persistence

- FR-70. The registry and allocations persist in a single state store under the daemon's state directory. Writes are crash-safe (transactional database or write-to-temp, flush, atomic rename) and carry a schema version with forward migrations.
- FR-71. The last 5 known-good states are retained as backups. A state store that fails to load is moved aside with a clear log entry and the daemon starts with an empty registry; configuration-pinned items are rebuilt from the configuration file and, in a peer group, routes owned by other machines are recovered from peers (7.16).
- FR-72. Rendered artifacts are always reproducible from state plus configuration, so losing them is harmless.
- FR-73. Only one daemon instance may run per machine: enforced with an exclusive lock; a second start exits with a clear message.

### 7.14 Control interface (API and CLI)

- FR-74. Transport: a local Unix socket with HTTP and JSON semantics, versioned under a `v1` path prefix, usable from shell scripts with standard tools. A server-sent event stream provides live events.
- FR-75. Authorization uses the kernel-provided peer credentials of the socket connection. Members of the socket group may register, update and remove routes in allowed zones and see everything. Administrators (root or admin group) may manage zones, overrides, configuration reload, bootstrap, teardown and cluster membership. Every mutating call is recorded in an audit log with uid, pid and command line.
- FR-76. Operations: register or update route; unregister; list with filters (zone, owner, node, state, binding); get; pin and unpin; renew lease; allocate port and release port; list and manage zones (admin); force reconcile; status; doctor; configuration check and reload (admin); cluster operations (MM-15); event stream; version and capability discovery.
- FR-77. Responses use stable machine-readable error codes (for example invalid-hostname, name-taken, zone-not-managed, upstream-forbidden, apply-failed, timeout, unauthorized, peer-unreachable) plus a human-readable message and remediation hint.
- FR-78. The CLI mirrors the API: register, unregister, list, get, pin, renew, port, zones, cluster (init, join, leave, remove, status, nodes, sync), status, doctor, events (follow), reload, bootstrap, teardown, trust (show, install, qr), and completion generation. Output is human-readable by default and JSON with a flag; exit codes distinguish success, user error, daemon unreachable, timeout and partial failure.
- FR-79. If the daemon is unreachable, CLI commands fail fast with a clear message and, where meaningful, an option to wait for the daemon to become ready (useful in boot-time scripts).
- FR-80. The API is backward compatible within a major version; unknown request fields are ignored, and the capability endpoint reports supported features.

### 7.15 Observability and diagnostics

- FR-81. Structured logs to the journal with consistent fields (subsystem, route, zone, node, request identifier, result, duration). Levels configurable at runtime.
- FR-82. The status view shows: daemon state (ready, reconciling, degraded), advertised address and why it was chosen, each zone's DNS and TLS state, dnsmasq and Caddy unit state and last apply result, host resolver state, counts of routes by state, last network event, peer states and replication lag, and warnings.
- FR-83. Doctor checks (each pass, warn or fail, with a fix hint): packages present; units enabled and active; ports 53, 80 and 443 owned by the expected services; dnsmasq drop-in matches state; dnsmasq answers for each zone with the expected address; Caddy admin interface bound locally; Caddy config valid; every route resolves and responds; host resolver routes zones correctly; no mDNS shadowing for `.local` zones; root certificate trusted in system and browser stores; firewall allows LAN access and does not expose DNS beyond the LAN; advertised address is stable (DHCP reservation advice); clock sanity for certificate validity; in a peer group, every peer is reachable, every peer's DNS answers identically, replication is caught up, and clients are offered at least two DNS servers.
- FR-84. Event types: route-added, route-updated, route-removed, route-health-changed, address-changed, degraded-entered, degraded-left, apply-failed, rolled-back, service-restarted, gc-removed, peer-up, peer-down, route-conflict, owner-offline, owner-online.
- FR-85. A built-in status page, served at a reserved `status.<zone>` host on the LAN, shows the registry and health read-only, including which machine owns each route. It can be disabled.

### 7.16 Multi-machine operation (peer group)

Two or more machines (for example A and B) each run the daemon, dnsmasq and Caddy and form a peer group. A route registered on any machine works from every machine and from other LAN devices, and a machine's routes keep working while any other machine, including the one set up first, is offline.

Worked example: machine B registers `app.myapp.test` for an app running on B. B's daemon records the route as owned by B and replicates it to A. Both machines' DNS servers now answer that name with B's address, and B's Caddy serves it. If A is switched off, B still holds the complete record set in its own DNS server and still serves the route. Clients keep working because they were given both machines as DNS servers (MM-9).

**Design principles**

- P1. Peers, not primary and replica. Every machine runs the full stack and can serve DNS and proxy on its own. No machine is required for another machine's routes to work.
- P2. Single writer per route. Each route has exactly one owner machine, the one whose app it fronts. Only the owner changes it, so replication needs no conflict resolution beyond name collisions (MM-11).
- P3. Availability over consistency. A two-machine group cannot use a quorum-based store (Raft, etcd, Consul): when one machine dies the survivor has no majority and refuses writes, which is exactly the case this feature exists for. State converges eventually and every machine keeps accepting changes while disconnected.
- P4. The data plane does not depend on the control plane. DNS answers and proxying use each machine's local copy of the state; replication is only needed to change state.

**Requirements**

- MM-1. Membership and pairing: `cluster init` on the first machine creates the group and prints a single-use join token valid for 10 minutes. `cluster join` on another machine redeems it. Both machines exchange identity keys and record each other. The data model includes a node identity and an owner field on every route from the first release, so a single machine is simply a group of one.
- MM-2. Peer channel: daemons talk over a dedicated cluster port using mutual TLS with the identity keys exchanged at pairing. The port listens only on configured LAN interfaces (or on a private overlay such as WireGuard when configured). It is separate from the local control socket. `cluster remove` revokes a machine.
- MM-3. Liveness: peers exchange heartbeats every 2 seconds; a peer is marked unreachable after 3 missed heartbeats (6 seconds). Peers announce their advertised address whenever it changes, so an address change on B reaches A within seconds. If an announcement is missed, peers fall back to mDNS discovery or configured hostnames.
- MM-4. Replicated data: nodes (name, advertised address, status), routes (including owner), cluster-wide zone and policy settings, and the shared CA material (MM-12). Not replicated: port allocations, local bindings' process details, host resolver state and anything specific to one machine.
- MM-5. Replication: each owner keeps a per-owner sequence number on its route changes. Peers push changes immediately (typical propagation under 1 second) and run anti-entropy every 30 seconds by comparing the highest sequence number held per owner and pulling anything missing. Deletions are kept as tombstones for 7 days, then compacted; a machine away longer than that takes a full snapshot on return.
- MM-6. Ownership: a route registered on B is owned by B and its upstream defaults to B's loopback. The CLI can register a route on behalf of another machine (`--node`); the request is forwarded to that machine, which becomes the owner. Routes owned by a machine that is removed from the group are either deleted or adopted by another machine, as the administrator chooses.
- MM-7. DNS in a peer group: each route is answered with the advertised address of the machine that should serve it (by default its owner). Exact-name records are written to a hosts-style file that dnsmasq re-reads on a signal, so ordinary route changes need no restart; wildcard routes use address lines and still need a coalesced restart. An optional per-machine subzone mode (for example `*.b.myapp.test`) answers a wildcard with that machine's address so route changes never touch DNS at all, at the cost of machine names appearing in hostnames. Record TTL is short (default 30 seconds) so address changes are picked up quickly.
- MM-8. Proxy in a peer group: each machine's Caddy serves the routes it owns. An optional `mirror` policy also loads routes owned by others and forwards to them, so any machine's address works for any name; it is off by default because it makes a route depend on the forwarding machine.
- MM-9. DNS availability: every machine answers the full set of routes from its local replica. Clients should be offered at least two DNS servers (the machines themselves) through the router's DHCP settings, so name resolution survives a single machine failing; clients switch after their resolver timeout, typically 1 to 5 seconds. Doctor verifies that each machine answers identically and warns when only one DNS server is offered. Later versions may manage a floating virtual address (VRRP) so devices need only one DNS address and failover takes a few seconds.
- MM-10. Owner offline policy, per zone: `keep` (default) leaves DNS pointing at the unreachable owner so brief outages do not cause flapping and clients see a plain connection failure; `fallback` points the names at a surviving machine after a grace period (default 60 seconds), whose proxy shows a friendly 'machine B is offline' page, and restores the records when the owner returns. A route is never reassigned to another machine automatically.
- MM-11. Name collisions: when peers are reachable, registration checks the whole group and fails with `name-taken`. When machines are disconnected, both may accept the same name. On reconnect the earliest registration (by hybrid logical clock, ties broken by machine identity) wins; the other route is marked conflicted, is not answered by DNS, and raises a `route-conflict` event and a notification so its owner can rename it. Reserved names apply group-wide.
- MM-12. Shared certificate authority: the group uses one root CA so each device trusts a single certificate. The first machine creates or imports it and distributes the signing material to peers over the authenticated channel; it is stored readable only by the Caddy service. Each machine's Caddy issues leaf certificates for the names it serves, so issuance works while disconnected. A documented rotation procedure covers replacing the root and intermediates.
- MM-13. Clocks: ordering uses hybrid logical clocks; the daemon warns when wall-clock skew between peers exceeds 5 seconds, since it affects certificate validity.
- MM-14. Partitions: every machine keeps serving, and keeps accepting registrations, while disconnected. The status view shows 'disconnected from B since T'. Changes made on each side replicate on reconnection, with conflicts handled per MM-11.
- MM-15. API and CLI additions: cluster status, node list, join, leave, remove, forced sync; route listings show the owner machine; register accepts a target machine. Operations on a route owned by another machine are forwarded to its owner, which stays the only writer.
- MM-16. Security: only paired machines are trusted. Every replicated record is re-validated on receipt with the same rules as a local registration (hostname inside a managed zone, permitted upstream, reserved names), and a peer may only change routes it owns. Join tokens are single-use and expire after 10 minutes. The cluster port never listens on non-LAN interfaces.
- MM-17. Scope: the first release targets 2 to 5 machines on one LAN segment. WAN operation and NAT traversal are out of scope; machines on different networks can be joined over a private overlay network.

## 8. Security Requirements

- SEC-1. No open resolver: dnsmasq must answer only LAN clients. The daemon verifies listening scope and flags any interface with a public address or reachable from outside, and never listens for DNS beyond configured interfaces.
- SEC-2. Hijack prevention: routes and DNS entries exist only inside managed zones. Public names such as well-known domains cannot be overridden through the normal API.
- SEC-3. Input hardening: all names, ports, paths and labels pass strict allow-list validation before touching any generated configuration. No value is ever concatenated into a configuration file without validation and escaping. Fuzz tests are mandatory (section 12).
- SEC-4. SSRF containment: upstream restrictions in FR-8, enforced at register time and re-checked at render time.
- SEC-5. The control socket is mode 0660 owned by the daemon user and socket group; it is never exposed over TCP. Caddy's admin interface is never reachable from the LAN.
- SEC-6. State, configuration and generated files have least-privilege permissions. The CA private key is never read or logged by the daemon, except to transfer it to paired peers over the authenticated channel (MM-12); only the public root certificate is exported to devices.
- SEC-7. Rate limiting and size limits on the control interface (request size, requests per second per uid) to prevent a buggy script from flooding reconciliation.
- SEC-8. DNS rebinding awareness: documentation and doctor explain that internal names resolve to a private address by design, and that dev servers should keep host-header allow lists.
- SEC-9. Hooks run as the daemon user with a clean environment, timeouts, and never as a side effect of untrusted input.
- SEC-10. Audit trail of all mutations (FR-75), retained according to journal policy.
- SEC-11. Peer trust: mutual TLS with pinned identities, single-use short-lived join tokens, receiver-side validation of every replicated record, and ownership enforcement (MM-16). Compromise of one machine exposes only what that machine's key could already do: its own routes and the shared CA, so the CA key must be treated as sensitive on every machine.

## 9. Failure Modes and Expected Behavior

| Situation | Expected behavior |
| --- | --- |
| dnsmasq or Caddy not installed | Status degraded; doctor names the package; API still accepts registrations and applies them once installed |
| Port 53 or 80/443 held by another process | Apply fails with the owning process identified; last good configuration keeps running; retry with backoff |
| Invalid rendered configuration | Swap never happens; previous files stay; the offending route is quarantined with error text |
| Caddy reload fails | Roll back to the previous directory, reload, mark new routes failed, emit rolled-back event |
| dnsmasq restart fails | Restore previous drop-in, restart, verify, report |
| No network | Advertise loopback, degraded status, recover automatically |
| Address flapping | Debounce and a minimum interval between DNS applies (default 5 seconds); log a flapping warning |
| Daemon crash or kill | dnsmasq and Caddy continue with last configuration; systemd restarts the daemon; startup reconciliation converges |
| State store corrupted | Move aside, start empty, log, restore pinned routes from configuration and other machines' routes from peers |
| Second daemon instance | Exits immediately with a clear message |
| Upstream app stopped | Route kept, marked unhealthy, friendly error page, GC after stale period |
| systemd-resolved restarts | Per-domain routing re-applied within 5 seconds |
| Disk full or read-only filesystem | Reject mutations with a clear error, keep serving, doctor warns |
| System clock jump | Leases use monotonic time; certificate-validity warning in doctor |
| Docker or VPN creates interfaces | Ignored by classification rules unless explicitly included |
| A peer machine goes offline | Other machines keep serving their own routes and DNS from local replicas; peer-down event; owner-offline policy applies to the missing machine's routes |
| Machines cannot see each other (partition) | Both sides keep serving and accepting registrations; reconciled on reconnection; conflicts flagged |
| A peer's address changes | Peer announces it; DNS records on other machines update within seconds; if the announcement is missed, discovery fallback or anti-entropy repairs it |
| Peer returns after a long absence | Snapshot resync if beyond tombstone retention; otherwise incremental sync |
| Clock skew between machines | Warning; ordering still safe through hybrid logical clocks |
| Join token expired or reused | Pairing refused with a clear message; a new token can be issued |
| Shared CA material out of sync | Doctor flags which machine differs; resync over the peer channel |

## 10. Non-Functional Requirements

- NFR-1. Latency: a single route registration is resolvable and served within 2 seconds at the 95th percentile (excluding first-time certificate issuance). An advertised-address change is reflected in DNS within 5 seconds of the network event after debounce.
- NFR-2. Scale: at least 500 routes without degraded behavior; configuration render and validation under 1 second at that size.
- NFR-3. Resources: idle CPU near zero; resident memory under 50 MB; no busy polling.
- NFR-4. Availability: a daemon restart must not interrupt in-flight requests or DNS answers; Caddy reloads must be graceful.
- NFR-5. Determinism: identical inputs produce byte-identical generated files (stable ordering), so change detection by hash is reliable.
- NFR-6. Portability within scope: works with NetworkManager, systemd-networkd or manual network configuration; works with or without systemd-resolved.
- NFR-7. Maintainability: subsystems communicate through narrow interfaces so DNS or proxy backends can be swapped later; implementation language is chosen for single static binary, good netlink and systemd D-Bus libraries, and safe concurrency.
- NFR-8. Documentation: manual pages for the daemon, CLI and configuration; a troubleshooting guide derived from the doctor checks.
- NFR-9. Peer group: a route registered on one machine is resolvable from every reachable machine within 1 second typically and 5 seconds at the 99th percentile. No machine is a single point of failure for another machine's routes. After a machine fails, clients with two DNS servers recover name resolution within their resolver timeout.

## 11. Packaging and Installation

- Arch package with a PKGBUILD (AUR-ready): binary, systemd unit, sysusers entry for the service user and groups, tmpfiles entry for directories, polkit rule restricting unit control, default configuration, shell completions (bash, zsh, fish), manual pages.
- Hard dependencies: dnsmasq, caddy. Optional dependencies: nss tools (browser trust checks), a QR rendering library, a firewall tool, a VRRP daemon (only if the later floating-address feature is adopted).
- Post-install message directs the user to run bootstrap; nothing is reconfigured automatically on package install. Joining a peer group is a separate explicit step.
- Package removal leaves user data and generated files alone unless teardown is run first; the package documents this.
- Upgrades are non-destructive: state migrations run automatically with a backup taken first. Peers on different minor versions interoperate; the replication protocol is versioned and a machine refuses peers with an incompatible major version, with a clear message.

## 12. Testing and Acceptance

**Strategy**

- Unit tests: hostname validation and slug derivation, collision rules, binding expiry, port allocation, address selection policies, debouncing, renderers (golden files), state migrations, replication merge logic (ordering, tombstones, conflict resolution).
- Integration tests in isolated network namespaces with dummy interfaces: add, remove and change addresses; default route and VPN simulations; suspend and resume simulation; verify DNS answers and proxy routing with real dnsmasq and Caddy against fake upstream servers.
- Multi-machine tests: two or three simulated machines in separate namespaces; pairing; replication; partitions and heals; killing machines; address changes; clock skew; conflicting registrations during a partition.
- Fault injection: kill dnsmasq and Caddy, corrupt state, block ports, feed invalid configuration, fill the disk, restart systemd-resolved, flap interfaces.
- Security tests: fuzzing of every API and replication input and rendered output, permission checks on the socket and files, attempts to register names outside zones, forbidden upstreams, and routes owned by another machine.
- Soak test: 24 hours with churning registrations, network flaps and peer restarts without leaks, drift or unexpected restarts.

**Acceptance criteria**

- AC-1. On a clean Arch install, bootstrap followed by one register command makes a name resolve and load over HTTPS from the host and from a second LAN device.
- AC-2. Changing the host's IP (DHCP renewal, switching Wi-Fi to Ethernet) updates DNS with no user action in under 5 seconds after the event.
- AC-3. Registering 10 routes in a loop from a script produces 10 working names with no port conflicts and a single coalesced proxy reload.
- AC-4. When the path a route is bound to disappears, with no unregister call, the route is removed within one audit cycle.
- AC-5. Killing dnsmasq, Caddy or the daemon itself restores full service within 10 seconds without user action and without losing registrations.
- AC-6. An invalid route never breaks any other route, and a failed apply always leaves the previous working configuration active.
- AC-7. Attempts to register names outside managed zones, or upstreams in forbidden ranges, are rejected with the documented error codes.
- AC-8. Teardown returns the system to its pre-bootstrap configuration.
- AC-9. Doctor reports every induced fault from the failure table with a correct fix hint.
- AC-10. With machines A and B paired, a route registered on B resolves and loads from A, from B and from a third LAN device within 5 seconds.
- AC-11. With A powered off, B-owned routes still resolve (through B's DNS) and load for a client offered both DNS servers, and a new route can still be registered on B.
- AC-12. When A returns, it receives routes registered on B during the outage within 30 seconds without manual action.
- AC-13. Two machines that registered the same name while disconnected end up with one winner and one flagged conflict, with an event raised, and no manual intervention needed to restore a consistent state.
- AC-14. A single trust installation of the shared root certificate on a device makes HTTPS work for routes served by either machine.

## 13. Phasing

- **Milestone 1 - Foundation:** service skeleton, state store (with node identity and route owner fields from the start), configuration, bootstrap, DNS manager, address selector, reconciler with audit loop, status and doctor basics.
- **Milestone 2 - Routing:** control API and CLI, route registry, Caddy manager with validation and rollback, coalescing, synchronous registration, port allocator.
- **Milestone 3 - Resilience:** full network watcher (netlink, resume, NetworkManager), offline mode, host resolver integration, health probes, leases, garbage collection, service watching.
- **Milestone 4 - Scripting surface:** stable scripting surface (JSON output, exit codes, bindings), hooks, events stream.
- **Milestone 5 - Trust and polish:** TLS modes, trust tooling, CA download host and QR code, status page, firewall helper, packaging, manual pages, full test and soak suite.
- **Milestone 6 - Peer group:** pairing and peer channel, replication and anti-entropy, per-route DNS records with hosts-file reload, shared CA distribution, owner-offline policies, conflict handling, cluster doctor checks, multi-machine test suite. Optional later: floating virtual DNS address, per-machine subzones, mirror policy.

## 14. Open Questions and Future Work

1. **Language and runtime:** recommended direction is a single static binary in Go or Rust; final choice pending library availability for netlink, systemd D-Bus and Unix socket HTTP.
2. **Privilege model:** unprivileged user with polkit (preferred) versus sandboxed root. Needs a prototype to confirm everything required works without root.
3. **Embedded DNS instead of dnsmasq:** a small built-in DNS responder with forwarding would remove restarts for wildcard changes, allow per-interface answers, and remove one dependency, at the cost of owning DNS correctness and caching. The peer group makes this more attractive, since the daemon already holds every record.
4. **Caddy ownership:** drop-in directory (chosen for v1) versus the daemon owning the entire Caddy configuration through its API.
5. **State store format:** embedded transactional database (recommended) versus atomically written structured file.
6. **Replication implementation:** a purpose-built per-owner log with anti-entropy (described in MM-5, recommended given single-writer ownership) versus an existing library for gossip or conflict-free replicated data.
7. **Client failover:** two DNS servers offered by DHCP (simple, required) versus a floating virtual address between machines (seamless, extra moving part). Decide whether the floating address belongs in the first peer-group release.
8. **Shared CA distribution:** every machine holds the root signing key (simple, resilient, larger exposure) versus one machine holds the root and issues intermediates to the others (smaller exposure, but the root machine is needed to rotate).
9. **Owner-offline default:** `keep` (chosen as default) versus `fallback` with a friendly offline page. Confirm after real-world use.
10. **Stale devices after an address change:** a DHCP reservation per machine (recommended, documented) versus running a DHCP server on the machines. Decide whether DHCP mode belongs in scope.
11. **mDNS publication:** publishing names via multicast DNS would let `.local` zones work on phones and Apple devices without DNS configuration. Candidate for a later version.
12. **Split horizon and IPv6:** per-interface answers and AAAA support.
13. **Docker integration:** automatic route discovery from container labels, registering routes for containers as they start and stop.
14. **Naming:** final product name, socket path, cluster port and configuration paths.
