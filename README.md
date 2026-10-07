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
spliced. Any other `$` key is an error unless a host format defines it; `$id` is
reserved for a future version.

## Design goals

- **Still JSON.** Every document parses with any JSON parser, and the resolved
  output is plain JSON.
- **Built on existing standards.** References use RFC 3986, addressing uses RFC
  6901, and merging uses RFC 7396. The spec adds only what none of them cover.
- **Fails loudly.** Unknown `$` keys, keys next to `$ref`, duplicate anchors and
  cycles are all errors. A typo can't quietly resolve to the wrong document.
- **Safe by default.** Resolvers don't fetch over the network unless the user
  enables it, and remote documents can never read local files.

## Implementations

| Language | Repository                                                                | Spec version |
| -------- | ------------------------------------------------------------------------- | ------------ |
| Go       | [bminer/composable-json-go](https://github.com/bminer/composable-json-go) | 1.0.0-draft  |

An implementation that passes the [conformance tests](#conformance-tests) is
welcome here.

## Conformance tests

[`tests/`](tests) holds a language-neutral conformance suite. Each case gives a
set of documents and either the resolved output or the error resolving them must
report. To test an implementation, write a small harness that serves the
documents from memory, resolves the root, and compares the result. See the
[test suite README](tests/README.md) for the format and error codes.

## Repository layout

| Path                                                 | Contents                              |
| ---------------------------------------------------- | ------------------------------------- |
| [`SPEC.md`](SPEC.md)                                 | the specification                     |
| [`tests/cases/`](tests/cases)                        | conformance tests, one file per topic |
| [`tests/suite.schema.json`](tests/suite.schema.json) | JSON Schema for the test files        |

## Status

Version **1.0.0-draft**. The specification may still change incompatibly before
1.0.0. See [Versioning](SPEC.md#versioning) for the release policy.

## License

[MIT](LICENSE)
