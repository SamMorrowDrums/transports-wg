# SEP-XXXX: Definition Versions

- **Status**: Draft (discussion; not an accepted protocol change)
- **Type**: Standards Track
- **Created**: 2026-09-04
- **Author(s)**: TBD
- **Sponsor**: None
- **Related**: [PR #45](https://github.com/modelcontextprotocol/transports-wg/pull/45), [SEP-2549](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2549), [HTTP list retrieval and caching](XXXX-http-list-retrieval-and-caching.md)

## Abstract

This SEP lets servers label their tool, prompt, resource, and resource template lists, and their instructions, with a version string. Clients can send back the versions they are working from, and a server can use them to reject a request made against definitions that have since changed.

The mechanism is advisory. Servers choose whether to advertise versions and whether to check them. Clients choose whether to send them. There is no capability negotiation, no per-primitive versioning, and no HTTP mapping.

## Motivation

A client that caches `tools/list` according to its `ttlMs` can keep calling tools after the server has changed them. The TTL is a freshness hint, not a promise that nothing will change before it expires. Definitions change for many reasons: a deployment, a permission change, a feature flag, or a user editing their settings. When this happens, the model may be working from a stale description, schema, or set of instructions.

Sometimes ordinary validation catches the problem, for example when a required argument was added. Often it does not. A tool whose description changed, or whose behavior now depends on different instructions, can accept the old arguments and do something the model did not intend.

List-change notifications help, but they need a stream the client may not hold, and they arrive asynchronously. Definition versions give clients a cheap way to ask "has anything changed?" and give servers a way to say "yes, refresh first" before doing any work.

## Specification

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14).

### `DefinitionVersions`

```ts
export interface DefinitionVersions {
  tools?: string;
  prompts?: string;
  resources?: string;
  resourceTemplates?: string;
  instructions?: string;
}
```

Each collection version identifies the complete set of definitions the caller can see, not a single page. The `resources` and `resourceTemplates` versions cover descriptors, not resource contents. The `instructions` version identifies the exact instructions text; absent instructions and empty instructions have different versions. An omitted field makes no claim about that collection.

A version **MUST** be deterministic and collision-resistant. Adding, removing, or changing a definition changes the version. Ordering, page size, cursors, and TTL do not. An empty collection still has a version.

A version **MAY** also change when the definitions have not, for example after a change to how versions are computed. Clients treat this like any other change and refresh.

Clients **MUST** treat versions as opaque strings and compare them only for equality.

