# Composable JSON

Composable JSON defines a standard way for JSON documents (RFC 8259) to reuse or
override values from other JSON documents.

Reuse is expressed with _directives_: object keys beginning with `$`, which are
reserved for the purpose. A _resolver_ replaces each directive with the value it
references, producing ordinary JSON. This specification defines six directives,
the syntax of a reference, and how a resolver combines what it references.

## Motivation

JSON has no concept of composition or reuse. There is no include, no import, and
no inheritance; every document stands alone, and anything shared between them
must be restated in each one. That becomes a problem as soon as a set of
documents differ only slightly from each other (e.g. one configuration per
environment, or a suite of test cases sharing a baseline and varying only
slightly). Restating the common part everywhere means a change to a shared value
has to be repeated in every copy, and a copy that drifts gives no sign that it
has.

Other formats grew their own answers to this – YAML anchors, `extends` in
tsconfig and ESLint, Kustomize overlays, and the configuration languages – but
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

`$ref`, `$extend` and `$splice` are consumed during resolution and do not appear
in the output. `$anchor`, `$defs` and `$comment` are retained.

### Host directives

A specification built on this one — a configuration format or a test format, say
— may define `$`-prefixed keys of its own, and must document them. A resolver is
told which those are, and **errors on any `$`-prefixed key that neither it nor
its host format declares.**

Reserving the namespace rather than only the keys defined here is what makes
that last rule possible, and it is worth the cost: a misspelled `$extned`
resolves to a document that inherited nothing and reports no error, which is
among the harder failures to diagnose from the output alone.

