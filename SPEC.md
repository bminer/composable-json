# Composable JSON

- **Version:** 1.0.0-draft
- **Date:** 2026-10-02
- **Editor:** Blake Miner
- **Latest version:**
  <https://github.com/bminer/composable-json/blob/master/SPEC.md>
- **License:** MIT

Composable JSON defines a standard way for JSON documents (RFC 8259) to reuse or
override values from other JSON documents.

Reuse is expressed with _directives_: object keys beginning with `$`, which are
reserved for the purpose. A _resolver_ replaces each directive with the value it
references, producing ordinary JSON. This specification defines seven
directives, the syntax of a reference, and how a resolver combines what it
references.

## Motivation

JSON has no concept of composition or reuse. There is no include, no import, and
no inheritance; every document stands alone, and anything shared between them
must be restated in each one. That becomes a problem as soon as a set of
documents differ only slightly from each other (e.g. one configuration per
environment, or a suite of test cases sharing a baseline and varying only
slightly). Restating the common part everywhere means a change to a shared value
has to be repeated in every copy, and a copy that drifts gives no sign that it
has.

Other formats grew their own answers to this — YAML anchors, `extends` in
tsconfig and ESLint, Kustomize overlays, and the configuration languages — but
each is tied to a particular format or tool (see [Prior art](#prior-art)). This
specification does that job for plain JSON: a Composable JSON document is still
valid JSON that any parser will read.

## Example

Given `service.base.json`:

```json
{
	"server": { "host": "0.0.0.0", "port": 8080, "tls": false },
	"logging": { "level": "info", "format": "json" },
	"database": { "host": "localhost", "port": 5432, "poolSize": 10 }
}
```

a document that builds on it using the `$extend` directive:

```json
{
	"$extend": "./service.base.json",
	"server": { "tls": true },
	"logging": { "level": "debug" },
	"database": { "host": "db.staging.internal" }
}
```

resolves to:

```json
{
	"server": { "host": "0.0.0.0", "port": 8080, "tls": true },
	"logging": { "level": "debug", "format": "json" },
	"database": { "host": "db.staging.internal", "port": 5432, "poolSize": 10 }
}
```

Objects merge key by key, to any depth, and the importing document wins.

## Part 1 — Writing documents

### Directives at a glance

| Directive  | Purpose                                              | Value                  |
| ---------- | ---------------------------------------------------- | ---------------------- |
| `$ref`     | replaces this node with the referenced value         | one reference          |
| `$extend`  | merges referenced objects under this node's own keys | one or more references |
| `$splice`  | inserts referenced arrays' elements in its place     | one or more references |
| `$anchor`  | names this node so that references can find it       | a name                 |
| `$defs`    | holds fragments referenced within the document       | any object             |
| `$comment` | records a note, ignored when resolving               | a string               |
| `$schema`  | names a schema for editors, ignored when resolving   | a string               |

`$ref`, `$extend` (and its synonym `$extends`) and `$splice` are consumed during
resolution and do not appear in the output. `$anchor`, `$defs` and `$comment`
are retained, and so is `$schema`, except in a value a reference delivers.

Retained keys keep a resolved value addressable: a later reference can still
find an anchor or point into `$defs`. A consumer of the final output rarely
needs them, so an implementation may offer to omit them from the output.

### Host directives

A specification built on this one — a configuration format or a test format, say
— may define `$`-prefixed keys of its own, and must document them. A resolver is
told which those are, and **errors on any `$`-prefixed key that neither it nor
its host format declares.**

Reserving the namespace rather than only the keys defined here is what makes
that last rule possible, and it is worth the cost: a misspelled `$extned`
resolves to a document that inherited nothing and reports no error, which is
among the harder failures to diagnose from the output alone.

This specification neither consumes nor interprets a host directive's key: it is
retained in the output, and merges like any other key, for the host to act on.
Its value is resolved like any other value, so it may itself use `$ref`,
`$extend` and `$splice`, and the host receives it fully resolved:

```json
{ "$csv": { "$extend": "./rows.base.json", "file": "./cases.csv" } }
```

reaches the host with `rows.base.json` already merged into the value of `$csv`.

A host should avoid names this specification might plausibly take later.
`$import`, `$merge`, `$delete` and `$override` have all been considered.

`$id` is reserved for a future version (see
[Relationship to existing standards](#relationship-to-existing-standards)). This
version ignores it: it is not an error, has no effect on resolution, and is
retained in the output like `$comment`. A host format must not define it.
Because documents may already contain `$id`, giving it a meaning later would be
a major release (see [Versioning](#versioning)).

`$literal` is reserved too, for a possible directive whose value is taken
verbatim: not resolved, not merged, and free to contain `null` and `$`-prefixed
keys. Unlike `$id`, it is an error in this version, so defining it later would
be a minor release. A host format must not define it.

### References

A reference is a URI reference (RFC 3986) whose fragment identifies a value
within the referenced document.

The character following `#` decides how the URI fragment is read, and so how the
reference is resolved:

- `/` — an RFC 6901 JSON Pointer, resolved from the document root
- end of string, or no fragment at all — the whole document
- a letter or `_` — an [`$anchor`](#anchor) name, terminated by `/` or end of
  string, optionally followed by a JSON pointer path relative to the anchored
  node
- anything else — an error

```abnf
reference       = URI-reference                    ; RFC 3986 §4.1
cj-fragment     = json-pointer / anchor-path
json-pointer    = *( "/" reference-token )         ; RFC 6901 §3
anchor-path     = anchor-name *( "/" reference-token )
anchor-name     = ( ALPHA / "_" ) *( ALPHA / DIGIT / "-" / "_" / "." )
reference-token = <reference-token, RFC 6901 §3>
```

The `fragment` component of a reference is first percent-decoded and read as
UTF-8, as RFC 6901 §6 describes, and the result must match `cj-fragment`. The
decoding comes first, so `%2F` separates tokens like `/` does; a `/` within a
key is written `~1`. `ALPHA` and `DIGIT` are the core rules of RFC 5234, and
`anchor-name` is the same production as the [`$anchor`](#anchor) value.

| Reference                      | Resolves to                                 |
| ------------------------------ | ------------------------------------------- |
| `./base.json`                  | the whole document                          |
| `./base.json#/staff/members/1` | JSON pointer into that document             |
| `#/middleware/0`               | JSON pointer into the current document      |
| `./base.json#oncall/members/1` | node with `oncall` anchor, then `members/1` |
| `#oncall/members/1`            | the same, within the current document       |

A reference URI without a scheme resolves against the location the containing
document was loaded from. For the precise rules — base URIs, supported schemes,
percent-encoding and native paths — see
[Base URIs, schemes, and encoding](#base-uris-schemes-and-encoding).

### `$ref`

`$ref` replaces the node that contains it with the value its reference resolves
to. Its value is a single reference string.

The referenced value may resolve to anything: an object, an array, a scalar, or
`null`. A node containing `$ref` may contain no other key, reserved or
otherwise, except [`$comment`](#comment), which is dropped along with the node.

Given `defaults.json`:

```json
{ "retry": { "attempts": 3, "backoff": "exponential" } }
```

then:

```json
{
	"name": "uploader",
	"attempts": { "$ref": "./defaults.json#/retry/attempts" }
}
```

resolves to:

```json
{ "name": "uploader", "attempts": 3 }
```

### `$anchor`

`$anchor` declares a name for the object that contains it. It is legal on any
object, and its value must match `^[A-Za-z_][-A-Za-z0-9._]*$` — the same
production JSON Schema 2020-12 uses. Names must be unique within a resolved
document; see [Anchor scope and collisions](#anchor-scope-and-collisions). Under
an `$extend` node, its value may also be `null`, which removes an imported name
(see [Removing a name](#removing-a-name)).

Given `roster.json`:

```json
{
	"title": "Rotations",
	"primary": { "$anchor": "oncall", "members": ["ana", "bo", "cy"] }
}
```

the reference `./roster.json#oncall/members/1` resolves to `"bo"`: the node
named `oncall`, then `members/1` within it. The plain pointer
`./roster.json#/primary/members/1` addresses the same value, but breaks if
`primary` is renamed or moved deeper. An anchor keeps a reference working where
a pointer would break.

An anchor is retained in the resolved output, so a name declared in one document
remains addressable after that document is imported into another.

### `$extend`

`$extend` combines one or more referenced objects with the node that contains
it, and the node's own keys take precedence. It is legal on any object, and its
value is a reference string or an array of references. The [Example](#example)
shows the common case.

`$extends` (plural) is accepted as a synonym, for familiarity with the `extends`
key of tsconfig, ESLint and Docker Compose; it behaves identically. A node may
use `$extend` or `$extends`, not both. Documents should prefer `$extend`, whose
imperative form matches `$splice`.

Every reference must resolve to an object or to `null`, which is treated as an
empty object so that an optional base can be left empty on purpose. Any other
value, including one that fails to resolve, is an error.

`$extend` departs from the JSON merge patch specification (RFC 7396) in three
places. RFC 7396 silently replaces a target of any other type with an empty
object; this specification rejects them, since a referenced array or string
almost always means a reference pointing to the wrong place. When several
references are combined, a `null` among them is a value rather than a deletion,
as described below. Finally, a `null` that reaches the node's own keys through a
reference is a value too: only a `null` written in the document deletes (see
[Literal `null`](#literal-null)).

The merge itself closely follows RFC 7396 and applies recursively at every key
according to the table below:

| At a key, the referenced object has | The node has    | Result                            | RFC 7396     |
| ----------------------------------- | --------------- | --------------------------------- | ------------ |
| an object                           | an object       | the two are merged recursively    | as specified |
| any value (including `null`)        | no such key     | the referenced value is kept      | as specified |
| any value, or no such key           | `null`          | the key is absent from the result | as specified |
| any value, or no such key           | any other value | the node's value is used          | as specified |

`null` means "remove this key" only when the node supplies it. A `null` inside
the referenced object is an ordinary value, kept unless the node overrides it.

A `null` the node supplies is never kept, even where there is nothing to remove.
In the last row, the node's value is used with any `null` inside its objects
dropped, but not inside its arrays, exactly as RFC 7396 does. Given `base.json`:

```json
{ "name": "svc" }
```

then:

```json
{ "$extend": "./base.json", "cache": { "ttl": null, "size": 10 } }
```

resolves to:

```json
{ "name": "svc", "cache": { "size": 10 } }
```

`base.json` has no `cache`, so the node's `cache` is used, but without its
`null`.

Arrays are ordinary values here, matched by the last row. If the referenced
object holds `{"hosts": ["a", "b"]}` and the node holds `{"hosts": ["c"]}`, the
merged result is `{"hosts": ["c"]}`. Array contents are never inspected and
never combined, matching RFC 7396.

When `$extend` lists several references, they are first combined into a single
referenced object, left to right: objects merge recursively, and any other value
from a later reference — `null` included — replaces the earlier one. The node's
own keys are then applied by the table above. A document therefore overrides
everything it extends. As mentioned above, only the node can delete a key. See
also [Literal `null`](#literal-null).

### `$splice`

`$splice` replaces the node that contains it with the elements of one or more
referenced arrays. It is legal only on an object that is itself an element of an
array, and its value is a reference string or an array of references.

```json
{
	"middleware": [
		{ "$splice": "./middleware/common.json#/middleware" },
		{ "type": "compress" }
	]
}
```

If the referenced array holds two elements, `middleware` above resolves to
three.

- Every reference must resolve to an array, or to `null`, which is treated as an
  empty array. Anything else is an error.
- Several references are spliced in the order listed, so
  `{"$splice": ["./a.json#/x", "./b.json#/y"]}` inserts both arrays' elements,
  one after the other.
- An empty array or `null` contributes no elements, so a `$splice` whose
  references all resolve to either removes the node.
- Like `$ref`, a node containing `$splice` may contain no other key except
  `$comment`, which is dropped along with the node. Once the node is replaced by
  referenced elements, there is nothing left to attach other keys to.

To insert a referenced array as a _single_ element instead, use [`$ref`](#ref).

Splicing is the only way to combine arrays, and has no counterpart in RFC 7396,
which never combines them. To modify one element of a referenced array rather
than take the whole sequence, reference that element with `$extend` — its target
is an object, so sibling keys are allowed:

```json
{
	"middleware": [
		{
			"$extend": "./middleware/common.json#rateLimit",
			"perMinute": 600
		}
	]
}
```

### `$defs`

`$defs` is the conventional place for fragments a document references only
within itself:

```json
{
	"tasks": [
		{ "$splice": "#/$defs/commonTasks" },
		{ "$extend": "#/$defs/deploy", "env": "staging" }
	],
	"$defs": {
		"commonTasks": [{ "run": "git checkout" }, { "run": "npm ci" }],
		"deploy": { "run": "./deploy.sh" }
	}
}
```

resolves to:

```json
{
	"tasks": [
		{ "run": "git checkout" },
		{ "run": "npm ci" },
		{ "run": "./deploy.sh", "env": "staging" }
	],
	"$defs": {
		"commonTasks": [{ "run": "git checkout" }, { "run": "npm ci" }],
		"deploy": { "run": "./deploy.sh" }
	}
}
```

The name is borrowed from JSON Schema. `$defs` is retained in the resolved
output; consumers are expected to ignore it. Other documents may reference into
it, as they may into any part of the output.

A fragment in `$defs` may carry an `$anchor`, but it exists to be copied, and
the original stays in place. Copying an anchored fragment within its own
document therefore puts the name on two nodes, which is a
[collision](#anchor-scope-and-collisions), unless the copy [renames](#renaming)
or [removes](#removing-a-name) it:

```json
{
	"tasks": [
		{ "$anchor": "deployStaging", "$extend": "#deploy", "env": "staging" }
	],
	"$defs": { "deploy": { "$anchor": "deploy", "run": "./deploy.sh" } }
}
```

Another document may reference the fragment by its anchor without renaming it.
Only the copy reaches that document, unless it also imports the `$defs` that
holds the original.

A fragment can also be left unnamed, with a name declared on each copy where it
is made:

```json
{ "$anchor": "deploy", "$extend": "#/$defs/deploy", "env": "staging" }
```

### `$comment`

`$comment` holds a string that this specification ignores and that consumers of
the resolved document are expected to ignore. JSON has no comment syntax, and a
format assembled from many hand-edited files needs somewhere to explain itself.

```json
{
	"$comment": "Ports below 1024 require root; see the deployment runbook.",
	"server": { "port": 8080 }
}
```

It is legal on any object, and is retained in the resolved output — so a comment
written in one document survives into every document that extends it. The one
exception is a `$comment` beside `$ref` or `$splice`, the only key those allow:
it explains the reference, and is dropped along with the node that the reference
replaces.

```json
{
	"limits": {
		"$comment": "Shared with the batch workers; change both together.",
		"$ref": "./limits.json"
	}
}
```

### `$schema`

`$schema` holds a string, conventionally the URI of a JSON Schema, that editors
use to validate and complete the document as it is written. Many configuration
formats already place it at the top of their files, and it would otherwise be an
error here.

```json
{
	"$schema": "./service.schema.json",
	"$extend": "./service.base.json",
	"server": { "tls": true }
}
```

This specification ignores it. Its value is not a reference and is never
resolved or retrieved. It is legal on any object, but a reference never delivers
one: any `$schema` within a referenced value, at any depth, is dropped, whether
the reference points into another document or the same one. Each file names the
schema its own editors should use, and the output keeps only the ones the
top-level document wrote outside referenced values, or none if it wrote none.

### Choosing a directive

| To                                            | Use                  |
| --------------------------------------------- | -------------------- |
| take a value exactly as it is                 | [`$ref`](#ref)       |
| build on an object and override parts of it   | [`$extend`](#extend) |
| insert an array's elements into another array | [`$splice`](#splice) |

Where `$extend` and `$ref` would both work — a referenced object with no sibling
keys — they produce the same result, since merging with nothing is replacement.

In an array element, `$splice` inserts a referenced array's elements, while
`$ref` inserts the array itself as one element; see
[Splicing versus nesting](#splicing-versus-nesting).

### Examples

#### Composing a list

Given `middleware/common.json`:

```json
{
	"middleware": [
		{ "$anchor": "requestId", "type": "requestId", "header": "X-Request-Id" },
		{
			"$anchor": "rateLimit",
			"type": "rateLimit",
			"perMinute": 60,
			"burst": 10
		}
	]
}
```

A document may take the sequence whole and append to it:

```json
{
	"middleware": [
		{ "$splice": "./middleware/common.json#/middleware" },
		{ "type": "compress", "level": 6 }
	]
}
```

```json
{
	"middleware": [
		{ "$anchor": "requestId", "type": "requestId", "header": "X-Request-Id" },
		{
			"$anchor": "rateLimit",
			"type": "rateLimit",
			"perMinute": 60,
			"burst": 10
		},
		{ "type": "compress", "level": 6 }
	]
}
```

Note that the referenced anchors survive into the result, so a later document
can still address `#rateLimit`.

Or it may select entries individually, ordering them itself and overriding one
along the way:

```json
{
	"middleware": [
		{ "$extend": "./middleware/common.json#requestId" },
		{ "$extend": "./middleware/common.json#rateLimit", "perMinute": 600 },
		{ "type": "compress", "level": 6 }
	]
}
```

The overridden entry keeps `burst`, because only `perMinute` was replaced.

#### Splicing versus nesting

`$splice` and `$ref` against the same array, in one document. Given
`hosts.json`:

```json
{ "primary": ["a.example.com", "b.example.com"] }
```

then:

```json
{
	"spliced": [{ "$splice": "./hosts.json#/primary" }, "c.example.com"],
	"nested": [{ "$ref": "./hosts.json#/primary" }, "c.example.com"]
}
```

resolves to:

```json
{
	"spliced": ["a.example.com", "b.example.com", "c.example.com"],
	"nested": [["a.example.com", "b.example.com"], "c.example.com"]
}
```

#### Precedence and removal

Given `defaults.json`:

```json
{ "cache": { "enabled": true, "ttl": 300, "maxEntries": 5000 }, "debug": false }
```

and `overlay.json`:

```json
{ "cache": { "ttl": 60 }, "debug": true }
```

then:

```json
{
	"$extend": ["./defaults.json", "./overlay.json"],
	"cache": { "maxEntries": null }
}
```

resolves to:

```json
{ "cache": { "enabled": true, "ttl": 60 }, "debug": true }
```

Three rules at once: references are combined in order, so `overlay.json` wins on
`ttl` and `debug`; the node's own keys are applied last of all; and because the
`null` is written in the node, it removes `maxEntries` rather than setting it.
Had `overlay.json` held that `null` instead, the result would keep
`"maxEntries": null`.

## Part 2 — Resolving documents

### Resolution order

Resolving a document means producing a single JSON value from it:

1. Parse the document.
2. Resolve every node, as described below.
3. Verify that `$anchor` values are unique across the result.

A node's value depends on its children and on whatever its directives reference:

- A `$ref` node resolves to its target.
- An `$extend` node resolves to the values its references select, merged with
  its own resolved keys.
- An array resolves to its resolved elements, with each `$splice` node replaced
  by the elements of the arrays it references.
- Any other node resolves to itself, with each child resolved.

A node cannot resolve before its children, so resolution proceeds innermost
first.

#### What a reference needs

A reference depends on as little of the referenced document as possible. Three
rules decide exactly how much:

1. **The target is resolved completely.** It is copied into the result, so all
   of it is needed.
2. **Reaching the target resolves only the path to it.** A pointer is followed
   one step at a time, and each step resolves only enough to find the next key
   or index:
   - In an object containing `$extend`, the step needs the values its references
     select and the object's own key for that step, but not its other keys.
   - In an array, the step needs the values its `$splice` elements reference,
     which decide where every element falls, but not the other elements.
   - At a `$ref` node, the step continues in its target.
   - Any other node is stepped into directly.
3. **A node needed while it is still being resolved is a cycle**, and an error
   (see [Cycles and memoization](#cycles-and-memoization)).

Pointers therefore address the resolved document. An array index counts elements
after splicing, and a key may be one the object inherited through `$extend`. The
rest of the referenced document is left alone, and need not resolve at all for
the reference to succeed.

The rules apply to every reference, whether it points into the same document or
another one.

Given `service.base.json` from the [Example](#example):

```json
{
	"$extend": "./service.base.json",
	"server": { "tls": true },
	"healthcheck": { "port": { "$ref": "#/server/port" } }
}
```

`#/server/port` steps into the root, which contains `$extend`. That step needs
`service.base.json` and the root's own `server` key, `{"tls": true}`, but not
`healthcheck`. Their merge has `port` 8080, which the reference selects. The
document references a value it inherited, from inside itself, without a cycle.

An array step counts elements after splicing, and parts off the path are never
touched. Given `hosts.json` from
[Splicing versus nesting](#splicing-versus-nesting), and `app.json`:

```json
{
	"hosts": [{ "$splice": "./hosts.json#/primary" }, "c.example.com"],
	"fallback": { "$ref": "#/hosts/2" },
	"legacy": { "$ref": "./missing.json" }
}
```

`#/hosts/2` steps into `hosts`, an array, so it needs the value the `$splice`
references. That value has two elements, so `"c.example.com"` falls at index
two. Another document may reference `./app.json#/fallback` and succeed, since
nothing on that path needs `legacy`. Resolving `app.json` itself needs every
node, so it reports that `missing.json` does not exist.

Two documents may also reference each other. Given `a.json`:

```json
{ "name": "a", "peer": { "$ref": "./b.json#/name" } }
```

and `b.json`:

```json
{ "name": "b", "peer": { "$ref": "./a.json#/name" } }
```

`a.json`'s `peer` needs only `/name` from `b.json`, which holds no directive,
and never `b.json`'s own `peer`. Each document resolves to its own name and the
other's.

#### Finding an anchor

A resolver finds the node an anchor names in two passes:

1. It looks among the anchors written in the referenced document.
2. If none of them has the name, it resolves the document's references to
   _other_ documents, deferring those within the document, and looks among the
   anchors their values bring in. The anchored node then resolves like any other
   node at its position, merged with whatever the document's own keys supply
   there. If that merge renames or removes the anchor, the name is not found.

A name can arrive no other way. A copy made by a reference within the document
would [collide](#anchor-scope-and-collisions) with its original, so deferring
those references loses nothing. If resolving the references to other documents
needs the node making the lookup, that is a cycle, and if any of them fails to
resolve, so does the lookup.

Neither pass changes a successful result. A written anchor and an imported one
with the same name are a collision either way.

Given `middleware/common.json` from [Composing a list](#composing-a-list):

```json
{
	"middleware": [{ "$splice": "./middleware/common.json#/middleware" }],
	"limit": { "$ref": "#rateLimit/perMinute" }
}
```

No `rateLimit` anchor is written in the document, so the second pass resolves
the `$splice`, which brings one in, and `limit` resolves to 60. The `$ref` is
deferred, so it is never needed while it is being resolved.

A lookup only finds the node's position. What the reference selects is the
resolved node at that position, which may differ from the node that carried the
name. Given `common.json`:

```json
{ "limits": { "$anchor": "limits", "cpu": 2, "memory": 1 } }
```

then:

```json
{
	"$extend": "./common.json",
	"limits": { "cpu": 4 },
	"jobCpu": { "$ref": "#limits/cpu" }
}
```

resolves `jobCpu` to 4, not 2. The second pass finds `limits` on the node
`common.json` brings in at `/limits`. The node at `/limits` is that one merged
with the document's own `{"cpu": 4}`.

### Literal `null`

RFC 7396 gives `null` exactly one meaning: delete the key. A merge patch cannot
set a value _to_ `null`. This specification keeps that meaning only where
something is merged. A `null` deletes a key when it is all of these:

- **written** in the document, not delivered by a reference;
- **under an `$extend` node**, as the value of a key in that node or in an
  object nested inside it;
- **not inside an array**, since arrays are never merged.

Every other `null` is an ordinary value, and appears in the output.

#### Outside `$extend`, a `null` is a value

Nothing is merged outside an `$extend` node, so there is nothing to delete. In
this document, `proxy` stays `null`:

```json
{ "service": { "$extend": "./service.base.json" }, "proxy": null }
```

#### Inside an array, a `null` is a value

An array is replaced whole, never merged (see [`$extend`](#extend)), so a `null`
inside it is kept, even under an `$extend` node. This matches RFC 7396.

```json
{ "$extend": "./base.json", "hosts": [null, { "a": null }] }
```

resolves `hosts` to `[null, {"a": null}]`, whatever object `base.json` holds.

#### A referenced `null` is a value

A reference delivers values, never deletions. A `$ref` replaces its node
outright, so a `$ref` to `null` sets the node to `null`, even under an `$extend`
node. Given `defaults.json` from
[Precedence and removal](#precedence-and-removal), and `nothing.json` containing
only `null`:

```json
{ "$extend": "./defaults.json", "cache": { "$ref": "./nothing.json" } }
```

resolves to `{"cache": null, "debug": false}`, not to a document without
`cache`.

Likewise, a `null` inside a referenced document is a value, however deeply it is
nested, and whether that document is the first reference in an `$extend` list or
the last. A referenced value is resolved completely, with its own deletions
applied, before it is delivered.

A reference that resolves to `null` as a whole is different: `$extend` treats it
as an empty object and `$splice` as an empty array, so it contributes nothing.

#### A written `null` deletes at every level

A written `null` is not used up by the nearest `$extend`. It deletes from every
`$extend` node it is written under, and only then is it dropped. Given
`base.json`:

```json
{ "cache": { "ttl": 300, "size": 1 } }
```

and `lru.json`:

```json
{ "ttl": 60, "mode": "lru" }
```

then:

```json
{
	"$extend": "./base.json",
	"cache": { "$extend": "./lru.json", "ttl": null }
}
```

resolves to `{"cache": {"size": 1, "mode": "lru"}}`. The `null` removes `ttl`
from `lru.json` and, one level up, from `base.json`. Since nodes resolve
innermost first, a resolver must carry the deletion out of the inner merge until
every `$extend` above it has applied it.

A written `null` with nothing to delete is dropped all the same (see
[`$extend`](#extend)).

### Anchor scope and collisions

Each document has its own namespace. When `a.json` imports `b.json`, the names
in `b.json` belong to `b.json` while it resolves. Only the names inside the
**selected fragment** travel into `a.json`, where they must not collide with
names already there.

Using `roster.json` from [`$anchor`](#anchor):

| Reference                        | Selected value       | Does `oncall` travel? |
| -------------------------------- | -------------------- | --------------------- |
| `./roster.json#oncall/members/1` | `"bo"`               | no                    |
| `./roster.json#oncall`           | the `primary` object | yes                   |
| `./roster.json`                  | the whole document   | yes                   |

So `a.json` may declare its own `$anchor` of `oncall` in the first case, but not
in the other two.

The same holds for every directive that selects a fragment. Anchors inside the
elements that `$splice` inserts travel into the containing document, exactly as
those inside an object merged by `$extend` do.

#### Same-document references

A reference within a document copies the selected value, anchors included, and
the original stays where it is. Copying an anchored node within its own document
therefore always produces two nodes with the same name:

```json
{
	"primary": { "$anchor": "oncall", "members": ["ana", "bo"] },
	"backup": { "$ref": "#oncall" }
}
```

is an error, because `primary` and `backup` both carry `oncall`. To reuse such a
node, use `$extend` to give the copy a name of its own (see
[Renaming](#renaming)) or none at all (see [Removing a name](#removing-a-name)),
or move the node, without its anchor, into [`$defs`](#defs) and reference it by
pointer.

#### `$anchor` does not affect merging

`$anchor` is an address, not a merge key. It has no influence on what merges
with what: nodes merge because they occupy the same position, never because they
share a name.

Two nodes at the **same** position merge into one, which carries a single name.
This is not a collision, because only one node exists in the result. Given
`base.json`:

```json
{ "limits": { "$anchor": "resourceLimits", "cpu": 2 } }
```

then:

```json
{ "$extend": "./base.json", "limits": { "memory": "1Gi" } }
```

resolves to:

```json
{ "limits": { "$anchor": "resourceLimits", "cpu": 2, "memory": "1Gi" } }
```

Two nodes at **different** positions that share a name are a collision, and an
error, because the name no longer identifies one node:

```json
{
	"$extend": "./base.json",
	"requests": { "$anchor": "resourceLimits", "cpu": 1 }
}
```

Some formats do use a named key to pair up array elements during a merge —
Kubernetes' strategic merge patch is the best known. This specification does
not. Arrays are never merged element by element, and `$anchor` exists only so
that a reference can name a node.

#### Renaming

Because a node's own keys merge last, a local `$anchor` overrides an imported
one:

```json
{
	"$anchor": "rateLimitInternal",
	"$extend": "./middleware/common.json#rateLimit",
	"perMinute": 6000
}
```

This, or [removing the name](#removing-a-name), is how to import the same named
node twice. Both work only with `$extend`, since `$ref` and `$splice` permit no
sibling keys other than `$comment`.

#### Removing a name

A written `"$anchor": null` deletes an imported name, as any written `null`
under an `$extend` deletes a key (see [Literal `null`](#literal-null)). The copy
keeps its contents but loses the name, so a node can be reused within its own
document:

```json
{
	"primary": { "$anchor": "oncall", "members": ["ana", "bo"] },
	"backup": { "$extend": "#oncall", "$anchor": null }
}
```

resolves `backup` to `{"members": ["ana", "bo"]}`, with no collision.

A `null` value is valid only where a written `null` deletes: among the keys of
an `$extend` node, at any depth, but not inside an array. Anywhere else it is an
error. Where there is no name to delete, it is dropped like any other written
`null`.

Each `"$anchor": null` removes the name at its own position only. An anchor
deeper in the copy needs its own, such as `"inner": {"$anchor": null}`. An
anchor inside an array cannot be removed, since arrays are never merged.

### Base URIs, schemes, and encoding

A reference that omits a scheme is a relative reference, resolved per RFC 3986
§5 against the **base URI of the document containing it**; that is, the URI from
which that document itself was loaded.

A document loaded from `file:///srv/app/config/staging.json` therefore resolves
`./common/rate-limit.json` to `file:///srv/app/config/common/rate-limit.json`,
while the same reference in a document loaded from
`https://example.com/config/staging.json` resolves to
`https://example.com/config/common/rate-limit.json`. A relative reference in a
remote document stays remote.

Nothing in a document can change its base URI. There is no equivalent of JSON
Schema's `$id`. A document retrieved through a redirect, however, has as its
base URI the last URI used, per RFC 3986 §5.1.3.

A document that was not loaded from a URI — read from standard input, or
supplied as a string — has no base URI unless the caller provides one. Its
fragment-only references (`#...`) still resolve, but any other relative
reference is an error.

This specification places no restriction on the scheme. An implementation may
support any subset — commonly `file` alone — and **must report an error for a
scheme it does not support** rather than ignoring the reference or falling back
to another scheme.

An implementation **must not retrieve a reference over a network** — `http`,
`https`, or any other remote scheme — unless its user has explicitly enabled it.
A reference using a scheme that has not been enabled is an error, like one using
a scheme that is not supported at all. The default matters because a remote
document's own relative references stay remote, so one enabled reference can
fetch many more.

A document retrieved from a remote location **must not reference a local
resource** — a `file` URI, or any other scheme that reads from the machine doing
the resolving. Its relative references already stay remote, so the rule governs
its absolute ones. Without it, a remote document could read local files and
carry their contents into the resolved output. The restriction runs one way
only: a local document may reference a remote one, subject to the rule above.

Likewise, a document retrieved with protection in transit — over `https`, say —
**must not reference a resource retrieved without it**, such as over plain
`http`. A resolved document is only as trustworthy as its least protected part,
so one unprotected reference would let anyone on the network alter it. This
restriction also runs one way: a document retrieved without protection may
reference one retrieved with it.

A redirect is held to the same rules as a reference: it may not lead from a
remote resource to a local one, nor from a protected resource to an unprotected
one.

Reference paths are URI paths. They use `/` as the separator on every platform,
and are percent-decoded per RFC 3986 before use. An implementation reading from
a filesystem is responsible for converting between file URIs and native paths.
On Windows in particular, `C:\app\base.json` is not a valid URI: the separator
is wrong, and `C:` parses as a scheme. Use a relative reference, or a full
`file:///C:/...` URI.

Reference tokens within a fragment use RFC 6901 escaping — `~1` for `/` and `~0`
for `~` — and are additionally percent-encoded where RFC 6901 §6 requires. In
particular, `#` may appear only as the fragment delimiter, so a `#` within a
path or a key is written `%23`.

### Cycles and memoization

Resolved values are cached by `(resolved URI, pointer)`. An anchor reference is
first converted to the JSON Pointer of the node it names, so `#oncall` and
`#/primary` share one entry. Without the cache, a document imported along more
than one path is resolved once per path, and when such imports are layered the
number of paths — and the work — grows exponentially.

A node needed while it is still being resolved is a cycle. Detection uses the
same key as the cache, which catches all three shapes:

```json
{ "a": { "$extend": "#/b" }, "b": { "$extend": "#/a" } }
```

```json
{ "$anchor": "x", "tasks": [{ "$extend": "#x" }] }
```

```text
a.json#/p  ->  b.json#/q  ->  a.json#/p
```

In the first, `a` needs all of `b`, and `b` needs all of `a`. The second is
worth noting: resolving `x` requires resolving its own descendant, which
requires `x`. It is not obviously a cycle when read.

Because detection is per node rather than per document, two documents may
reference each other, as in [What a reference needs](#what-a-reference-needs),
so long as no node needs itself.

The rules in [What a reference needs](#what-a-reference-needs) decide exactly
what is a cycle, so that every resolver agrees. The cost is that they reject a
few documents a more elaborate resolver could handle:

```json
{ "a": { "$extend": "#/b", "k": 1 }, "b": { "x": { "$ref": "#/a/k" } } }
```

`a` needs all of `b`. Then `b`'s `x` steps into `a`, an object containing
`$extend`, and that step needs all of `b` again before `k` can be found, even
though `k` is `a`'s own key. A resolver must report this as a cycle, even one
that could resolve it.

A cycle has no JSON representation: it could only resolve to a value that
contains itself, and JSON is a tree. A resolver that produces JSON must
therefore report every cycle as an error.

An implementation may instead resolve into an in-memory graph, in which a `$ref`
inside a larger value points back at an enclosing one, as [JsonRef](#prior-art)
does. This is discouraged. The result cannot be written back out as JSON, which
is the point of this specification, and a document that relies on it resolves
only with that implementation. Some cycles cannot resolve even then: those
formed through `$extend` or `$splice`, which need the complete referenced value
before they can produce their own, and loops of `$ref` nodes that refer only to
one another.

### Errors

Resolution reports an error only in a part of a document that it needs (see
[What a reference needs](#what-a-reference-needs)). A broken reference or an
unknown directive in a part of a referenced document that nothing needs is not
reported, though it is when that document is resolved itself. A referenced
document must still parse, so invalid JSON and duplicate keys are errors
wherever they occur.

- A document that is not valid JSON, including a referenced resource that is not
  a JSON text
- An object with duplicate keys, in any document
- A reference that cannot be resolved, including one using a scheme the
  implementation does not support or the user has not enabled
- A relative reference, other than a fragment-only one, in a document with no
  base URI
- A reference from a remote document to a local resource
- A reference from a document retrieved with protection in transit to a resource
  retrieved without it
- A reference cycle, unless the implementation resolves into a graph (see
  [Cycles and memoization](#cycles-and-memoization)); the error should name the
  full chain
- Duplicate `$anchor` within a resolved value; the error should give both paths,
  and say when one of them was imported
- A value referenced by `$extend` that is neither an object nor `null`
- A value referenced by `$splice` that is neither an array nor `null`
- `$splice` on an object that is not an element of an array
- Any key alongside `$ref` or `$splice` other than `$comment`, including
  `$anchor`
- A node containing both `$extend` and its synonym `$extends`
- An `$extend` or `$splice` given an empty array of references
- A directive whose value is malformed or of the wrong type — an `$anchor` name
  that does not match its pattern, a `null` `$anchor` where a written `null`
  does not delete, a `$ref` given an array, or a reference whose fragment does
  not match `cj-fragment` (see [References](#references))
- A `$`-prefixed key defined by neither this specification nor the host format
  (see [Host directives](#host-directives)), other than the reserved `$id`

### Security considerations

Resolving a document means retrieving every resource it references, so a
document from an untrusted source can make a resolver do more than its author
appears to ask. The rules in
[Base URIs, schemes, and encoding](#base-uris-schemes-and-encoding) close the
gravest gaps: network retrieval is off unless enabled, a remote document cannot
read local resources, and a protected document cannot draw on unprotected ones.
Beyond those, this specification leaves the remedies to implementations, which
should consider the following.

- **Internal network addresses.** Once network retrieval is enabled, a document
  can reference any host the resolver can reach, including ones its author could
  not reach directly, such as services on an internal network or a cloud
  provider's metadata endpoint. A resolver should let its user limit which hosts
  may be retrieved.
- **Local files.** A local document can reference any file the resolver can
  read, and any file containing JSON, credentials included, can be carried into
  the output. A resolver should let its user confine retrieval to particular
  directories, including against `..` segments and links that lead elsewhere.
- **Resource exhaustion.** Because a value may be referenced any number of
  times, a small document can resolve to an exponentially larger one, and a long
  chain of references can nest arbitrarily deeply. A resolver should limit the
  size of its output, the depth of resolution, the number of resources retrieved
  and the size of each, and treat exceeding a limit as an error.

## Part 3 — Background

### Versioning

This specification is versioned according to
[Semantic Versioning 2.0.0](https://semver.org/):

- A **patch** release makes editorial changes only. No document resolves
  differently.
- A **minor** release may make documents valid that were previously errors —
  most often by defining a new directive. Because a resolver rejects any
  `$`-prefixed key it does not recognize, a document valid under an earlier
  minor release resolves identically under a later one. The exception is the
  reserved `$id`, which is ignored rather than rejected (see
  [Host directives](#host-directives)).
- A **major** release may change how a previously valid document resolves.

A version with a pre-release suffix, such as `1.0.0-draft`, may still change
incompatibly.

Documents do not declare a version. An implementation states the version it
implements, and a host format states the version it builds on.

### Media type

A Composable JSON document is JSON, and may be served or stored as
`application/json` without loss. Where the distinction matters — chiefly so that
fragment semantics have an owner — the media type is:

```text
application/composable+json
```

The `+json` structured syntax suffix (RFC 6839) signals that generic JSON
tooling can parse the document without understanding this specification.

This name is not registered with IANA and is therefore in the standards tree
without authority to be there. An implementation needing a registered name today
should use the vendor tree — `application/vnd.<vendor>.composable+json` — per
RFC 6838.

Composable JSON documents conventionally use the `.json` file extension.

### Relationship to existing standards

The format is assembled from existing specifications wherever one fits.

| Concept              | Standard                       | Deviation                                                        |
| -------------------- | ------------------------------ | ---------------------------------------------------------------- |
| References           | RFC 3986 (URI Reference)       | None                                                             |
| Node addressing      | RFC 6901 (JSON Pointer)        | Adds `$anchor` names as an alternative fragment form             |
| Anchors              | JSON Schema `$anchor`          | Same keyword and name; fragment may add a pointer; no base URIs  |
| Replacement          | JSON Reference (expired draft) | Sibling keys other than `$comment` are an error, not ignored     |
| Merge semantics      | RFC 7396 (JSON Merge Patch)    | Only a written `null` deletes; targets must be objects or `null` |
| Combining references | RFC 7396 (JSON Merge Patch)    | `null` is a value, not a deletion                                |
| Array splicing       | none                           | `$splice` has no standard counterpart                            |
| Escaping             | RFC 6901 §3 (`~0`, `~1`)       | None                                                             |

JSON Schema and JSON-LD both solve document identity and reference, but their
machinery is far larger than this needs. Rather than subset either one, this
specification extends JSON Pointer with a single idea — a node may name itself —
and reuses JSON Merge Patch, with the deviations above, for everything else.

JSON Schema's `$id`, which establishes a base URI for a schema resource, has no
counterpart here. `$anchor` alone provides all the naming this format needs, and
in consequence a document's base URI is always the location it was loaded from
and cannot be moved by anything the document says.

[JsonRef](http://jsonref.org/) v0.4.0 made the same break with JSON Schema
independently, and its `$id` has exactly the role `$anchor` has here: it names a
node without moving the base URI, and its fragment forms are the ones above.
This specification keeps `$anchor` because JSON Schema, by far the more widely
used of the two, gives `$id` base-URI semantics — enough that JsonRef itself
must allow a root `$id` to be an absolute URI. `$id` is nonetheless reserved, so
that a later version may give it a meaning without conflicting with any host
format.

`$ref` itself originates in the JSON Reference draft, which specified that any
members other than `$ref` "SHALL be ignored". JSON Schema draft-07 and OpenAPI
3.0 inherited that rule; JSON Schema 2019-09 and later reversed it, applying
such members in addition. This specification does neither: sibling keys are an
error, so that a document written against either interpretation fails loudly
rather than behaving in a way its author did not intend. `$comment` alone is
allowed, since it means the same thing under every interpretation.

#### Composable JSON cannot compose JSON Schema

Borrowing five keywords from JSON Schema invites the assumption that this format
can be used to assemble schema documents from parts. It cannot, for three
independent reasons.

**`$ref` means something different.** In JSON Schema it is an applicator
evaluated during validation: the referenced schema's assertions apply in
addition to those of the schema containing it. Here it is a substitution
performed before any consumer sees the document. Sibling keys, which JSON Schema
2019-09 and later permit alongside `$ref`, are an error here.

**Recursive schemas become cycle errors.** A schema that refers to itself is the
ordinary way to describe a tree, and is unremarkable in JSON Schema. To this
specification it is a reference cycle, which cannot resolve to JSON at all,
since substitution cannot terminate.

**Structural merging is not schema composition.** JSON Schema composes with
`allOf`, `anyOf` and `oneOf`, which are logical operators. Merging
`{"maximum": 5}` with `{"maximum": 10}` yields `{"maximum": 10}`, whereas an
`allOf` of the two requires both, so the effective maximum stays 5. A merge of
two schemas is a different schema, not their conjunction — and it fails quietly,
producing a valid schema that means something else.

JSON Schema reached the same conclusion. Draft-03 had an `extends` keyword
performing structural inheritance; draft-04 removed it and introduced `allOf` in
its place. Structural merging is a reasonable primitive for configuration and a
poor one for schemas, and only the first is this format's concern.

Separately, `$vocabulary`, `$dynamicRef` and `$dynamicAnchor` are errors here,
being `$`-prefixed keys this specification does not define — unless a host
format declares them as its own. `$id` is reserved and, in this version,
ignored. `$schema` is accepted only as a note for editors: it does not make the
document a schema.

### Prior art

Several existing tools solve part of this problem. None was adopted, for the
reasons below.

| Tool                              | Cross-document | Sub-document selection | Named nodes           | Arrays                |
| --------------------------------- | -------------- | ---------------------- | --------------------- | --------------------- |
| Composable JSON                   | any node       | pointer and anchor     | yes                   | replace, splice       |
| json-merger (JavaScript)          | any node       | pointer and JSONPath   | no; positional scopes | eight operations      |
| conflate (Go)                     | root only      | no                     | no                    | merge                 |
| `extends` in tsconfig, ESLint, CI | root only      | no                     | no                    | replace               |
| YAML anchors and merge keys       | no             | n/a                    | yes                   | n/a                   |
| JSON Schema `$ref` / `$anchor`    | yes            | pointer and anchor     | yes                   | n/a; no merging       |
| JsonRef v0.4.0 `$ref` / `$id`     | yes            | pointer and `$id`      | yes                   | n/a; no merging       |
| Kustomize                         | bases only     | no                     | no                    | merge by declared key |
| Jsonnet, Dhall, CUE, Pkl          | imports        | no                     | n/a; variables        | language operators    |

[JsonRef](http://jsonref.org/) is the closest prior design for addressing. Its
`$id` is this specification's `$anchor`, its fragment forms are the same, and it
too refuses to let a name move the base URI. It differs in three ways: keys
alongside `$ref` are ignored rather than an error; cycles are permitted, because
its output is a graph rather than JSON; and merging is explicitly out of scope,
which leaves out the half of the problem this specification exists to solve.

[json-merger](https://github.com/boschni/json-merger) is the closest existing
design overall, and arrived independently at the same reference syntax —
`a.json#/someArray/0`. It was not adopted because it is JavaScript, and because
its scope is far larger: fourteen operations, including embedded expressions.
Its one structural difference is instructive. To address a particular node it
offers positional scopes — `$source`, `$target`, `$parent`, `$root` — rather
than names, so a reference breaks when the surrounding document is reorganized.
Stable names are the reason `$anchor` exists here.

[conflate](https://github.com/miracl/conflate) is the nearest equivalent in Go:
it merges JSON, YAML and TOML from paths and URLs, and validates against JSON
Schema. It was not adopted because its includes are a root-level list, with no
way to select part of a document and no way for a nested node to compose from a
base. Other Go libraries in this area — `deepmerge`, `go-jsonmerge`,
`go.uber.org/config` — are merge primitives with no reference layer at all.

YAML anchors and merge keys (`&name`, `*name`, `<<:`) are the conceptual
ancestor of `$anchor`: a node names itself, and others merge it in. They are
strictly document-scoped and cannot cross files.

Configuration languages — Jsonnet, Dhall, CUE, Pkl, HOCON — express all of this
and a great deal more. Adopting one means adopting a language and its toolchain,
and documents cease to be ordinary JSON. That trade is not worth making for a
format whose entire job is composition.

What none of them combine is cross-document merging with named nodes. JSON
Schema and JsonRef name nodes but never merge; YAML merges and names but cannot
cross files; json-merger crosses documents and merges but addresses
positionally; the configuration languages arrive by becoming languages. That
intersection is the only genuinely new thing here, and it exists so that a
reference can survive the document it points into being reorganized.

### Normative references

- [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) — Uniform Resource
  Identifier (URI): Generic Syntax
- [RFC 5234](https://www.rfc-editor.org/rfc/rfc5234) — Augmented BNF for Syntax
  Specifications: ABNF
- [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901) — JavaScript Object
  Notation (JSON) Pointer
- [RFC 7396](https://www.rfc-editor.org/rfc/rfc7396) — JSON Merge Patch
- [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) — The JavaScript Object
  Notation (JSON) Data Interchange Format
- [JSON Schema Core 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)
  — source of `$anchor`, `$defs`, `$comment` and `$schema`

### Informative references

- [JsonRef v0.4.0](http://jsonref.org/) — JSON Reference, an individual
  specification. Cited as the closest prior design for named-node addressing.
- [RFC 6838](https://www.rfc-editor.org/rfc/rfc6838) — Media Type Specifications
  and Registration Procedures
- [RFC 6839](https://www.rfc-editor.org/rfc/rfc6839) — Additional Media Type
  Structured Syntax Suffixes
- [RFC 8820](https://www.rfc-editor.org/rfc/rfc8820) — URI Design and Ownership
  (BCP 190). Updates RFC 3986 with guidance for specification authors; it does
  not alter URI syntax.
- [draft-pbryan-zyp-json-ref-03](https://datatracker.ietf.org/doc/html/draft-pbryan-zyp-json-ref-03)
  — JSON Reference. Expired March 2013; never published as an RFC. Cited as the
  origin of `$ref`, not as a current standard.
