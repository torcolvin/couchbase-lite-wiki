LiteCore's replicator needs to be able to authenticate to a server. There's quite a set of protocols for this; here's what we support in Couchbase Lite 1.x:

* HTTP Basic
* Sync Gateway (or CouchDB) session cookie
* OAuth 1
* OpenID Connect
* Facebook
* Persona [I think this is obsolete now?]
* TLS client certificate

The top-level question is whether to implement these in LiteCore, or in the multiple Couchbase Lite platform bindings.

**In LiteCore:**
* Single implementation
* Consistent behavior across platforms

**In platform code:**
* Access to higher-level libraries that can do more of the work (CFNetwork, java.net.*, etc.)
   * Less code to write
   * Fewer potential bugs (including security vulnerabilities)
* Better integration with platform credential storage (Keychain)
* Can reuse a lot of existing code from 1.x