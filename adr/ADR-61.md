# Reconfiguring the JetStream meta peer set for disaster recovery

| Metadata | Value             |
|----------|-------------------|
| Date     | 2026-05-20        |
| Author   | @MauriceVanVeen   |
| Status   | Proposed          |
| Tags     | server, jetstream |

| Revision | Date       | Author          | Info           |
|----------|------------|-----------------|----------------|
| 1        | 2026-05-20 | @MauriceVanVeen | Initial design |

## Context and Problem Statement

The JetStream meta group is the Raft group that owns stream and consumer assignments
across a cluster. Like any Raft group, its quorum is computed from the configured peer
set, not from the set of currently live peers. A peer that is shut down and intended to
not come back, without being explicitly peer-removed still counts toward the quorum
requirement.

This creates a disaster recovery trap when a cluster is expanded and then shrunk
informally. Consider a 3-node cluster grown to 5 nodes for a migration or capacity test.
If the two added servers are later turned off without peer-removing them, the meta group
still believes its size is 5 and needs 3 votes for quorum. If a single one of the three
originally live servers then fails, only 2 peers remain reachable, the metagroup loses
quorum, and the meta layer becomes unavailable. With no quorum there is no leader to
process a peer-remove either, so the meta layer stalls.

The currently supported remedy is to bring every previously configured peer back online,
under the same server name, at the same time, so that quorum can be re-formed and a real
peer-remove (`$JS.API.SERVER.REMOVE`) can run. In real disaster recovery scenarios that is
frequently hard to achieve: hosts are gone, the previous server names are unknown, or the
lost peers are simply unrecoverable. An otherwise healthy deployment is left stuck.

This ADR introduces an explicitly unsafe operator-only API to forcibly rewrite the meta
group's peer set on the surviving servers, allowing them to re-form a working meta layer
without needing to resurrect the lost peers.

## Design

### Endpoint

A new subject is added:

```
$JS.API.SERVER.RECONFIGURE.<server_id>
```

The subscription for this subject lives only on the system account. The trailing
`<server_id>` token mirrors the server monitoring endpoints such as
`$SYS.REQ.SERVER.<id>.JSZ`: it is the id of the server the request is addressed to, the
same server id reported by `JSZ` and the other monitoring endpoints. Each member of the
meta group subscribes to the subject formed from its own server id, so a request is
delivered to and applied by exactly one named server.

A reconfigure request is therefore always targeted at a single server. An operator
reconfigures the surviving meta members by issuing one request per server, which is a
safer way to drive an unsafe operation: each server is named explicitly and reconfigured
on its own. There is no leader involved in this flow because the precondition for using
this API is precisely that no leader can be elected.

### Request

```json
{
  "servers": [
    {"peer": "<server_name>"},
    {"peer_id": "<server_peer_id>"}
  ]
}
```

`servers` is the complete new peer set, not a delta. Each entry identifies one peer using
either:

- `peer`: the human-readable server name.
- `peer_id`: the Raft peer id, which is the hash of the server name.

`peer_id` is derived from `peer`. When only `peer` is supplied, the server hashes it to
obtain the `peer_id`. When only `peer_id` is supplied, it is used as-is. When both are
supplied, the entry is rejected unless `hash(peer) == peer_id`. The request is also
rejected if two or more entries resolve to the same `peer_id`. Either identifier is
accepted because, during disaster recovery, operators may have ready access to one but not
the other depending on what tooling they're using or state they're looking at.

The supplied `servers` set must be a subset of the receiving server's current view of the
meta peer set. This API can only shrink the peer set by dropping lost peers; it can never
introduce a peer the server does not already know about. Requiring a subset keeps the
operation strictly a removal of unreachable members, which is the disaster recovery case
this API is for, and prevents an operator from accidentally grafting in unrelated or
mistyped peers and creating a divergent meta group.

### Discovering the current state

Before issuing any reconfigure request, an operator needs two pieces of information from
the surviving servers: the current configured peer set, and which server holds the most
up-to-date meta log.

