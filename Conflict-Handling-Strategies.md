A key part of a replication algorithm is how conflicts are handled. To recap: a **conflict** is two or more different **revisions** of a document that have a **common ancestor** revision (usually a common parent.) A conflict is created when two databases make different changes to the same revision of a document, without knowing about each other’s changes, and then later replication brings both of the conflicting revisions to the same database.

## 1. Overview
The main tasks in conflict handling are:

1. Detecting a conflict, either as:
	* Conflict _prevention_, before a revision is saved (in which case the save will fail), or
	* Conflict _detection_, when an existing revision is added to the database through replication
2. Resolving the conflict:
	1. Finding the common ancestor revision
	2. Retrieving the common ancestor’s body
	3. Determining the body of the merged revision (highly application-dependent, so not discussed here)
	4. Saving the merged revision such that the conflict is no longer active

The key part of any conflict handling system is a **revision ID** (**revID**). This is just a piece of document metadata maintained by the database, whose value changes every time the document is updated. It can be as simple as a counter or a timestamp. Ideally a revID should not be repeatable for the same document, so a digest alone doesn’t work. But different documents’ revIDs are independent.

### Conflict Prevention
Conflict prevention is worth discussing separately: it’s pretty straightforward, and works the same way regardless of what versioning system is in use. The database merely has to return the revID from a Read operation, along with the body, and require that any Update or Delete operation include that original revID. If the revID given with an Update/Delete doesn’t match the document’s current revID, there’s a conflict and the operation fails. Otherwise, the database saves the revision along with a newly-generated revID, and returns the new revID back to the application (in case it wants to update again.)

The application (actually the platform code’s `Document` class) handles a failed save by fetching the current revision from the database. It also keeps a copy of the original revision that’s been modified by the app code, and uses that as the common ancestor. It can then let the app code resolve the aborted conflict by merging those; then it retries with the new merged revision (whose base revID is now the newer one fetched from the server.)

## 2. Versioning Systems
There are many versioning systems that can enable conflict resolution. The three we’ve used or explored are **Revision Trees**, **Version Maps** (an extension of **Version Vectors**), and **Simple MVCC** (also known as **CAS**).

### Revision Trees
These are currently used in Couchbase Mobile 1.x (as inherited from CouchDB.) Every document’s metadata includes a tree of all known revisions, with each tree node having a revID and pointing to its parent. When a new revision is saved, its node is added to the tree as a leaf. 

If a tree has multiple leaves that aren’t **tombstones** (deletion markers), it’s in conflict. A conflict is resolved by saving the merged version as a child of one leaf, while deleting the other leaf by adding a tombstone to it.

This supports arbitrary replication topologies. In theory it provides perfect identification of common ancestors, but in practice the ever-growing tree metadata becomes a problem as it takes up server space and replication bandwidth, so it has to be **pruned** by removing the oldest revisions. This reduces its size but also reduces accuracy.

A complete revision tree can always identify the common ancestor of any two revisions. However, a pruned revision tree may not contain that ancestor anymore, and the two conflicts will reside in disjoint subtrees. Much worse, non-conflicting changes can be falsely identified as conflicts if the two peers’ histories have diverged long enough that their common ancestry is lost.

### Version Vectors
The classic Version Vector is a [well-known][1] data structure that stores a separate generation counter for each of a known set of peers. Each peer is assigned a position in the vector. The vector for a new empty document starts as all zeroes, and every time a peer updates the document, it increments the counter at its position.

Version vectors have an algebraic _partial ordering_ defined by comparing the counters at corresponding positions. Obviously if all corresponding counters in vectors A and B are equal, the vectors denote the same revision.  Otherwise, if all the counters of vector A are less than or equal to the corresponding counters of vector B, then A is an ancestor of B. Two vectors that can’t be ordered this way (neither is an ancestor of the other) are in conflict.

To resolve a conflict, you compare other available versions to find the nearest known common ancestor (i.e. the vector with the highest counts that’s an ancestor of both conflicting versions). The merged revision is assigned a vector where each component is the maximum of the corresponding components from the conflicting versions.

