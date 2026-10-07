# Conformance tests

These tests describe the behaviour the [specification](../SPEC.md) requires.
They are data, not code: each file in [`cases/`](cases) lists documents, the one
to resolve, and either its resolved output or the error resolving it must
report. Any implementation, in any language, can run them through a small
harness.

## File format

Each file follows [`suite.schema.json`](suite.schema.json):

```json
{
	"$schema": "../suite.schema.json",
	"description": "$extend merges referenced objects",
	"spec": "../../SPEC.md#extend",
	"tests": [
		{
			"description": "deletes a key the node sets to null",
			"documents": {
				"main.json": { "$extend": "./base.json", "b": null },
				"base.json": { "a": 1, "b": 2 }
			},
			"expected": { "a": 1 }
		}
	]
}
```

A test has these fields:

| Field          | Meaning                                                                   |
| -------------- | ------------------------------------------------------------------------- |
| `description`  | what the test checks                                                      |
| `documents`    | documents by URI reference; each value is the document's JSON content     |
| `rawDocuments` | documents given as raw text, for content that is not valid JSON           |
| `root`         | the document to resolve; defaults to `main.json`                          |
| `options`      | resolver settings; see below                                              |
| `expected`     | the resolved output, compared as JSON values, so key order does not count |
| `error`        | instead of `expected`, the error resolving must report; see below         |

Document keys are URI references resolved against `file:///suite/`, so
`main.json` is `file:///suite/main.json` and `sub/a.json` is
`file:///suite/sub/a.json`. A key may also be an absolute URI, such as
`https://example.com/a.json`. A harness should serve documents from memory,
keyed by absolute URI, and never touch the network or the filesystem. A document
missing from the test does not exist.

The test files are not Composable JSON documents themselves. Their `$`-prefixed
keys belong to the documents inside them.

### Options

| Option           | Meaning                                                       |
| ---------------- | ------------------------------------------------------------- |
| `remote`         | enable the `http` and `https` schemes; off by default         |
| `hostDirectives` | directives the host format declares, such as `["$env"]`       |
| `noBaseURI`      | supply the root document as text, with no base URI to resolve |

Only the `file` scheme and, when `remote` is set, `http` and `https` are
supported. `file` is local, `https` is remote with protection in transit, and
`http` is remote without it.

### Errors

Each code corresponds to an entry in the specification's
[Errors](../SPEC.md#errors) list. A harness may check only that resolution
fails, but an implementation that classifies its errors should match these
codes.

| Code                      | Error                                                                  |
| ------------------------- | ---------------------------------------------------------------------- |
| `invalid-json`            | a document is not valid JSON                                           |
| `duplicate-key`           | an object has duplicate keys                                           |
| `unresolvable-reference`  | a reference cannot be resolved, including an unsupported or off scheme |
| `missing-base-uri`        | a relative reference in a document with no base URI                    |
| `remote-to-local`         | a remote document references a local resource                          |
| `insecure-reference`      | an `https` document references an `http` resource                      |
| `cycle`                   | a node is needed while it is still being resolved                      |
| `duplicate-anchor`        | two nodes in the result carry the same `$anchor`                       |
| `extend-type`             | `$extend` references a value that is neither an object nor `null`      |
| `splice-type`             | `$splice` references a value that is neither an array nor `null`       |
| `splice-position`         | `$splice` on an object that is not an array element                    |
| `sibling-keys`            | a key other than `$comment` alongside `$ref` or `$splice`              |
| `extend-synonym-conflict` | a node with both `$extend` and `$extends`                              |
| `empty-references`        | `$extend` or `$splice` given an empty array                            |
| `malformed-directive`     | a directive's value is malformed or of the wrong type                  |
| `unknown-directive`       | a `$`-prefixed key defined by neither the specification nor the host   |

Each error test is written so that only one error applies.