The configured peer set is read from `JSZ`. Today `meta_cluster.replicas` is only
populated on the server that is the meta leader. In the disaster recovery scenario this
API targets there is, by definition, no meta leader, so that field is empty on exactly the
servers an operator needs to inspect. This ADR therefore requires `meta_cluster.replicas`
to be populated on every server, reflecting that server's own view of the configured peer
set, regardless of whether it is the leader. This lets an operator list the configured
peers from any surviving server and construct the new (subset) `servers` list.

Which server should be reconfigured is determined from `RAFTZ`. Per Raft semantics, the
peer with the most up-to-date log is the only one that can safely become leader without
losing committed entries. Before sending the per-server reconfigure request, an operator
should request `RAFTZ` from the surviving servers and target the server with the highest
`pterm`, or, when `pterm` is equal, the highest `pindex`, in that order. Reconfiguring
that server is what allows it to win the subsequent election and propagate its log, so the
remaining peers converge on the most complete log rather than truncating committed
entries.

### Server behavior

On receiving a valid request, a server:

1. Verifies the request was received on the system account.
2. Verifies that itself is included in the supplied `servers` list, matched by either
   `peer` or `peer_id`. If not, the request is rejected with an error. A server is not
   allowed to reconfigure itself out of the meta group via this API.
3. Verifies that the supplied `servers` set is a subset of its current view of the meta
   peer set. If the request names any peer the server does not currently know as a meta
   member, the request is rejected: this API may only remove peers, never add them.
4. Verifies that, from its own perspective, the meta group currently has no leader. If
   this server knows of a current meta leader, the request is rejected: a healthy meta
   layer must be reconfigured through the normal peer-remove path, not this API.
5. Verifies that this server is itself in a state where it would be allowed to transition
   to `Candidate` and start an election: it is a voting member of the meta group and is
   not currently within its election timeout.
6. If the supplied set is identical to its current view of the peer set, the request is
   acknowledged as a no-op, without logging or an advisory.
7. Otherwise, the server overwrites its local view of the meta group's peers with the
   supplied set, recomputes quorum from the new size, and resumes Raft. It logs a `WARN`
   and emits an advisory (see below) describing the new peer set, making clear that an
   unsafe reconfiguration has occurred.

Checks 4 and 5 are the safety gate for this API: together they require that the server
believes the meta layer is genuinely stuck, no leader is known and the server itself can
start an election, before it will rewrite the peer set.

Once a majority of the new peer set has applied the change, the meta group can elect a
leader and the meta layer becomes available again. Upon becoming leader, the server sends
the latest peer set to all other peers, allowing the servers to converge on the final
agreed-upon peer set.

### Advisory

Whenever a server actually applies a reconfiguration, it publishes an advisory in addition
to logging the `WARN`.

The advisory is published on:

```
subject: $JS.EVENT.ADVISORY.SERVER.RECONFIGURE
type:    io.nats.jetstream.advisory.v1.meta_reconfigure
```

The body reports the server id and name that applied the change, its previous view of the
peer set, the new peer set that was applied, and the client information of whoever issued
the request. The latter is carried in a `client` field of type `ClientInfo`.

Advisories are ordinary published messages and do not depend on the meta leader, so they
are delivered even while the meta layer has no quorum.

### Response

The response follows the standard JetStream API envelope, with `type`:

```
io.nats.jetstream.api.v1.meta_reconfigure_response
```

The body reports the server id and name that applied the change. Errors are returned using
the standard JS API error format.

## Consequences

- Operators gain a supported path out of a wedged meta layer without needing to resurrect
  dead hosts under their original names.
- The API is intentionally unsafe and allows for data loss depending on how it's used.
  Misuse, by invoking it against a healthy meta layer, or supplying inconsistent peer sets
  to different servers across the per-server requests, can split-brain the meta group. The
  release notes and operator documentation for this feature must clearly mark it as a
  last-resort disaster recovery tool.
- The endpoint is operator tooling. No NATS client library changes are required; it is
  expected to be invoked via the `nats` CLI or equivalent against the system account.