Version vectors are very widely used, but they work best with a smallish fixed set of peers. Too many peers bloats the metadata, and modifying the set requires a lot of bookkeeping and careful reconciliation: the structure of the vectors changes over time, and those changes have to be communicated to all the peers involved, while at the same time revisions are being created and replicated.

Version vectors are also less accurate than revision trees at identifying common ancestors; the pairwise-maximum merge of the vectors loses some information, so conflicts between two different previously-merged pairs of conflicts can result in ambiguity where multiple revisions have equal claim to being the closest ancestor. (I am not sure whether this turns out to be a big problem in practice.) On the plus side, version vectors don’t seem prone to false conflicts.

### Version Maps
The Version Map is a spin-off of the version vector. It was invented for Couchbase Mobile, but most of its ideas have appeared independently in various academic papers on versioning. The basic idea is to use a **map** (aka dictionary) instead of an array; every peer is assigned a unique key. This resolves the scalability problems of version vectors, because:

* entries with zero counts can safely be omitted, so the only entries appearing in the map are from peers that have modified that document;
* it’s easy to add new peers on the fly by just assigning them new unique IDs.

The downside is that each entry in a version map has a peer ID as well as a count, which makes it a lot larger, especially if the peer IDs are UUIDs (16 bytes) or digests (20+ bytes). However, we believe that in most distributed document systems no document is modified by a large number of peers, largely because this is already a good design practice for avoiding conflicts.

This system has been [designed][2] and mostly [implemented][3] (in LiteCore), but not deployed yet.

### Simple MVCC, or CAS
This widely-used system (as seen in Couchbase Server and WebDAV) is basically identical to Conflict Prevention as described above, but extended to a client-server architecture. The server maintains the revIDs. When a client attempts to push a new revision to the server, it has to provide the original revID, and if it no longer matches, the server rejects it.

To the server this is conflict _prevention_, but from the client’s perspective it’s conflict _resolution_, because the client has already saved a real revision (locally) before discovering that a conflict now exists on the server. But its resolution strategy is still identical to the one described in the Conflict Prevention section: it pulls the current server revision, fetches the original server revision from its local database, and lets the app code merge those together with the local current revision.

Simple MVCC is attractive because the document metadata is minimal (just a revID, which can be an integer, i.e. 2-4 bytes per document). On the other hand, the use cases for distributed systems are pretty limited, as it usually requires a client-server **star topology** where one peer (the server) is the “source of truth” for the system. It’s not possible for the clients to replicate changes between each other (at least, not changes that haven’t yet been pushed to the server), because false conflicts can easily result.

Another drawback is that conflicts must be resolved locally before being replicated, since the server will refuse to accept a conflict. This can be a problem when a conflict requires user intervention. Consider a case where a user has edited a wiki page while offline, and now wants to upload the change securely to a server (or just to work on it from another device.) If there’s a conflict, the app won’t be able to upload the change until the user does the work of comparing and merging the two versions of the text. This also prevents the user from being able to delegate the merge task to someone else like an editor.

It _is_ possible to extend this to a hierarchical system, where clients replicate with an **intermediate server**, and the intermediate servers replicate with higher-level servers, with (exactly) one topmost server as the Source Of Truth. This can be an attractive topology for load balancing, or for allowing a group of nearby clients to function with limited Internet connectivity. However, it requires that conflicts can be resolved on the intermediate servers, not just on the clients: this is because a server may accept a revision from a client, and then later have it rejected when it tries to push it up to its next-higher server. There’s no option to send the conflict back down to the client, so the server is responsible for working it out on its own. (This mostly rules out use cases where conflicts can require user intervention, e.g. editing wiki pages.)

[1]:	https://en.wikipedia.org/wiki/Version_vector
[2]:	https://github.com/couchbaselabs/couchbase-lite-api/wiki/Version-Vectors
[3]:	https://github.com/couchbase/couchbase-lite-core/tree/master/LiteCore/VersionVectors