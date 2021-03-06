# System Overview

The Universal Resolver (UR) is a name resolver that works with any decentralized
naming system.

Also see the [DIF IND Community Resolver](https://docs.google.com/document/d/1rEPRjmRCwhLEfW7Cdwf-aGYXACqK-IFhD9o8bXaT6H0/)
Google Doc for design goals and architectural thoughts. 

## Design Goals

The main design objectives of the UR are:
 
* **Generic:** The UR will be a neutral implementation that will be usable as broadly as possible, i.e. not limited towards a specific use case or higher-level protocol.
* **Simple:** The most common uses of the UR should be very simple to execute (e.g. "give me the DDO or the Hub endpoint for this identifier").
* **Extensible:** The UR may ship by default with a few key implementations of common identifier systems, but must also be extensible to easily add new ones through a plugin mechanism.


## Out of Scope
The following functionality is out of scope for the UR:

* The UR will not address registration or modification of identifiers and associated data.
* The UR will not involve authentication or authorization of the UR user, i.e. all its functionality will be publicly usable by anyone. [debatable?]
* The UR will not implement any functionality that builds on top of resolution, such as the DIF Hub protocols, Blockstack storage layer, XDI, DID Auth, DKMS, etc. [debatable?], but it will be able to serve as a building block for such higher-level protocols.

## Architecture
The universal resolver's architecture will consist of the following:
* Abstract interface definition of resolution / discovery operations.
* Extensible mechanism for "identifier system plugins".
* A base set of identifier system plugins we want to support initially.
* Shared components that will be useful across multiple identifier system plugins.
* A Web Proxy.

Note that an identifier system plugin may internally fulfill resolution in different ways, e.g.:
![System Architecture](/docs/figures/architecture.png)

![System Architecture](/docs/figures/architecture-old.png)

* The plugin for BNS may be able to access a local full Bitcoin/Virtualchain node, or it may itself call a remote API.
* The plugin for Sovrin DIDs may dynamically construct a DDO rather than retrieving an actual DDO file from the Sovrin Ledger.

![System Architecture](/docs/figures/overview.png)

second doc:

The universal resolver's main task is to provide an API wrapper around one or
more specific naming systems.  It does so by running a client for each system in
a colocated container or VM (bold boxes).  The "Service Orchestrator" is
responsible for spawning the required naming system clients, feeding them their
configuration data, and forwarding
HTTP requests to each of them (see below).

### Naming Service Clients

Naming service clients come in two flavors:  a lightweight client, and a full
node.  The difference between the two is that a full node will synchronize its
name state with a blockchain or distributed ledger, and host all the requisite
state (e.g. the chain state) and run all the requisite software (e.g. blockchain
peers) locally.  **This is a resource-intensive configuration, and is meant
primarily for dedicated servers.**

A lightweight client will instead contact a full node running on an external host.
The details as to which host to contact (and how to contact it) will be fed to the lightweight node's
container by the service orchestrator upon instantiation.  This configuration is
meant for devices that cannot reliably synchronize state with the naming
service, due to e.g. resource constraints and offline operating modes (such as
laptops and mobile phones).

The interface to both lightweight client and full node containers is identical.

## Identifier Systems

The UR will aim to initially include plugins for the following identifier systems:

* As many DID methods as possible, to the extent they have been specified:
    * **did:sov:***
    * **did:btcr:***
    * **did:uport:***
    * **did:cnsnt:***

* The Blockchain Name System (BNS):
    * *.id
* The Ethereum Name Service (ENS):
    * *.eth

## Open Questions

#### Identifier Ambiguity
How can the UR determine which identifier system should be used to resolve a given identifier?  This is also known as the "multiple namespaces problem". E.g. the identifier markus.eth could be a semantic name registered in ENS, but it could also be registered in BNS. If one objective for the UR is extensibility, then ambiguity may be inevitable. In addition, some identifier systems may offer multiple deployed instances such as a "mainnet" and a "testnet".

The UR should implement some logic (we can use the term "identifier system selector") to make sure the outcome of every resolution request is deterministic, based on

1. Community consensus and recommendations by the IND WG.
2. Custom configuration by the UR user.
3. Input parameter by the UR user.

#### Reverse Lookup
Should reverse lookup be supported, e.g. find semantic names given a DID?

#### Identifier Discovery
Should search for identifiers be supported, e.g. return a list of all identifiers that satisfy certain criteria? This could involve the use of higher-level functionality such as the DIF Hub protocols or Blockstack storage layer.

#### Local Names
Should local names be supported? This would imply some kind of local state or context to be held by the UR user, such as an address book.

#### Caching Behavior
Should resolution results be cached? This could be determined by a combination of
Community consensus and recommendations by the IND WG.
Custom configuration by the UR user.
Input parameter by the UR user.
Contents of the identifier's descriptor object (DDO, BNS zone file).

## Interface Specification
The UR will offer the following operations and input/output parameters:
### resolve()
#### Input
* **id:** The identifier to be resolved, i.e. semantic name, DID, etc.
* **resulttype** (optional): The UR user may be interested in different types of results. Not all result types may be supported by all identifier system plugins. Possible result types:
    * The raw descriptor object associated with an identifier (e.g. DDO, BNS zone file).
    * The DID that a semantic name maps to.
    * The current public key associated with an identifier.
A certain service endpoint from the descriptor object ("routing").
* **resultquery (optional):** This parameter could be used to further narrow down the result, e.g. which service endpoint should be selected from a DDO (the Hub endpoint, OpenID endpoint, XDI endpoint, etc.).
* **hint (optional):** One or more hint parameters could indicate to the UR certain preferences for the resolution process, e.g. with regard to the "Identifier Ambiguity" and "Caching Behavior" questions mentioned above.
#### Output
* **result:** Depending on the resulttype, this is the resolved DDO, BNS zone file, DID, public key, service endpoint, etc.
* **resultmetadata (optional):** Various metadata about the resolution process, e.g. is this a cached result, has the result been obtained from a local full node or a local thin client or a remote API, has a signature on a DDO been found and verified, etc.

## Return Values

We are still waiting on a well-defined schema for a DDO, but the jist of the resolver's return value is that it is a compound object containing:

* the version string (matches `:versionString` in the request)
* the DDO identified by the fully-qualified name
* a catch-all `supplementary` object that provides service-specific hints to the client.

The `supplementary` field is for forward compatibility with future systems.  This is a field a service client can use to return something service-specific to clients (for example, Blockstack and ENS might want to give back the relevant transaction ID that created the DID).  If it is discovered that each client returns the same types of data in their `supplementary` fields, then we will standardize the response in a future version of this specification.

Schema (NOTE: missing precise DDO fields)

```
    {
       'type': 'object',
       'properties': {
          'version': {
             'type': 'string',
             'pattern': '^1\.0$',
          },
          'ddo': {
             'type': 'object',
             'properties': {
                '...'
             }
             'required': [
                '@context',
                'id',
                'signature',
             ],
          },
          'supplementary': {
             'type': 'object',
             'additionalProperties': true,
          },
       },
       'additionalProperties': false,
    }           
```

Examples (DDO fields gleaned from https://github.com/WebOfTrustInfo/rebooting-the-web-of-trust-fall2016/blob/master/did-spec-wd03.md)

```
    {
       'version': '1.0',
       'ddo': {
           '@context': '...',
           'id': 'did:sov:21tDAKCERh95uGgKbJNHYp',
           'equiv-id': [
              'did:sov:33ad7beb1abc4a26b89246',
              'did:uport:2opT3phRXKtkaqjv6LAyR9pqkVwADVECZwx',
              'did:bstk:479395-27'
           ],
           'verkey': 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCABMC',
           'control': [
              'self',
              'did:sov:21tDAKCERh95uGgKbJNHYp',
           ],
           'service': {
              'dif': 'https://dif.microsoft.com',
              'blockstack': 'https://explorer.blockstack.org',
              'ens': 'https://etherscan.io/enslookup',
              'openid': 'https://vicci.org/id',
           },
           'type': 'https://schema.org/Person',
           'creator': 'did:sov:21tDAKCERh95uGgKbJNHYp',
           'created': '2016-10-10T17:00:00Z',
           'updated': '2017-03-14T18:00:30Z',
           'signature': {
              'type': 'LinkedDataSignature2015',
              'created': '2016-02-08T16:02:20Z',
              'creator': 'did:sov:21tDAKCERh95uGgKbJNHYp/keys/1',
              'signatureValue': 'QNB13Y7Q9oLlDLL6AHyL31OE5fLji9DwJSA8qnv81oRaKonij8m+Jv4XdiEYvJ97iRlzKU/92/0LafSL5JftEgl960DLcbqMFxOtbAmFOIMa7eDcrgTL5ytXeYCYKLjHQG3s8a3UKDKRuEK54qK1G5hGKGoLgAVa6xgcDLjW7M19PEJV/c3HpGA7Eez6VFMoTt4yESjZvOXC97xN3KpshOx2HT/btgUbo0XjA1Oi0QHdgrLcUsQGt6w23RjeSToalrsA1G69OFeN2OiQrz9Jb4561hvKLSyWObwRmS6n5Vgr5xkvUm6MONRq0Vg33kXevoVM64KTBkISul61tzjn4w==',
           },
        },
        'supplementary': {
           'whois': {
              "block_preordered_at": 373622, 
              "block_renewed_at": 373622, 
              "expire_block": 489247, 
              "has_zonefile": true, 
              "last_transaction_height": 422657, 
              "last_transaction_id": "5e4c0a42874ac59a9e21d351ec8b06971231ed2c02e0bac61b7dcc8ee8cb1ad2", 
              "owner_address": "16EMaNw3pkn3v6f2BgnSSs53zAKH4Q8YJg", 
              "owner_script": "76a914395f3643cea07ec4eec73b4d9a973dcce56b9bf188ac", 
              "zonefile_hash": "4d718e536b57c2dcc556cf2cdbadf1b6647ead80"
           },
           'zonefile': "$ORIGIN judecn.id\n$TTL 3600\npubkey TXT \"pubkey:data:04cabba0b5b9a871dbaa11c044066e281c5feb57243c7d2a452f06a0d708613a46ced59f9f806e601b3353931d1e4a98d7040127f31016311050bedc0d4f1f62ff\"\n_file URI 10 1 \"file:///home/jude/.blockstack/storage-disk/mutable/judecn.id\"\n_https._tcp URI 10 1 \"https://blockstack.s3.amazonaws.com/judecn.id\"\n_http._tcp URI 10 1 \"http://node.blockstack.org:6264/RPC2#judecn.id\"\n_dht._udp URI 10 1 \"dht+udp://fc4d9c1481a6349fe99f0e3dd7261d67b23dadc5\"\n"
       },
    }       
```
## Web Proxy Resolver
The UR will include an implementation of a Web Proxy. This is a web server that exposes the UR's **resolve()** operation via an HTTP API that can answer resolution requests using standard HTTP calls (probably HTTP GET). The required id input parameter(s) for the **resolve()** operation - i.e. the identifier to be resolved - can be passed as part of the URL, e.g.:

<pre>
https://decentralized-identity.github.io/resolve/<b>did:sov:21tDAKCERh95uGgKbJNHYp</b>
https://decentralized-identity.github.io/resolve/<b>peacekeeper.id</b>
</pre>

For additional input parameters such as **resulttype**, we need to specify:


* How to set their values in a request to the web proxy (e.g. using query string parameters or HTTP headers).
* What are their default values (e.g. the default result type, or the default service endpoint).

Depending on the result type, the web proxy may behave in different ways:

* Send the result (e.g. DDO) in the response body with an appropriate MIME type.
* If the result is a single service endpoint URI, send an HTTP redirect (maybe HTTP 307? [debatable]).

## Implementation
TBD details on how the UR will be implemented, maintained, documented, how identifier system plugins can be written, etc.

## Terminology

Commonly-used terms in this document.

* DID: decentralized identifier
* DDO: DID doc (JSON.ld format)

## Protocol

![Protocol](/docs/figures/protocol.png)



The universal resolver relies on its naming service clients to resolve
fully-qualified names or DIDs into DDOs.  It does this simply by forwarding requests it
receives to the requisite client, and caching the response.  **The HTTP headers in
the request will be used to determine whether or not to check the cache, and for
how long to cache the DDO response.**

The service orchestrator will forward the request to **at most one naming
service client.**.  It will choose which one based on the suffix of the name
(see below).

# API

## Resolver Interface

A single endpoint for the universal resolver is defined:  `GET /:versionString/identifiers/:fullyQualifiedNameOrDID`

* `:versionString` is the resolver API version.  For now, this is `1.0`.
* `:fullyQualifiedNameOrDID` is the fully-qualified name or DID to query.  This includes any/all indications of things like which system the name lives in, which blockchain the name is registered on, which namespace it lives in, and so on.

Examples:

* `GET /1.0/identifiers/judecn.id.bsk` resolves `judecn.id` in Blockstack’s virtualchain to a DDO.
* `GET /1.0/identifiers/nickjohnson.eth.ens` resolves `nickjohnson.eth` using ENS to a DDO.
* `GET /1.0/identifiers/did:sov:33ad7beb1abc4a26b89246` resolves the DDO for `did:sov:33ad7beb1abc4a26b89246` using Sovrin (`sov`).

### DIDs

A DID is a string that starts with `did:`.  If `:fullyQualifiedNameOrDID` starts
with `did:`, it will be treated as a DID _even if it is also a well-formed
name_.  The reason for this is to discourage the use of names that have an
ambiguous interpretation.

See [this
document](https://docs.google.com/document/d/1rEPRjmRCwhLEfW7Cdwf-aGYXACqK-IFhD9o8bXaT6H0/edit#heading=h.yphg7n6k1rpo) for more details on resolving DIDs to DDOs.

### Fully-Qualified Names

Each name _must_ end in a '.' followed by a URL-safe suffix.  This suffix identifies the service that can resolve
the name.  This is similar in function to the method ID in a DID.  Anyone can
claim an identifier, provided that it is unique.

**Names _may not_ start with `did:`.**  This prefix is reserved for DIDs.

A name _must_ resolve to at most one DDO.  The service client _may_ do so by
first resolving the name to a DID, and then resolving the DID to a DDO.

Identifiers in use today:

* `.bsk`: for Blockstack-hosted names
* `.ens`: for ENS-hosted names
* ... (insert yours here)

## Service Client Interface

The service client interface has two endpoints, one for fully-qualified names
and one for DIDs.  The reason for two endpoints is that the service client _must
not_ be responsible for disambiguating an identifier string.  This
responsibility belongs only to the resolver.

The service client makes these endpoints available via HTTP.  The two endpoints are:

* A `names` endpoint: `GET /:versionString/names/:fullyQualifiedName`
* A `dids` endpoint: `GET /:versionString/dids/:DID`

A client _must_ implement at least one endpoint.  However, if it does not
implement an endpoint, it _must_ handle requests to it by returning `HTTP 501`.

Both endpoints return at most one DDO.  Implementations are encouraged, but not
required, to resolve the `:fullyQualifiedName` to a DID and then resolve the DID
to the DDO.
