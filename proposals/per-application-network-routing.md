# Proposal: per-application network interface routing

**Status:** design proposal — no routing rules are installed by this document or by the current UI.

## Goal

Let a user choose a preferred physical network for an individual desktop application, independently of Plexo's existing per-download network selection.

Examples:

- A game prefers **Ethernet**.
- Chrome prefers **Wi-Fi**.
- Other applications continue to use the operating system's default route.

This is related to [issue #25](https://github.com/anmolkapil/plexo/issues/25), but it is a smaller and different first capability: **select one interface per application; do not aggregate bandwidth**. It should not be presented as a way to combine a game's or browser's traffic across multiple links.

## Why this needs a separate networking layer

Plexo's current network binding chooses an interface for sockets created by Plexo itself. A third-party application creates its own sockets and DNS traffic, so a preference stored in Plexo does not affect that traffic by itself.

This cannot be implemented reliably by adding a dropdown or by binding a source IP alone. It needs a platform backend that can steer traffic belonging to the selected process, plus explicit permission handling. We should not ship controls that look active but do not actually change routing.

## Proposed user experience

A future **Application routing** section could contain:

- **Add application** — browse to an executable or select a supported application identifier.
- **Preferred network** — choose from active interfaces already discovered by Plexo.
- **Policy when unavailable** — *Block traffic until this network returns* or *Use the system default route*. The choice must be explicit; a strict policy should never silently leak traffic to another interface.
- **Status** — *Active*, *Network unavailable*, *Permission required*, or *Unsupported on this OS*.
- **Remove rule** — remove only the routing policy created by Plexo.

Rules should survive restarts. A rule whose executable or interface cannot be resolved must be shown as inactive, not silently reassigned. Existing download network controls and their settings must remain unchanged.

## Platform feasibility and suggested rollout

The UI should depend on a typed backend contract and report real backend capability. Each platform must be evaluated separately.

### Windows

Use a narrowly scoped native backend for process-specific routing. Windows exposes per-socket interface options for applications that create their own sockets, but that does not automatically steer sockets created by an unrelated application. Firewall rules that merely block traffic on an interface are not equivalent to routing it through the requested interface.

The backend must define its privilege model, work with IPv4 and IPv6, and safely remove only the rules it owns. Do not depend on source-address binding alone when multiple default routes exist; see [issue #65](https://github.com/anmolkapil/plexo/issues/65) and Microsoft's [Winsock IP protocol socket options](https://learn.microsoft.com/en-us/windows/win32/winsock/ipproto-ip-socket-options).

### macOS

Investigate a Network Extension-based implementation and its signing, entitlement, user-approval, and distribution implications before exposing a supported control. Apple's [App Proxy Provider](https://developer.apple.com/documentation/networkextension/app-proxy-provider) and [Packet Tunnel Provider](https://developer.apple.com/documentation/networkextension/packet-tunnel-provider) documentation describes per-app flow and packet tunnelling approaches. This is not equivalent to setting a source address in a normal Electron process.

### Linux

Investigate process/cgroup-based policy routing with a dedicated routing table and firewall marks, or another maintained mechanism with equivalent guarantees. The backend must account for cgroup version, network-manager interaction, IPv4/IPv6 routes, privilege requirements, and cleanup after crashes or uninstall. It must not rewrite the system's default route.

### Rollout

Start with **one OS supported end-to-end**, chosen after maintainer review of implementation and packaging cost. Keep unsupported platforms read-only (with an explanation) until their backend is implemented and tested. Only add a cross-platform rule editor once there is at least one backend contract that can report verified enforcement state.

## Backend contract

Keep OS operations out of the renderer. The main process should call a typed service with operations conceptually equivalent to:

- list supported capabilities and current application rules;
- validate an application identity and preferred interface;
- apply, replace, or remove one rule;
- report enforcement status and a useful error;
- reconcile saved rules with the active interfaces at startup and after interface changes.

Persist only declarative preferences in Plexo settings. The backend should own and tag any OS objects it creates, so uninstalling or removing a rule cannot delete unrelated system configuration. Use structured process arguments rather than shell-built commands, validate every path/interface identifier in the main process, and never execute an application supplied by a URL or other untrusted input.

## Acceptance criteria

Before calling an application rule **Active**, tests should prove that its outbound traffic uses the intended interface rather than just a preferred source address.

- Test with Ethernet and Wi-Fi both connected and both having a default route.
- Verify IPv4, IPv6, DNS, TCP, and UDP where supported by the backend.
- Verify a second application remains on its own configured interface, and unconfigured applications remain unaffected.
- Disconnect and reconnect the selected network while the application is running.
- Restart Plexo and the target application; remove and recreate a rule.
- Verify denied permissions, unsupported systems, and a missing interface produce a clear status and clean rollback.
- Confirm no system-wide default route is changed and no unrelated firewall/routing rule is removed.

## Decisions needed before a full implementation

1. Which OS should be the first supported backend?
2. Is an opt-in privileged helper or system extension acceptable for packaging and user consent?
3. Should *Block traffic* be the default when the preferred interface disappears, or should users explicitly enable that strict mode?

The initial implementation should follow the answers above rather than add a UI that implies unsupported behavior.
