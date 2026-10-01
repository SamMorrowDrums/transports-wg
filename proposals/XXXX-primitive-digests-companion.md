# SEP-XXXX: Definition Versions

- **Status**: Draft (discussion; not an accepted protocol change)
- **Type**: Standards Track
- **Created**: 2026-09-04
- **Author(s)**: TBD
- **Sponsor**: None
- **Related**: [PR #45](https://github.com/modelcontextprotocol/transports-wg/pull/45), [SEP-2549](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2549), [HTTP list retrieval and caching](XXXX-http-list-retrieval-and-caching.md)

## Abstract

This SEP lets MCP Servers supply a digest with their CacheableResults. 

Clients can return digests to the MCP Server, which can choose to reject the call if the digest is not serviceable. 

The mechanism is advisory. Servers choose whether to advertise versions and what to do with the ones they receive.

Extensions that define their own lists (`skills/list` for example) can participate using the same structure.

## Motivation

Hosts that cache Tool Lists can call tools after the Server has changed them. The `2026-07-28` specification `ttlMs` adds a "freshness hint, not a guarantee" and notes that:

> Servers MAY change the underlying data before TTL expires.

Tool definitions can change for may reasons; deployments, permissions, feature flags and user settings. Hosts can make tool call requests against stale schemas. Often standard validation will catch mismatches, but in pathological cases description and argument semantics may change and the model may issue tool calls that weren't intended.

The digest mechanism provides three options for a Server:
- **Ignore incoming digest:** Requests are handled using existing mechanisms as they are today.
- **Handle Request:** The digest is used to select the appropriate response allowing graceful upgrades and task drains.
- **Raise Error:** An error is returned indicating that the requested digest version is not available.

*Authors note: "Handle Request" may be difficult or SDK dependent due to the need to handle multiple Request shapes for the same tool, prompt identity and so on.* 

The mechanism works alongside the existing `ttlMs` hint. Clients may choose more aggressive caching strategies based on optimistic calling.

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
  /**
   * Collections defined by extensions. Keys follow the `_meta` naming rules
   * and carry a mandatory prefix, for example "io.modelcontextprotocol/skills".
   */
  [extensionCollection: string]: string | undefined;
}
```

Each collection version identifies the complete set of definitions the caller can see, not a single page. The `resources` and `resourceTemplates` versions cover descriptors, not resource contents. The `instructions` version identifies the exact instructions text; absent instructions and empty instructions have different versions. An omitted field makes no claim about that collection.

Unprefixed keys are reserved for collections defined by the core protocol. An extension that defines a list operation **SHOULD** say which key it uses here and carry that key's version on its list results. The skills extension is the obvious first case: `skills/list` already mirrors the base protocol's `ttlMs` and `cacheScope`, and would mirror this in the same way, under `io.modelcontextprotocol/skills`. A skills version covers the catalog entries, not the files they point to; those already carry their own digests.

A version **MUST** be deterministic and collision-resistant. Adding, removing, or changing a definition changes the version. Ordering, page size, cursors, and TTL do not. An empty collection still has a version.

A version **MAY** also change when the definitions have not, for example after a change to how versions are computed. Clients treat this like any other change and refresh.

Clients **MUST** treat versions as opaque strings and compare them only for equality.

The exact canonicalization and the definition fields a version covers are not yet specified (see [Open Questions](#open-questions)).

### Where versions appear

`server/discover` **MAY** advertise versions for any collection and for the instructions it returns. Each list response **MAY** carry the version of its collection. A server **MUST** compute versions in discovery using the same authorization and tool-selection context it uses for listing, so that the two agree.

Versions travel in result `_meta`, under a typed key on `ResultMetaObject`:

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

This is the same pattern the draft schema already uses for `io.modelcontextprotocol/serverInfo` and `subscriptionId`: cross-cutting protocol data that any result can carry without a schema change to the result itself. It works today with existing SDKs, and it works for extension results such as `skills/list`, which are not in the core schema and could not inherit a new field from `CacheableResult` anyway. The `io.modelcontextprotocol/` key is proposed here and would be allocated on acceptance.

Conceptually the version belongs next to `ttlMs` and `cacheScope`. They say how long a result may be held and who may share it; the version says what it is. Once there is field experience, a follow-up **MAY** promote `definitionVersions` to a field on `CacheableResult`, keeping the `DefinitionVersions` type and its semantics unchanged. Servers would emit both for a transition period and clients would prefer the field. The main wrinkle is that `resources/read` also returns a `CacheableResult` and there is no content version to put there, so the field would always be omitted for it. That is a matter for the follow-up.

### Using versions on the client

Clients **MUST** keep each version with the definitions it describes. Seeing a newer version in discovery does not relabel definitions that are already cached; it tells the client they are out of date. A `list_changed` notification says the same thing: a client that receives one **SHOULD** treat its cached version for that collection as stale, and a server that sends one **SHOULD** have changed the version too.

When assembling a collection from several pages, a client **MUST NOT** combine pages that report different versions. It restarts from the first page instead. To make a consistent listing possible, a server can keep a snapshot for the duration of pagination or reject stale cursors.

### Sending known versions

A client **MAY** send the versions it is working from in request `_meta`. It **SHOULD** only send versions it received from the same server in the same authorization context, and omit the field otherwise. Versions are sent on each request rather than fixed for a session, because what the client is working from can change between one call and the next.

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

Known versions are hints, not preconditions. No capability flag is required, and the instructions version is relevant to any operation.

### Handling known versions

A server that receives known versions on `tools/call`, `prompts/get`, `resources/read`, or an extension's equivalent has three choices:

- **Ignore them.** The request is handled exactly as it would be without hints. This is the default and is always allowed, including after the server has advertised versions.
- **Honor them.** If the server still holds the definitions a known version describes, it **MAY** serve the request under those definitions. This is how a server drains an old definition set across a deployment, or lets a long-running task finish against the tools it started with. A successful response then means the old definitions ran, and the client should not assume otherwise until it refreshes and sends the new version.
- **Reject them.** If a known version does not match and the server will not honor it, the server rejects the request as stale.

Whatever it does, a server **SHOULD NOT** reject a request because it names targets the server does not version or carries values that are not strings; it ignores those. Any string that does not equal a version the server recognizes is a mismatch.

A server that rejects a request for a version mismatch **MUST** do so before operation-specific validation and before execution. That way a changed schema is reported as a stale definition, not as invalid arguments, and no side effects occur.

The rejection is a JSON-RPC error. Its `data` **SHOULD** list the stale targets, using the same keys as `DefinitionVersions`, so the client knows what to refresh. It **MUST NOT** include the current versions:

```json
{ "stale": ["tools", "io.modelcontextprotocol/skills"] }
```

A standard error code needs to be allocated before this SEP is finalized; this draft does not propose a numeric code or an HTTP status. `UnsupportedProtocolVersionError` in the draft schema is the nearest model.

A client receiving this error **SHOULD** refresh the affected definitions and then decide whether the operation still makes sense. Refreshing means fetching the definitions from the server: the client **MUST NOT** satisfy the refresh from a cached list, even one whose TTL has not expired. The client **SHOULD NOT** blindly retry writes, and it **MUST NOT** copy a new version into its request without also fetching the definitions that version describes.

### TTL and cache scope

Definition versions do not change TTL or cache scope. A version does not renew a TTL, is not part of the cache key, and is not an HTTP ETag. An unexpired TTL still does not guarantee that definitions are unchanged.

## Rationale

**Collection versions instead of per-primitive versions.** An earlier draft attached a digest to every primitive and made the check mandatory. That design could not detect newly added primitives, and it required every replica behind an endpoint to honor any digest the server had advertised. One version per collection is simpler to compute and compare. It also covers additions and removals. The cost is coarseness: any change to a collection invalidates requests against it, even if the specific tool the client wants is unchanged. Collection versions are still enough for draining, because a server that wants to honor an old version holds the whole old snapshot; it does not need a history per tool.

**Advisory rather than enforced.** Making the check optional means no capability negotiation and no promise that must hold across a fleet of servers. The consequence is that a successful response does not prove the versions matched, or that the current definitions were the ones that ran. Clients get two cheap things regardless: a way to compare the versions in `server/discover` against what they cached, and a way to tell the server what they are working from so that it can do something sensible with the information.

**An open map.** The core protocol has five things worth versioning today, and extensions will add more; `skills/list` already exists and mirrors the base list caching fields. Letting extensions add prefixed keys to `DefinitionVersions` follows the pattern `capabilities.extensions` already uses, and means one structure, one request key, and one `stale` list cover everything, rather than each extension inventing its own.

**`_meta` first.** The version conceptually belongs beside `ttlMs` and `cacheScope`, and may end up there. Starting in `_meta` lets the mechanism be tried against real servers and clients without a schema or SDK change, keeps the request and response sides symmetric (the request side has to be `_meta` in any case), and reaches extension results that a `CacheableResult` field would not.

**The cost of checking.** Checking a single request means computing the version of the whole collection, which can cost more than serving the request itself: a server that normally builds only the one tool being called must now build them all. Servers can limit this by advertising versions only where the complete collection is cheap to build, and ignoring hints elsewhere. Clients help by sending known versions only when they hold one, so unchecked requests keep their existing cost.

**Independent of HTTP caching.** Versions identify definitions, while an ETag identifies an HTTP representation of a particular page. Keeping them separate lets this SEP work over any transport, and lets the [HTTP list retrieval](XXXX-http-list-retrieval-and-caching.md) proposal define ETags on its own terms.

**Instructions alongside collections.** Instructions can change how a model uses otherwise unchanged tools, so they are versioned independently in the same structure.

## Backward Compatibility

This SEP adds optional fields and does not change the behavior of existing requests. Clients can ignore response versions. Servers can ignore request hints, including after advertising versions. Implementations that do not recognize the new fields continue to work as they do today.

## Security Implications

A version carries the same confidentiality and authorization context as the definitions it describes, and a result carrying several versions is as private as the most private of them. For example, a tool list may be identical for many users while instructions name the individual user; a discovery result carrying both versions is then private to that user. Versions grant no access. They also do not prove that a server's implementation is unchanged, only its advertised definitions. Operations continue to enforce current permissions regardless of the versions a client sends.

The same care applies to the cache scope of a result carrying versions. A result may be marked `public` only if every caller would receive the same result, not merely if it contains no user data. For example, an anonymous caller asking for the tool list might receive three tools, while a signed-in caller making the same request should receive five, because two of the tools require sign-in. The anonymous list contains no user data, but it is not public: if the two callers share a cache, the signed-in caller is served the shorter list without reaching the server, and their known versions then fail to match.

## Reference Implementation

The [HF MCP server](https://github.com/huggingface/hf-mcp-server) has a prototype (not yet merged). It uses result and request `_meta` with application keys, `huggingface.co/definition-versions` in results and `huggingface.co/known-definition-versions` in requests, and needs no SDK changes. It rejects on mismatch; it does not yet honor old versions.

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
- When honoring is enabled, a request carrying an old version runs against the old definitions, and one carrying an unknown version is rejected.
- Unversioned targets, extension keys the server does not version, and malformed hints do not cause a rejection.
- After a mismatch, the client refreshes from the server rather than from a still-fresh cached list.

## Open Questions

- Whether, and when, to promote `definitionVersions` from `_meta` to a field on `CacheableResult`, and what to do about `resources/read` if so.
- How long a server that honors old versions should keep them, and whether it should say so. This draft leaves it to the server.
- How the skills extension (or any extension whose listing may be deliberately partial) should define its version, given that "the complete set the caller can see" is then the set the server chooses to enumerate.
- The canonicalization, and which definition fields a version covers (for example, whether a tool's own `_meta` is included).
- Whether servers must provide a consistent snapshot across pages or may require clients to restart.
- The standard error code for a version mismatch.