Host directives are resolved by the host, not by this specification, and are
therefore neither consumed nor interpreted here. A host should avoid names this
specification might plausibly take later. `$import`, `$merge`, `$delete` and
`$override` have all been considered, and so has `$id`, which may yet replace
`$anchor` (see
[Relationship to existing standards](#relationship-to-existing-standards)).

### References

A reference is a URI reference (RFC 3986) whose fragment identifies a value
within the referenced document.

```abnf
reference    = URI-reference                  ; RFC 3986
fragment     = json-pointer / anchor-path
json-pointer = *( "/" reference-token )       ; RFC 6901
anchor-path  = anchor *( "/" reference-token )
anchor       = ( ALPHA / "_" ) *( ALPHA / DIGIT / "-" / "_" / "." )
```

The character following `#` decides how the URI fragment is read, and so how the
reference is resolved:

- `/` — an RFC 6901 JSON Pointer, resolved from the document root
- end of string, or no fragment at all — the whole document
- anything else — an [`$anchor`](#anchor) name, terminated by `/` or end of
  string, optionally followed by a pointer path relative to the anchored node

| Reference                      | Resolves to                                 |
| ------------------------------ | ------------------------------------------- |
| `./base.json`                  | the whole document                          |
| `./base.json#/staff/members/1` | JSON pointer into that document             |
| `#/middleware/0`               | JSON pointer into the current document      |
| `./base.json#oncall/members/1` | node with `oncall` anchor, then `members/1` |
| `#oncall/members/1`            | the same, within the current document       |

A reference without a scheme resolves against the location the containing
document was loaded from. For the precise rules — base URIs, supported schemes,
percent-encoding and native paths — see
[Base URIs, schemes, and encoding](#base-uris-schemes-and-encoding).

### `$ref`

`$ref` replaces the node that contains it with the value its reference resolves
to. Its value is a single reference string.

The referenced value may be anything: an object, an array, a scalar, or `null`.
A node containing `$ref` may contain no other key, reserved or otherwise.

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
document; see [Anchor scope and collisions](#anchor-scope-and-collisions).

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
it, the node's own keys taking precedence. It is legal on any object, including
one that is an element of an array, and its value is a reference string or an
array of references. The [Overview](#overview) shows the common case.

`$extends` is accepted as a synonym, for familiarity with the `extends` key of
tsconfig, ESLint and Docker Compose; it behaves identically. A node may use
`$extend` or `$extends`, not both. Documents should prefer `$extend`, whose
imperative form matches `$splice`.

Every reference must resolve to an object. Anything else is an error: use
[`$ref`](#ref) to take a value as it is, or [`$splice`](#splice) to insert an
array's elements into an array. That restriction is the only way `$extend`
departs from RFC 7396, which accepts a value of any type.

The merge itself follows RFC 7396, recursively at every key. The referenced
object supplies the first column below, the node supplies the second, and where
they disagree the node wins.

| In the referenced object | In the node     | Result                                       | RFC 7396     |
| ------------------------ | --------------- | -------------------------------------------- | ------------ |
| object                   | object          | merged key by key, recursively               | as specified |
| any                      | key absent      | the referenced value is kept                 | as specified |
| any                      | `null`          | the key is removed                           | as specified |
| any                      | any other value | the node's value replaces the referenced one | as specified |

Arrays are ordinary values here, matched by the last row. If the referenced
object holds `{"hosts": ["a", "b"]}` and the node holds `{"hosts": ["c"]}`, the
merged result is `{"hosts": ["c"]}`. Array contents are never inspected and
never combined, matching RFC 7396.

When `$extend` lists several references, each is applied in order, and the
node's own keys last. A document therefore overrides everything it extends. See
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

- Every reference must resolve to an array; anything else is an error.
- Several references are spliced in the order listed, so
  `{"$splice": ["./a.json#/x", "./b.json#/y"]}` inserts both arrays' elements,
  one after the other.
- An empty array contributes no elements, so a `$splice` whose references all
  resolve to empty arrays removes the node.
- A node containing `$splice` may contain no other key. Once the node is
  replaced by several elements, there is nothing left to attach them to.

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
		"commonTasks": [{ "$anchor": "checkout", "run": "git checkout" }],
		"deploy": { "run": "./deploy.sh" }
	}
}
```

The name is borrowed from JSON Schema. `$defs` is retained in the resolved
output; consumers are expected to ignore it.

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
written in one document survives into every document that extends it. The name
and the role are JSON Schema's.

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

Three rules at once: references are applied in order, so `overlay.json` wins on
`ttl` and `debug`; the node's own keys are applied last of all; and `null`
removes `maxEntries` rather than setting it.

## Part 2 — Resolving documents

### Resolution order

Resolving a document means producing a single JSON value from it:

1. Parse the document.
2. Resolve its `$ref`, `$extend` and `$splice` directives depth first —
   innermost node first. For each reference: resolve the referenced document
   completely, select the fragment, then replace, merge or splice as that
   directive specifies.
3. Verify that `$anchor` values are unique across the result.

Step 2 is recursive: a referenced document is fully resolved, including its own
references, before a fragment is selected from it. A document therefore resolves
after everything it references.

Anchor lookup is resolved on demand. If finding `#oncall` requires some other
node to resolve first, that node is resolved first.

### Literal `null`

RFC 7396 gives `null` exactly one meaning: delete the key. A merge patch cannot
set a value _to_ null. These semantics are adopted as written. In a document
that uses `$extend`, a literal `null` therefore deletes rather than sets; a
document with no imports is unaffected, since nothing is merged.

`$ref` is not subject to this, since replacement is not a merge: a `$ref` whose
referenced value is `null` does indeed set the node to `null`.

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

This is the supported way to import the same named node twice. It works only
with `$extend`, since `$ref` and `$splice` permit no sibling keys.

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
Schema's `$id`.

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

Reference paths are URI paths. They use `/` as the separator on every platform,
and are percent-decoded per RFC 3986 before use. An implementation reading from
a filesystem is responsible for converting between file URIs and native paths.
On Windows in particular, `C:\app\base.json` is not a valid URI: the separator
is wrong, and `C:` parses as a scheme. Use a relative reference, or a full
`file:///C:/...` URI.

Reference tokens within a fragment use RFC 6901 escaping — `~1` for `/` and `~0`
for `~` — and are additionally percent-encoded where RFC 6901 §6 requires.
Because `#` only ever appears as the fragment delimiter, it needs no escape
within a path.

### Cycles and memoization

Resolved values are cached by `(resolved URI, pointer)`. Without this, a deep
chain of imports re-resolves exponentially.

A reference that is already being resolved is a cycle. Detection operates on the
same key, which catches all three shapes:

```json
{ "a": { "$extend": "#/b" }, "b": { "$extend": "#/a" } }
```

```json
{ "$anchor": "x", "tasks": [{ "$extend": "#x" }] }
```

```text
a.json#/p  ->  b.json#/q  ->  a.json#/p
```

The second is worth noting: resolving `x` requires resolving its own descendant,
which requires `x`. It is not obviously a cycle when read.

Because detection is per node rather than per document, two documents may
reference each other so long as no individual node does.

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

- A reference that cannot be resolved, including one using a scheme the
  implementation does not support
- A reference cycle, unless the implementation resolves into a graph (see
  [Cycles and memoization](#cycles-and-memoization)); the error should name the
  full chain
- Duplicate `$anchor` within a resolved value; the error should give both paths,
  and say when one of them was imported
- A value referenced by `$extend` that is not an object
- A value referenced by `$splice` that is not an array
- `$splice` on an object that is not an element of an array
- Any key alongside `$ref` or `$splice`, including `$anchor`
- A node containing both `$extend` and its synonym `$extends`
- A directive whose value is malformed or of the wrong type — an `$anchor` name
  that does not match its pattern, say, or a `$ref` given an array
- A `$`-prefixed key defined by neither this specification nor the host format
  (see [Host directives](#host-directives))

## Part 3 — Background

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

| Concept         | Standard                       | Deviation                                            |
| --------------- | ------------------------------ | ---------------------------------------------------- |
| References      | RFC 3986 (URI Reference)       | None                                                 |
| Node addressing | RFC 6901 (JSON Pointer)        | Adds `$anchor` names as an alternative fragment form |
| Anchors         | JSON Schema `$anchor`          | Same keyword and fragment form; no base URI support  |
| Replacement     | JSON Reference (expired draft) | Sibling keys are an error rather than ignored        |
| Merge semantics | RFC 7396 (JSON Merge Patch)    | Unmodified; `$extend` accepts only objects           |
| Array splicing  | none                           | `$splice` has no standard counterpart                |
| Escaping        | RFC 6901 §3 (`~0`, `~1`)       | None                                                 |

JSON Schema and JSON-LD both solve document identity and reference, but their
machinery is far larger than this needs. Rather than subset either one, this
specification extends JSON Pointer with a single idea — a node may name itself —
and reuses JSON Merge Patch unchanged for everything else.

JSON Schema's `$id`, which establishes a base URI for a schema resource, has no
counterpart here. `$anchor` alone provides all the naming this format needs, and
in consequence a document's base URI is always the location it was loaded from
and cannot be moved by anything the document says.

JsonRef v0.4.0 made the same break with JSON Schema independently, and its `$id`
has exactly the role `$anchor` has here: it names a node without moving the base
URI, and its fragment forms are the ones above. This specification keeps
`$anchor` because JSON Schema, by far the more widely used of the two, gives
`$id` base-URI semantics — enough that JsonRef itself must allow a root `$id` to
be an absolute URI. Renaming `$anchor` to `$id` remains under consideration; if
it happens, the name pattern would follow JsonRef's exactly.

Fragment identifier syntax and semantics belong to the media type of the
retrieved resource, and BCP 190 (RFC 8820) accordingly forbids one specification
from defining structure within another's fragments. This specification defines
fragment semantics only for its own [media type](#media-type), following the
precedent of JSON Schema, which defines plain-name fragments for
`application/schema+json`. The pointer form needs no such licence: RFC 6901 §6
already defines it as a reusable fragment representation for any JSON-based
media type.

`$ref` itself originates in the JSON Reference draft, which specified that any
members other than `$ref` "SHALL be ignored". JSON Schema draft-07 and OpenAPI
3.0 inherited that rule; JSON Schema 2019-09 and later reversed it, applying
such members in addition. This specification does neither: sibling keys are an
error, so that a document written against either interpretation fails loudly
rather than behaving in a way its author did not intend.

#### Composable JSON cannot compose JSON Schema

Borrowing four keywords from JSON Schema invites the assumption that this format
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

Beyond those three, `$id`, `$schema`, `$vocabulary`, `$dynamicRef` and
`$dynamicAnchor` are errors here, being `$`-prefixed keys this specification
does not define — unless a host format declares them as its own.

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
- [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901) — JavaScript Object
  Notation (JSON) Pointer
- [RFC 7396](https://www.rfc-editor.org/rfc/rfc7396) — JSON Merge Patch
- [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) — The JavaScript Object
  Notation (JSON) Data Interchange Format
- [JSON Schema Core 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)
  — source of `$anchor` and `$defs`

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
