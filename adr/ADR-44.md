# Versioning for JetStream Assets

| Metadata | Value             |
|----------|-------------------|
| Date     | 2024-07-22        |
| Author   | @ripienaar        |
| Status   | Approved          |
| Tags     | jetstream, server |

# Context and Problem Statement

As development of the JetStream feature progress there is a complex relationship between connected-server, JetStream 
meta-leader and JetStream asset host versions that requires careful consideration to maintain compatibility.

 * The server a client connects to might be a Leafnode on a significantly older release and might not even have 
   JetStream enabled
 * The meta-leader could be on a version that does not support a feature and so would not forward the API fields it 
   is unaware of to the assigned servers
 * The server hosting an asset might not match the meta-leader and so can't honor the request for a certain 
   configuration and would silently drop fields

In general our stance is to insist on homogenous cluster versions in the general case but it's not reasonable to expect this
for leafnode servers nor is it possible during upgrading and other maintenance windows.

Our current approach is to check the connected-server version and project that it's representitive of the cluster as 
a whole but this is a known incorrect approach especially for Leafnodes as mentioned above.

We have considered approaches like fully versioning the API but this is unlikely to work for our case or be accepted 
by the team. Versioning the API would anyway not alleviate many of the problems encountered when upgrading and 
downgrading servers hosting long running assets. As a result we are evaluating some more unconventional approaches 
that should still improve our overall stance.

This ADR specifies a way for the servers to expose some versioning information to help clients improve the 
compatability story.

# Server-set metadata

In lieu of enabling fine grained API versioning we want to start thinking about asset versioning instead, the server
should report the version that created an asset and it should report the version that is hosting the asset and 
potentially expose a asset version.

We'll store this in the existing `metadata` field allowing us to expand this in time, today we propose the following:

| Name                           | Description                                      |
|--------------------------------|--------------------------------------------------|
| `_nats.server.version`         | The current server version hosting an asset      |
| `_nats.created.server.version` | The version server that first created this asset |
| `_nats.created.server.time`    | The time the asset was first created             |

We intend to store some client hints in here to help us track what client language and version created assets.

As some of these these are dynamic fields tools like Terraform and NACK will need to understand to ignore these fields 
when doing their remediation loops.

# Offline Assets

Today when an asset cannot be loaded it's simply not loaded. But to improve compatability, user reporting and 
discovery we want to support a mode where a stream is visible in Stream reports but marked as offline with a reason.

An offline stream should still be reporting, responding to info and more but no messages should be accepted into it 
and no messages should be delivered to any consumers, messages can't be deleted, configuration cannot be updated - 
it is offline in every way that would result in a change.  Likewise a compatible offline mode should exist for Consumers. 

The Stream and Consumer state should get new fields `Offline bool` and `OfflineReason string` that should be set for 
such assets.

When the server starts and determines it cannot start an asset for any reason it should set this mode and fields.

For starting incompatible streams in offline mode we would need to load the config in the current manner to figure out
which subjects Streams would listen on since even while Streams are offline we do need the protections of
overlapping subjects to be active to avoid issues later when the Stream can come online again.

# Safe unmarshalling of JSON data

The JetStream API and Meta-layer should start using the `DisallowUnknownFields` feature in the go json package and 
detect when asked to load incompatible assets or serve incompatible API calls and should error in the case of the 
API and start assets in Offline mode in the case of assets.

This will prevent assets inadvertently reverting some settings and changing behaviour during downgrades.

# Minimal supported server version for assets

When the server loads assets it should detect incompatible features using a combination of `DisallowUnknownFields` 
and comparing the server versions that created assets on a major.minor level only.

Incompatible assets should be loaded in Offline mode.

# Concerns

The problem with this approach is that we only know if something worked, or was compatible with the server, after the
asset was created. In cases where a feature adds new configuration fields this would be easily detected by the 
Marshaling but on more subtle features like previous changes to `DiscardNew` the client would need to verify the 
response to ensure the version requirements were met and then remove the bad asset, this is a pretty bad error 
handling scenario.

This can be slightly mitigated by including the current server version in the `$JS.API.INFO` response so at least 
the meta leader version is known - but in reality one cannot really tell a lot from this version since it is not 
guaranteed to be representative of the cluster and is particularly problematic in long running clients as the server
may have been upgraded since last `INFO` call.

An alternative would be that the server maintains an API compatability counter that increments when we add features.  
To use a feature the clients would send something that indicates the minimum required feature level and the server 
would fail to add it. While this would solve the problem it would be hugely problematic to make work in all the 
clients without bugs.

# Implementation Targets

For 2.11 we should start reporting the metadata and possibly support offline assets, we might only do the remaining
features outlined above in 2.12 timeline.