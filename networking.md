# Networking and authoritative replication

Kiloria separates packet framing, request authority, state encoding, and
transport. `PacketCodec` bounds a wire frame to 1 MiB. It does not create
sockets or authenticate peers; a production adapter must establish a trusted
server connection before passing peer identity and authentication state to
`RequestRouter` or packets to `SnapshotReceiver`.

## Server invocation routes

`NetworkRequestCodec` frames a route name, request ID, and bounded payload.
`RequestRouter` dispatches decoded requests to registered server handlers,
checks the trusted session identity supplied by the transport, applies
per-client/per-route token-bucket limits, and suppresses recently repeated
request IDs. Call `forgetClient(clientId)` when the connection closes to release
its rate-limit and replay state. Identity and authentication are never read
from client-controlled payload bytes. Route handlers execute outside the
router lock and should schedule world changes on the authoritative game thread;
the router does not make `GameWorld` concurrently safe.

The built-in `registerObjectMoveRoute` requires both an authenticated session
and an application authorization callback for each object. It validates the
move payload, rejects displacement beyond a configurable distance, and applies
accepted intent to the server's `GameWorld`. Snapshots can then publish the
resulting position:

```cpp
kiloria::network::RequestRouter router;
kiloria::network::registerObjectMoveRoute(router, serverWorld,
    [](kiloria::network::ClientId client, const kiloria::GameObject& object) {
        return ownsObject(client, object.id());
    });

// After authenticating a peer, unwrap PacketCodec and pass the request
// payload using that peer's server-assigned ClientId.
const auto response = router.dispatch(clientId, true, requestPayload);
```

Request payloads are limited to 64 KiB and route names to 128 ASCII bytes. The
router keeps a bounded recent-ID window (256 per client by default), so IDs
outside that window become eligible again; this is duplicate suppression, not a
cryptographic anti-replay guarantee. Use fresh request IDs per invocation and a
new server-assigned client ID for each authenticated session. The example
`kiloria_server_requests` shows acceptance, duplicate rejection, and per-object
authorization.

## Transform replication

`SnapshotPublisher::makePacket(scene, tick, sequence)` captures entity IDs,
names, parent links, and transforms in a versioned little-endian replication
payload. The receiver validates the packet type and framing, rejects malformed
hierarchies, duplicate/out-of-window packet sequences, and stale simulation
ticks, then exposes the new snapshot atomically. Only after acceptance should a
client call `applyLatest(scene)`. Matching entity IDs retain client-local
components; disappeared entities lose their components. The server is
authoritative for the replicated entity transforms and hierarchy.

```cpp
auto wire = kiloria::network::SnapshotPublisher::makePacket(serverScene, tick, sequence);
transport.sendToClients(wire); // application-provided transport

if (receiver.accept(receivedBytes))
    receiver.applyLatest(clientScene);
```

Snapshots currently send the complete entity state each time. Delta compression,
interpolation, interest management, reliability, encryption, and socket
transports are separate future modules. Authentication is an adapter-provided
identity that the route layer enforces; the packet sequence window is
deduplication only and is not a security boundary. Components are not
replicated by this transform channel; use explicit server-owned component
protocols rather than sending arbitrary process memory.

Build `kiloria_replication` for snapshot synchronization and
`kiloria_server_requests` for authoritative request dispatch.

## Client-side visual interpolation

`SnapshotInterpolator` retains a bounded history of strictly increasing,
validated snapshots. `sample(targetTick)` interpolates position and scale with
linear interpolation, orientations with quaternion slerp, and Euler display
angles along their shortest wrapped path. `sampleDelayed()` targets the newest
server tick minus the configured interpolation delay. Between topology changes,
the earlier snapshot's entity set and parent links remain stable; new and
removed entities switch at the newer snapshot boundary. Sampling clamps to the
oldest/newest buffered state and never extrapolates. Render this pose through a
presentation scene; keep authoritative collision/gameplay state separate.

```cpp
if (receiver.accept(packetBytes))
    interpolation.push(*receiver.latest());
if (auto pose = interpolation.sampleDelayed())
    updateRemotePresentationScene(*pose);
```

Use a delay appropriate to expected packet jitter and snapshot cadence. Ticks
are application-defined simulation units, not seconds. The example
`kiloria_snapshot_interpolation` demonstrates interpolation at a fractional
tick between two server poses.
