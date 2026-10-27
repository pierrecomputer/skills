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
| `CommitBuilder` | none: `files` takes a `CommitFile[]` or an iterable of `CommitFile` ([methods.md](methods.md)) |
| `CommitFileSource` | `FileContent`: a `Content` value, or a function that returns one ([Commits](#commits)) |
| `CommitResult` | `CommitWriteResponse` |
| `RefUpdate` | `RefUpdateResult` |
| `CommitInfo` | `Commit` |
| `BranchInfo` | `BranchListItem`; `Branch` has the same fields without `cursor` |
| `TagInfo` | `TagListItem`; `Tag` has the same fields without `cursor` |
| `ListFilesResult` | `Page<FileEntry, ReposFilesListResponse>` |
| `TreeEntry` | `FileEntry` |
| `ListFilesWithMetadataResult` | `Page<FileWithMetadata, ReposFilesMetadataListResponse>` |
| `CommitMetadata` | `FileLastCommit` |
| `GetBranchDiffResult`, `GetCommitDiffResult` | `ReposDiffGetResponse` |
| `FileDiff` | `FileDiff`, with `state` as a Git status letter |
| `WebhookPushEvent` | `GitPushEvent` |
| `WebhookValidationResult` | none: `verifyWebhook` returns `KnownWebhookEvent \| UnknownWebhookEvent` from `@pierre/sdk/webhooks` |
| `ApiError` | `APIError` |

A list method returns a `PagePromise<Item, Body>`, and `await` gives a `Page<Item, Body>`.
`Item` is the type of each item in `data`, and `Body` is the `Repos…ListResponse` type of the method.
So each other old `List…Result` type also becomes a `Page`, for example `Page<BranchListItem, ReposBranchesListResponse>` for `ListBranchesResult`.

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
- The old `CommitFileSource` also took a stream or an iterable of `string`, `ArrayBuffer`, or other `ArrayBufferView` chunks.
  Each chunk must now be a `Uint8Array`.
  Convert a `string` chunk with `new TextEncoder().encode(chunk)`, and an `ArrayBuffer` chunk with `new Uint8Array(chunk)`.
  In JavaScript without type checks, search for these sources, because the SDK does not check the type of a chunk.

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

## Test mocks

A test that mocks the old client or a `Repo` changes with the code.

```ts
import {
  NotFoundError,
  Page,
  RefUpdateError,
  type FileEntry,
  type Pierre,
  type ReposCommitsCreateParams,
  type ReposFilesListResponse,
} from '@pierre/sdk'

function onePage<Item, Body>(data: Item[], body: Body): Page<Item, Body> {
  const fields = { data, has_more: false, next_cursor: undefined, body, response: new Response() }
  return new Page(fields, () => Promise.reject(new Error('no next page')))
}

const entries: FileEntry[] = [{ path: 'README.md', mode: '100644', type: 'blob' }]
const listing: ReposFilesListResponse = { entries, paths: ['README.md'], ref: 'main', has_more: false }
const written: string[] = []

const pierre = {
  repos: {
    get: async () => {
      throw new NotFoundError(404, { code: 'repository_not_found' })
    },
    files: {
      list: () => onePage(entries, listing),
    },
    commits: {
      create: async (params: ReposCommitsCreateParams) => {
        for await (const file of params.files) written.push(file.path)
        throw new RefUpdateError(409, { result: { status: 'conflict' } })
      },
    },
  },
} as unknown as Pierre
```

- The client nests its namespaces, for example `pierre.repos.files.metadata.list`.
  Give the mock the same path to each method that the code calls, and cast it with `as unknown as Pierre`.
- A method returns an `APIPromise`.
  When the code only awaits the result, an `async` function can be the mock.
  Add `withResponse()` or `asResponse()` only when the code calls it.
- A list method returns a `PagePromise`.
  Return a `Page` from the mock, not a promise of a `Page`, so that `await`, `for await`, and `page.body` all work.
  The second argument of `Page` gets the next page from a cursor.
  The SDK calls it only when `has_more` is true and `next_cursor` is set.
- A mock of the old commit builder becomes a mock of `repos.commits.create`.
  Read `params.files` with `for await`, because it can be an async iterable.
  When a `content` is a function, call it to get the content.
- Make each error from the status and the body of the response, with the class that the SDK picks ([sdk.md](sdk.md)):
  `new NotFoundError(404, { code: 'not_found' })`, or `new APIError(412, { code: 'precondition_failed' })` for a status with no subclass.
  The old `new ApiError({ message, status, statusText, method, url, body })` becomes `new APIError(status, body)` or its subclass.
- A failed ref update is a `RefUpdateError` with `result` in the body, at the status that the server sends for that result: `new RefUpdateError(409, { result: { status: 'conflict' } })`, or `new RefUpdateError(412, { result: { status: 'precondition_failed' } })`.
  The old `new RefUpdateError(message, { status })` becomes this form.
- The message of an SDK error is `<status> <code>: <detail>`.
  Compare the class and `code` in place of a message.
  When the code shows the message, change the text that the test expects.

## Find the remaining uses

Run the project's type check, for example `npx tsc --noEmit`.
Each error after the import change is a use to move.
In JavaScript without type checks, search for the old method names and run the tests.
