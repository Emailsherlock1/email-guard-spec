# email-guard-spec

The language-agnostic contract behind Email-Guard: a reusable email check
that developers drop into signup forms, checkouts, and other forms where a
fake address costs real money.

This repo contains no runnable code. It holds the versioned specification
plus machine-readable conformance vectors. Core libraries in each language
implement the spec and prove it by running the vectors in their test suite.
That is what keeps `deleted+user274@deleted.invalid` blocked bit-identically
in PHP, JS, and whatever comes next.

## How the pieces fit

```
email-guard-spec (this repo)            the stable middle
  spec + conformance vectors
        |
   PHP core-lib        JS core-lib      one thin lib per language
     |       |              |
  Symfony  WordPress    Shopify         framework integrations
  bundle   plugin       extension
        |
   calls when a key is set:  EmailSherlock Verify API
```

Every integration works without an API key: syntax, reserved-TLD, and a
bundled disposable-domain snapshot run locally at zero cost. An API key adds
the data unlock: live MX, SMTP probe, a fresh disposable list, role and
catch-all detection. See the [Verify API docs](https://emailsherlock.com/api/docs).

## Repo layout

| Path | What it is |
|------|------------|
| [SPEC.md](SPEC.md) | The normative specification |
| [vectors/](vectors/) | Conformance vectors, grouped by category |
| [vectors/schema.json](vectors/schema.json) | JSON Schema for the vector files |
| [vectors/fixtures/](vectors/fixtures/) | Test fixtures (disposable snapshot used by the vectors) |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Running the vectors

A conforming implementation loads every `vectors/*.json` file, builds a guard
from `input.config`, mocks the API transport per `input.api`, checks
`input.email`, and asserts the result equals `expected`. Details in
[SPEC.md, section 8](SPEC.md#8-conformance).

## Versioning

The spec follows [SemVer](https://semver.org). Breaking changes to the
decision model, the vector format, or the data formats bump the major
version. New vectors and clarifications bump the minor version.

Current version: **1.0.0**. The PHP core library
([email-guard-core-php](https://github.com/Emailsherlock1/email-guard-core-php))
is the reference implementation and runs green against the full vector set.

## Related

- [EmailSherlock Verify API](https://emailsherlock.com/api/docs): the remote
  side of this contract (`POST /v1/verify/single`)
- Core libraries and integrations live in their own repos under the
  [Emailsherlock1](https://github.com/Emailsherlock1) org

## License

MIT, see [LICENSE](LICENSE).