The exact canonicalization and the definition fields a version covers are not yet specified (see [Open Questions](#open-questions)).

### Where versions appear

`server/discover` **MAY** advertise versions for any collection and for the instructions it returns. Each list response **MAY** carry the version of its collection. A server **MUST** compute versions in discovery using the same authorization and tool-selection context it uses for listing, so that the two agree.

This SEP proposes one of two locations for the field. Only one will be standardized.

#### Option A: result `_meta`

A typed key is added to `ResultMetaObject`:

```ts
export interface ResultMetaObject extends MetaObject {
  // Existing fields unchanged.
  "io.modelcontextprotocol/definitionVersions"?: DefinitionVersions;
}
```

```json
{
  "resultType": "complete",
  "tools": [],
  "ttlMs": 60000,
  "cacheScope": "private",
  "_meta": {
    "io.modelcontextprotocol/definitionVersions": {
      "tools": "sha256:..."
    }
  }
}
```

This fits an experimental extension and works with existing metadata handling. The `io.modelcontextprotocol/` key is proposed, not allocated.

#### Option B: a `CacheableResult` field

```ts
export interface CacheableResult extends Result {
  ttlMs: number;
  cacheScope: "public" | "private";
  definitionVersions?: DefinitionVersions;
}
```

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": { "tools": {} },
  "instructions": "Search before creating a repository.",
  "ttlMs": 60000,
  "cacheScope": "private",
  "definitionVersions": {
    "tools": "sha256:...",
    "instructions": "sha256:..."
  }
}
```

This makes versions a first-class protocol field next to the existing freshness and sharing hints. The field identifies definitions, not the result that carries it. `resources/read` also returns a `CacheableResult`, but this SEP does not define a content version for it, and servers omit the field there.

### Using versions on the client

Clients **MUST** keep each version with the definitions it describes. Seeing a newer version in discovery does not relabel definitions that are already cached; it tells the client they are out of date.

When assembling a collection from several pages, a client **MUST NOT** combine pages that report different versions. It restarts from the first page instead. To make a consistent listing possible, a server can keep a snapshot for the duration of pagination or reject stale cursors.

### Sending known versions

A client **MAY** send the versions it is working from in request `_meta`. It **SHOULD** only send versions it received from the same server in the same authorization context, and omit the field otherwise. The request shape is the same under either response option:

```ts
export interface RequestMetaObject extends MetaObject {
  // Existing fields unchanged.
  "io.modelcontextprotocol/knownDefinitionVersions"?: DefinitionVersions;
}
```

For example, the parameters of a `tools/call` request might look like this (other required `_meta` fields are omitted):

```json
{
  "name": "search",
  "arguments": { "query": "datasets" },
  "_meta": {
    "io.modelcontextprotocol/knownDefinitionVersions": {
      "tools": "sha256:...",
      "instructions": "sha256:..."
    }
  }
}
```

Known versions are hints, not preconditions. A server **MAY** check the relevant versions on `tools/call`, `prompts/get`, or `resources/read`, or **MAY** ignore them. The instructions version is relevant to any of these operations. No capability flag is required.

A server that checks known versions **SHOULD NOT** reject a request because it contains targets the server does not version or values that are not strings; it ignores them. Any string that does not equal the current version is treated as a mismatch.

### Rejecting stale requests

A server that rejects a request because of a version mismatch **MUST** do so before operation-specific validation and before execution. That way a changed schema is reported as a stale definition, not as invalid arguments, and no side effects occur.

The rejection is a JSON-RPC error. Its `data` **SHOULD** list the stale targets, so the client knows what to refresh, and **MUST NOT** include the current versions:

```json
{ "stale": ["tools"] }
```

A standard error code needs to be allocated before this SEP is finalized; this draft does not propose a numeric code or an HTTP status.

A client receiving this error **SHOULD** refresh the affected definitions and then decide whether the operation still makes sense. Refreshing means fetching the definitions from the server: the client **MUST NOT** satisfy the refresh from a cached list, even one whose TTL has not expired. The client **SHOULD NOT** blindly retry writes, and it **MUST NOT** copy a new version into its request without also fetching the definitions that version describes.

### TTL and cache scope

Definition versions do not change TTL or cache scope. A version does not renew a TTL, is not part of the cache key, and is not an HTTP ETag. An unexpired TTL still does not guarantee that definitions are unchanged.

## Rationale

**Collection versions instead of per-primitive versions.** An earlier draft attached a digest to every primitive and made the check mandatory. That design could not detect newly added primitives, and it required every replica behind an endpoint to honor any digest the server had advertised. One version per collection is simpler to compute and compare. It also covers additions and removals. The cost is coarseness: any change to a collection invalidates requests against it, even if the specific tool the client wants is unchanged.

**Advisory rather than enforced.** Making the check optional means no capability negotiation and no promise that must hold across a fleet of servers. The consequence is that a successful response does not prove the versions matched. The main value for clients is comparing the versions in `server/discover` with those they cached, which tells them cheaply whether to re-list.

**The cost of checking.** Checking a single request means computing the version of the whole collection, which can cost more than serving the request itself: a server that normally builds only the one tool being called must now build them all. Servers can limit this by advertising versions only where the complete collection is cheap to build, and ignoring hints elsewhere. Clients help by sending known versions only when they hold one, so unchecked requests keep their existing cost.

**Independent of HTTP caching.** Versions identify definitions, while an ETag identifies an HTTP representation of a particular page. Keeping them separate lets this SEP work over any transport, and lets the [HTTP list retrieval](XXXX-http-list-retrieval-and-caching.md) proposal define ETags on its own terms.

**Instructions alongside collections.** Instructions can change how a model uses otherwise unchanged tools, so they are versioned independently in the same structure.

## Backward Compatibility

This SEP adds optional fields and does not change the behavior of existing requests. Clients can ignore response versions. Servers can ignore request hints, including after advertising versions. Implementations that do not recognize the new fields continue to work as they do today.

## Security Implications

A version carries the same confidentiality and authorization context as the definitions it describes, and a result carrying several versions is as private as the most private of them. For example, a tool list may be identical for many users while instructions name the individual user; a discovery result carrying both versions is then private to that user. Versions grant no access. They also do not prove that a server's implementation is unchanged, only its advertised definitions. Operations continue to enforce current permissions regardless of the versions a client sends.

The same care applies to the cache scope of a result carrying versions. A result may be marked `public` only if every caller would receive the same result, not merely if it contains no user data. For example, an anonymous caller asking for the tool list might receive three tools, while a signed-in caller making the same request should receive five, because two of the tools require sign-in. The anonymous list contains no user data, but it is not public: if the two callers share a cache, the signed-in caller is served the shorter list without reaching the server, and their known versions then fail to match.

## Reference Implementation

The [HF MCP server](https://github.com/huggingface/hf-mcp-server) has a prototype (not yet merged). It uses Option A with application keys, `huggingface.co/definition-versions` in results and `huggingface.co/known-definition-versions` in requests, and needs no SDK changes.

- It versions tools and instructions, and checks known versions on `tools/call` only.
- A mismatch is rejected before tool lookup, argument validation, or execution, with the application error code `-32987` (outside JSON-RPC's reserved range) and `data: { "stale": [...] }`. Unversioned targets and non-string hints are ignored.
- Versions are offered only where the complete tool list is cheap to build: anonymous requests and requests for a named, fixed set of tools. Other requests get no versions, their hints are ignored, and they keep the existing single-tool fast path. Versioned results also carry `private` TTL cache hints; anonymous lists are not `public`, because they omit tools that require sign-in.
- Tools are sorted by name, each tool's own `_meta` is included, and canonicalized JSON is hashed with SHA-256. Result-envelope metadata is excluded. For testing, a deploy-wide or runtime salt changes every version without changing definitions, forcing clients through the mismatch path.

Client integration in [fast-agent](https://github.com/evalstate/fast-agent) is in progress. It is optimistic: it sends known versions with each call and refreshes definitions when a call is rejected.

### Testing Plan

Implementations should test:

- Versions are deterministic, and change when definitions change.
- Empty collections have versions, and absent instructions differ from empty instructions.
- Versions are unaffected by ordering, pagination, and TTL.
- Existing result metadata is preserved.
- Versions are isolated between callers with different authorization contexts.
- Requests are handled normally when hints are absent or ignored.
- When checking is enabled, stale requests are rejected before any side effect.
- Unversioned targets and malformed hints do not cause a rejection.
- After a mismatch, the client refreshes from the server rather than from a still-fresh cached list.

## Open Questions

- Which response location to standardize: `_meta` or `CacheableResult`. The prototype shows Option A works with no SDK changes; Option B needs schema and SDK support.
- The canonicalization, and which definition fields a version covers (for example, whether a tool's own `_meta` is included).
- Whether servers must provide a consistent snapshot across pages or may require clients to restart.
- The standard error code for a version mismatch.
