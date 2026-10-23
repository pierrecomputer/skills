# TypeScript and JavaScript

Use this file with [methods.md](methods.md) and [sdk.md](sdk.md).
They say what changes; this file shows how to write it.

## Package

- Install `@pierre/sdk` from the `canary` tag, for example `npm install @pierre/sdk@canary`.
  Until the first release, the `latest` tag holds the placeholder `0.0.0`, so do not install the package with no tag or version.
  `npm view @pierre/sdk dist-tags` lists the tags and their versions.
- Write the exact version in `package.json`, such as `0.1.0-canary.1`, with no `^`.
  A canary can change with no notice.
- `@pierre/sdk` is an ES module, and it needs Node 22.12 or later.
  On those versions, CommonJS code can `require` it.
- Find the project's Node version in `engines.node` of `package.json`, `.nvmrc`, `.node-version`, `.tool-versions`, the CI configuration, or a `Dockerfile`.
  **Ask** when a file names an older Node: "Update Node to 22.12 or later?"
  Default: stop, and report that the move needs the update.
  When no file names a version, continue, and report the requirement.
- Import the local functions from their subpaths: `remoteUrl` from `@pierre/sdk/git`, `createToken` from `@pierre/sdk/auth`, and `verifyWebhook` from `@pierre/sdk/webhooks`.
  Import the client and the error classes from `@pierre/sdk`.

## Client

[methods.md](methods.md) shows the client in TypeScript.
`timeout` is in milliseconds.

An option that is `undefined` falls back to its `PIERRE_*` variable, then to the default.
An empty variable counts as not set.
An empty string option is an error.
Pass `value || undefined` where the old code passed a value that can be empty.

## Types

| Old (`@pierre/storage`) | New (`@pierre/sdk`) |
| --- | --- |
| `GitStorage`, `CodeStorage` | `Pierre` |
| `Repo` | `string`, the repository name, or a local `Repo` type (below) |
| `CommitSignature` | `GitIdentity` |
| the builder's files | `CommitFile[]` |
| `CommitResult` | `CommitWriteResponse` |
| `RefUpdate` | `RefUpdateResult` |
| `CommitInfo` | `Commit` |
| `GetBranchDiffResult`, `GetCommitDiffResult` | `ReposDiffGetResponse` |
| `FileDiff` | `FileDiff`, with `state` as a Git status letter |
| `WebhookPushEvent` | `GitPushEvent` |
| `WebhookValidationResult` | none: `verifyWebhook` returns `KnownWebhookEvent \| UnknownWebhookEvent` from `@pierre/sdk/webhooks` |
| `ApiError` | `APIError` |

Import the types from `@pierre/sdk` with `import type`.

## A local `Repo` type

Use this form when the user keeps the function signatures ([methods.md](methods.md), "`repo` → none").

```ts
import { NotFoundError, type Pierre } from '@pierre/sdk'

export interface Repo {
  readonly pierre: Pierre
  readonly id: string
  readonly defaultBranch: string
}

export async function findRepo(pierre: Pierre, name: string): Promise<Repo | null> {
  const found = await pierre.repos.get({ repo_name: name }).catch((err) => {
    if (err instanceof NotFoundError) return null
    throw err
  })
  return found && { pierre, id: found.repo_name, defaultBranch: found.default_branch }
}

export async function startDraft(repo: Repo, branch: string): Promise<void> {
  await repo.pierre.repos.branches.create({
    repo_name: repo.id,
    base_ref: repo.defaultBranch,
    target_branch: branch,
  })
}
```

## Calls

```ts
// @pierre/storage
const repo = storage.repo({ id: name })
await repo.createBranch({ baseRef: 'main', targetBranch: 'feat', ttl: 600 })

// @pierre/sdk
await pierre.repos.branches.create(
  { repo_name: name, base_ref: 'main', target_branch: 'feat' },
  { tokenTtl: 600 },
)
```

- The second argument holds the request options: `signal`, `timeout` (milliseconds), `maxRetries`, `headers`, `tokenTtl` (seconds), and `refPolicies`.
- `.withResponse()` gives `{ data, response, headers, request_id }`.
  Read a status, such as `201` or `200` from `repos.create`, from `response.status`.

## Errors

Import the error classes from `@pierre/sdk`, and `WebhookVerificationError` also from `@pierre/sdk/webhooks`.

| Old field | New field |
| --- | --- |
| `ApiError.status` | `status` |
| `ApiError.statusText` | `title` |
| `ApiError.body` | `body`; read `code` and `detail` |
| `ApiError.method`, `ApiError.url` | none |
| `RefUpdateError.reason` | `result.status`; `code` can be absent |
| `RefUpdateError.refUpdate` | `result` |

`result` has the type `RefUpdateResult | NotesWriteResult`.
Read `result.status` with no cast:

```ts
import { NotFoundError, RefUpdateError } from '@pierre/sdk'

export function isStaleHead(err: unknown): boolean {
  return err instanceof RefUpdateError && err.result.status === 'conflict'
}

export function isMissingObject(err: unknown): boolean {
  return err instanceof NotFoundError && err.code !== 'repository_not_found'
}
```

## Pagination

```ts
for await (const repo of pierre.repos.list({ limit: 100 })) {
  // each item of every page
}

const page = await pierre.repos.list() // one page
page.data
page.next_cursor
```

`page.hasNextPage()`, `page.getNextPage()`, and `page.iterPages()` move one page at a time.

## Commits

```ts
await pierre.repos.commits.create({
  repo_name: name,
  target_branch: 'main',
  commit_message: 'Update docs',
  author: { name: 'Bot', email: 'bot@example.com' },
  expected_target_sha: sha,
  files: [
    { path: 'README.md', content: '# Web' },
    { path: 'logo.png', content: () => readFile('logo.png') },
    { path: 'old.txt', operation: 'delete' },
  ],
})
```

- `content` can be a string, a `Uint8Array`, an `ArrayBuffer`, a `Blob`, a `ReadableStream<Uint8Array>`, an iterable or async iterable of `Uint8Array`, or a function that returns one of them.
- For text in another encoding than UTF-8, pass `Buffer.from(text, encoding)`.

## Local functions

```ts
import { remoteUrl } from '@pierre/sdk/git'
import { verifyWebhook } from '@pierre/sdk/webhooks'

const url = await remoteUrl(pierre, { repo_name: name, ttl: 3600 })
const event = await verifyWebhook({ payload: rawBody, headers: req.headers, secret })
```

- `payload` is the raw body: a string, a `Uint8Array`, or an `ArrayBuffer`.
  Do not pass parsed JSON.
- `headers` is a Fetch `Headers` object or a record such as Node's request headers.
- `createToken(pierre, { scopes, repo, ttl })` returns a promise.

## Files

`repos.files.get` and `repos.archive.get` return a Fetch `Response`.
Read it with `await res.text()`, `await res.arrayBuffer()`, or `res.body`.

## Find the remaining uses

Run the project's type check, for example `npx tsc --noEmit`.
Each error after the import change is a use to move.
In JavaScript without type checks, search for the old method names and run the tests.
