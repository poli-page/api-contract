# Poli Page API contract

`contract.json` describes the Poli Page `/v1` API as its test suite sees it: every route and its
successful answer, every error the tests observed, the rules an SDK follows (retries, error
codes and messages) and the test cases each SDK and integration library runs against a local
mock server.

This repository is published from the `develop` branch of the Poli Page monorepo, by CI, each
time an API change changes the contract. Do not edit it here: changes are overwritten by the
next publish. Each publish is a commit `contract from develop <sha>`.

## Versions

Tags follow the Poli Page API version, the version prod runs:

- `vX.Y.0`: the contract of prod release `vX.Y.0`, for every minor release (an unchanged contract
  is tagged again). A release that changes the contract is always a minor; patch releases keep
  the contract and get no tag.
- `vX.Y.0-dev.N`: an unreleased contract from `develop`, which the next minor `vX.Y` will ship.

An SDK or integration `X.Y.z` passes contract `X.Y` and is released once prod runs API `X.Y`.

## Usage

The repository is a GitHub Action that copies `contract.json` to `$RUNNER_TEMP` and outputs its
path. Pin it by full commit SHA:

```yaml
- uses: poli-page/api-contract@<sha> # vX.Y.Z
  id: contract
- run: <your test command>
  env:
      POLI_CONTRACT: ${{ steps.contract.outputs.path }}
```

Your tests read the file at `$POLI_CONTRACT`.

To get a pull request for each new release, enable Dependabot for GitHub Actions in
`.github/dependabot.yml` (while you pin a `vX.Y.Z` tag, it skips the `-dev.N` prereleases):

```yaml
version: 2
updates:
    - package-ecosystem: github-actions
      directory: /
      schedule:
          interval: daily
      # Dependabot waits 3 days after a release by default; exempt the contract.
      cooldown:
          default-days: 3
          exclude:
              - 'poli-page/api-contract'
```

## `contract.json`

One JSON object, keys sorted, with five fields.

### `formatVersion`

`1`, the version of this layout.

### `routes`

Every `/v1` route, sorted by path then method: `method`, `path` (Hono syntax, `:id` for a
parameter) and `responses`, the successful answers (`status` and `body`; `body.kind` is `json`
with a JSON Schema (draft 2020-12) in `body.schema`, `raw` with its `contentType`, or `empty`).

### `errors`

Every error the API test suite saw on `/v1`: `method`, `route` (the route path, or `*` for a
path or method no route has), `status` and `code`. The error body is
`{ error, detail?, issues?, requestId }`.

### `policy`

The rules an SDK follows: `sdkMethods` (SDK method to route), `sdkErrors` (SDK-side codes and
their messages), `retryRules`, `nonRetryableCodes` (the API codes of `R-NONRETRY`; an SDK-side
code that is never retryable, such as `DOWNLOAD_FAILED`, has its own rule), `retryAfterClampMs`,
`normalizations` (what an SDK test may normalize before comparing), `errorCaseMethod` (the SDK
method a generated error case calls, per route), `cancellationSdks`, and for integration libraries
`implErrorBody` and `implStatusForSdkCode`.

### `cases`

The test cases, generated ones (one per recorded error of an SDK route, id `error.…`) first, then
the fixed ones.

## Case format

Each case has a unique `id`, a `title` and a `kind`.

- `sdk`: build a client with `options` (`maxRetries`, `retryDelayMs`, `timeoutMs`) and `apiKey`
  (absent means a fixed test key) pointed at a local server; the server answers `responses` in
  order (a `status` with `headers` and one of `json`, `text` or `base64`, or a `fault`: `drop` or
  `hang`) and records the requests. Call `call.method` with `call.args`, then compare `expect.requests` (method,
  path, auth, idempotency key, body; timing as lower bounds measured on the server, in integer milliseconds; after a hung
  attempt the bound counts only half of `timeoutMs`, since the client's timer starts before its
  request reaches the server), `expect.events` (`request` and
  `retry` hooks) and `expect.result` (the value, or the normalized error with its predicates).
  A case with `requires: "cancellation"` runs only in the SDKs of `policy.cancellationSdks`.
  A case with `stopAfterRetryEvents: N` is stopped at its Nth `retry` event (cancel or abandon the
  call): compare the events and requests up to that point, not the result.
- `impl`: for integration libraries. Send `request` to the example app, its SDK pointed at the
  local server answering `responses`, and compare the app's answer with `expect` (`status`,
  `location`, `json`, `contentType`, and the SDK's `requests` when present).

### Placeholders

Strings in a case may contain placeholders, replaced before the case runs:

- `{{mock}}`: the local server's origin, e.g. `http://127.0.0.1:41234`.
- `{{httpDate+N}}`: an HTTP-date (`Wed, 21 Oct 2026 07:28:00 GMT`) N seconds after the
  replacement.

Any other placeholder is an error. A case without `apiKey` uses any non-empty test key; requests
with `auth: "bearer"` carry `Authorization: Bearer <that key>`.

`policy.sdkErrors` messages use `{timeoutMs}`, `{status}` and `{reason}` (the OS or runtime error's
own text); an entry with `networkMessage` uses it instead when the failure has no HTTP status, and
one with `timeoutMessage` when the client's `timeoutMs` elapsed. An entry's `status` says where the
error's status comes from: `none`, `storage` (the presigned-URL download) or `api` (the API's
answer). An entry with `dynamicOnly` is raised only by dynamically typed SDKs, one with
`fileHelperOnly` only by render-to-file helpers; no case reaches the latter.

## License

MIT, see [LICENSE](LICENSE).
