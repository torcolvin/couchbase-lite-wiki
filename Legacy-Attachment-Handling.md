Couchbase Lite 2 changes the way attachments (now called "blobs") are represented in documents. They used to be stored in a top-level property named `_attachments`; now they can be stored anywhere in the document, in objects with a property `"cbltype":"blob"`.

This causes some compatibility issues in several situations:

* Imported CBL 1.x databases containing attachments
* Replication with Sync Gateway (which currently does not know about the new schema)
* Documents replicated via Sync Gateway with other databases like CBL 1.x, CouchDB and PouchDB

Here's how we address this.

## Existing documents with `_attachments`

If a document contains an `_attachments` property whose value is an object (map/dictionary), any nested objects within that are considered to be blobs.

This rule needs to be honored by Couchbase Lite's sub-document API, as well as LiteCore code that detects attachments in documents (i.e. to set the internal `kC4DocHasAttachment` flag.)

## Pushing To Sync Gateway

When pushing a document that contains blobs to Sync Gateway:

* The LiteCore replicator will synthesize an `_attachments` property in the JSON that it sends, containing an entry for each blob found in the document. (If this property already exists, the entries will be added to it.)
* The name (key) of each attachment will be the dot-delimited JSON path of the blob, prefixed with a `$`.
* The blob will not be removed from its original location.

### Example

A CBL document that looks like this:
```js
{ name: "Widget 124C41+",
  photos: {
    thumbnail: {
      _cbltype: "blob",
      digest: "xxxxxxxxxx",
      type: "image/jpeg",
      length: 4321 }
  }
} 
```
will be sent to Sync Gateway in this form:
```js
{ name: "Widget 124C41+",
  photos: {
    thumbnail: {
      _cbltype: "blob",
      digest: "E3548AF60A3A407CA67389653ED82C09",
      type: "image/jpeg",
      length: 4321 }
  },
  _attachments: {
    "$.photos.thumbnail": {
      digest: "E3548AF60A3A407CA67389653ED82C09",
      type: "image/jpeg",
      length: 4321 },
  }
}
```
(Note: For clarity I'm using JSON5 syntax here, i.e. omitting quotes around most keys.)

## Pulling From Sync Gateway

When pulling from Sync Gateway, every incoming document needs to be checked for an `_attachments` property. If it contains one, the replicator will do this:

* For each attachment in `_attachments`:
    * Look through the document for a blob with a matching `digest` value. 
    * If found, remove the corresponding `_attachments` entry.
* If all `_attachments` entries are removed, remove the property itself too.

The usual cases are: receiving a document created by CBL 2 (the synthesized `_attachments` property will simply be removed), or receiving a document created by CBL 1 or a different database (the `_attachments` property will be left alone since it contains the real attachments.) But this logic should also work in less common cases like a document upgraded from CBL 1.x where blobs have been added but the `_attachments` property not yet deleted.