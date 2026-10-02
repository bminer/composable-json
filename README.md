# Composable JSON

A small specification for reusing and overriding values across JSON documents,
while every document stays plain JSON.

JSON has no include, import or inheritance. Composable JSON adds them through a
few reserved `$` keys, called _directives_. A _resolver_ replaces each directive
with the value it references and produces ordinary JSON, which any consumer can
read without knowing this specification exists.

**[Read the specification →](SPEC.md)**

## Example

`service.base.json`:

```json
{
	"server": { "host": "0.0.0.0", "port": 8080, "tls": false },
	"logging": { "level": "info", "format": "json" }
}
```

`service.staging.json`:

```json
{
	"$extend": "./service.base.json",
	"server": { "tls": true },
	"logging": { "level": "debug" }
}
```

resolves to:

```json
{
	"server": { "host": "0.0.0.0", "port": 8080, "tls": true },
	"logging": { "level": "debug", "format": "json" }
}
```

## Directives

| Directive  | Purpose                                              |
| ---------- | ---------------------------------------------------- |
| `$ref`     | replaces this node with the referenced value         |
| `$extend`  | merges referenced objects under this node's own keys |
| `$splice`  | inserts referenced arrays' elements in its place     |
| `$anchor`  | names this node so that references can find it       |
| `$defs`    | holds fragments referenced within the document       |
| `$comment` | records a note, ignored when resolving               |
| `$schema`  | names a schema for editors, ignored when resolving   |

References are URIs with a JSON Pointer or anchor fragment, such as
`./base.json#/server` or `./base.json#rateLimit`. Objects merge following JSON
Merge Patch (RFC 7396), and arrays are replaced unless they are explicitly
spliced.

## Design goals

- **Still JSON.** Every document parses with any JSON parser, and the resolved
  output is plain JSON.
- **Built on existing standards.** References use RFC 3986, addressing uses RFC
  6901, and merging uses RFC 7396. The spec adds only what none of them cover.
- **Fails loudly.** Unknown `$` keys, keys next to `$ref`, duplicate anchors and
  cycles are all errors. A typo can't quietly resolve to the wrong document.
- **Safe by default.** Resolvers don't fetch over the network unless the user
  enables it, and remote documents can never read local files.

## Status

Version **1.0.0-draft**. The specification may still change incompatibly before
1.0.0. See [Versioning](SPEC.md#versioning) for the release policy.

Conformance test cases are planned for this repository.

## License

[MIT](LICENSE)
