---
name: Query business data through a Skyvia Connect MCP endpoint
description: >-
  Use the four tools a Skyvia Connect MCP endpoint exposes — Instructions, Objects, ObjectInfo and Execute —
  to discover what data sits behind an endpoint and query it safely with parameterized SQL. Grounded in the
  live tool manifest read from mcp.skyvia.com, including the schemaOnly dry-run and the byte-metered traffic
  quota that makes unbounded queries expensive.
api: mcp/skyvia-mcp.yml
base_url: https://mcp.skyvia.com
operations:
  - Instructions
  - Objects
  - ObjectInfo
  - Execute
transport: streamable-http
---

# Query data through a Skyvia Connect MCP endpoint

A Skyvia Connect MCP endpoint fronts **one** connection — one cloud app or database out of the 200+ Skyvia
supports. The tool set is always these four; the data behind them is whatever the endpoint owner bound.

Your endpoint URL is `https://mcp.skyvia.com/{endpointId}`. If the endpoint has user accounts configured,
authenticate with HTTP Basic: base64-encode `username:password` into the `Authorization` header. Endpoints
without user accounts accept any caller who has the URL.

## 1. Read the endpoint's own instructions first

Call **`Instructions`** (no arguments). It returns the server's usage guidance for this specific endpoint.
Do this before anything else — it is the only per-endpoint documentation that exists, and it costs almost
nothing.

## 2. Discover the objects

Call **`Objects`**:

- `includeNonQueryable` (boolean, default `false`) — leave it false unless you specifically need objects
  that cannot be queried.
- `responseFormat` (`Csv` | `Json` | `Markdown`, default `Csv`) — ask for `Json` if you are parsing,
  `Markdown` if you are reasoning over it.

This returns the tables, views or cloud objects behind the endpoint with brief information about each.

## 3. Inspect the object you care about

Call **`ObjectInfo`** with `objectName` (required):

- `addParentRelations` (boolean, default `false`) — set true to see what this object references.
- `addChildRelations` (boolean, default `false`) — set true to see what references it.
- `responseFormat` — same three options.

Turn the relation flags on before writing any query that joins. Skyvia's underlying connectors normalize
very different sources into a relational shape, and the relationships are not guessable from names.

## 4. Validate the query before you run it

Call **`Execute`** with `schemaOnly: true` inside `body`. The server validates the SQL and returns **only
the result schema, no data**. This is the dry run — use it every time before a query you have not run
before. It confirms the SQL parses against this source and shows you the columns you will get back.

## 5. Run the query

Call **`Execute`** with a `body` object:

```json
{
  "sql": "SELECT Id, Name, CreatedDate FROM Accounts WHERE CreatedDate > :since",
  "parameters": [
    { "name": "since", "dbType": "DateTime", "value": "2026-01-01T00:00:00Z" }
  ],
  "pageSize": 500,
  "schemaOnly": false
}
```

- **Always use `parameters`, never string concatenation.** Each parameter takes `name`, `dbType` and
  `value`. `dbType` is the ADO.NET `DbType` set — `AnsiString`, `Binary`, `Byte`, `Boolean`, `Currency`,
  `Date`, `DateTime`, `Decimal`, `Double`, `Guid`, `Int16`, `Int32`, `Int64`, `Object`, `SByte`, `Single`,
  `String`, `Time`, `UInt16`, `UInt32`, `UInt64`, `VarNumeric`, `AnsiStringFixedLength`,
  `StringFixedLength`, `Xml`, `DateTime2`, `DateTimeOffset`. Default is `String`.
- `pageSize` defaults to **500**. The first page comes back with a next-page token; fetch further pages
  with that token.
- `commandTimeout` (seconds) bounds a slow query. `expireTimeout` (seconds, default **3600**) is how long
  the result reader stays alive for paging.

## Cost discipline — this is the important part

Skyvia meters Connect by **traffic in bytes per month**, not by request count: 100 KB on Free, 1 MB on
Basic, 100 MB on Standard, 1 GB on Professional. A single unbounded `SELECT *` can exhaust a whole month's
quota in one call.

So, always:

1. `SELECT` named columns, never `*`.
2. Put a `WHERE` on it.
3. Keep `pageSize` at or below the default and stop paging once you have what you need.
4. Prefer `responseFormat: Csv` (the default) for bulk reads — it is materially smaller than `Json` for the
   same rows.

## What this endpoint cannot do

`Execute` runs whatever the endpoint's per-object and per-operation permissions allow — a read-only endpoint
and a read-write endpoint expose the **identical** tool contract, so the tool description is not a safety
boundary. Check with the endpoint owner what writes are permitted before issuing anything but a `SELECT`.

The MCP surface reaches **customer data only**. It cannot run an integration, trigger a backup, or change
anything in Skyvia itself — that is the Public REST API, a completely separate surface with a separate
credential. See `mcp/skyvia-tool-crosswalk.yml`.

## Related artifacts

`mcp/skyvia-mcp.yml` · `mcp/skyvia-tool-crosswalk.yml` · `plans/skyvia-plans-pricing.yml` ·
`authentication/skyvia-authentication.yml`
