# Old SDK methods

Each section moves one old client, method, or function to `@pierre/sdk`.
The examples use TypeScript, with `store` for the old client and `pierre` for the new one.
The language file shows how to write these calls in Python and Go.

- An option or a result field that a section does not name keeps its name in snake_case: `targetBranch` becomes `target_branch`.
- Each old `ttl` option becomes the request option for the token TTL ([sdk.md](sdk.md)).
- There is no repository handle: each repository method takes `repo_name`.

## Client

```ts
// @pierre/storage
import { GitStorage } from '@pierre/storage'
const store = new GitStorage({ name: 'acme', key: privateKeyPem })

// @pierre/sdk
import { Pierre } from '@pierre/sdk'
const pierre = new Pierre({ org: 'acme', privateKey: privateKeyPem })
const pierreFromEnv = new Pierre() // reads the PIERRE_* variables
```

| Old | New |
| --- | --- |
| `GitStorage`, `CodeStorage`, `createClient(options)` | `new Pierre(options)` |
| `name` | `org` |
| `key` | `privateKey` |
| `token`, `fetch` | `token`, `fetch` |
| `apiBaseUrl`, `storageBaseUrl` | `baseUrl`, `gitHost` |
| `apiVersion`, `defaultTTL` | none: remove them |
| `getConfig()`, `GitStorage.getDefaultAPIBaseUrl`, `getDefaultStorageBaseUrl` | `pierre.options`, with `org`, `baseUrl`, and `gitHost`, and no credential |

- The new client reads `PIERRE_ORG`, `PIERRE_PRIVATE_KEY` or `PIERRE_PRIVATE_KEY_FILE`, `PIERRE_TOKEN`, `PIERRE_API_BASE_URL`, and `PIERRE_GIT_HOST`.
  When the code passes these values explicitly, keep that form.
- A credential option replaces every credential variable.
  Give at most one of `privateKey`, `token`, and `getToken`.
- With `token` and no `org`, the client reads the organization from the token's `iss` claim.
- The new SDK reads only a PKCS #8 key (`-----BEGIN PRIVATE KEY-----`): EC P-256, P-384, P-521, or RSA.
  The old Python and Go SDKs also read other forms, such as `-----BEGIN EC PRIVATE KEY-----`.
  For Python and Go, check the form after the move without a question:
  1. Run the tests or a call that signs a request with the project's key.
     If it passes, the key is PKCS #8.
     A key error from the client means that it is not.
  2. If nothing can sign a request, read only the first line of the key file, for example `head -n 1 "$PIERRE_PRIVATE_KEY_FILE"`.
     It is `-----BEGIN PRIVATE KEY-----` for PKCS #8.
     Do not read the other lines.
  3. If the key is not PKCS #8, **Report** the command that converts it: `openssl pkcs8 -topk8 -nocrypt -in key.pem -out key-pkcs8.pem`.

  **Ask** only when no call can sign a request and you cannot read the key file, for example when the key comes from a secret store: "Is your key in PKCS #8 form? If not, convert it with the command above."
  Default: report the check.

## Repositories

### `createRepo` → `repos.create`

```ts
// @pierre/storage
const repo = await store.createRepo({ id: name, defaultBranch: 'main' })
await repo.createBranch({ baseRef: 'main', targetBranch: 'feat' })

// @pierre/sdk
const created = await pierre.repos.create({ repo_name: name, default_branch: 'main' })
await pierre.repos.branches.create({ repo_name: created.repo_name, base_ref: 'main', target_branch: 'feat' })
```

- `id` becomes `repo_name`.
  Without it, both SDKs make a random UUID; read it from `created.repo_name`.
- A copy, `baseRepo: { id, ref, sha }`, becomes `base_repo: { name: id, ref }`.
  Send `sha` as `ref`.
  The SDK signs the source token.
- A sync, `baseRepo` with `provider`, becomes `upstream` and `default_branch` ([names.md](names.md)).
- The result is `{ repo_name, message }`, not a `Repo`.
  The old `repo.defaultBranch` was the value that the old SDK sent: the sync's `defaultBranch`, else the `defaultBranch` option, else `main`.
  Use the same value; it needs no request.
  The old `repo.createdAt` was the client's clock; use `repos.get` for the stored `created_at`.
- The old call returned a `Repo` for a new name and for an existing name alike.
  `repos.create` returns `201` for a new repository and `200` for an existing name, and `message` says which.
  **Ask** only when the code must know which: "Treat an existing name as success, as before, or read the status?"
  Default: treat it as success.
- The old call raised "Repository already exists" for every `409`.
  A `409` now raises `ConflictError` with a `code`, for example `upstream_already_configured` or `repository_thawing`.
  **Report:** a check of the old message becomes a check of `ConflictError`.

### `findOne` → `repos.get`

```ts
// @pierre/storage
const repo = await store.findOne({ id: name }) // null when the repository does not exist

// @pierre/sdk
const found = await pierre.repos.get({ repo_name: name }).catch((err) => {
  if (err instanceof NotFoundError) return null
  throw err
})
```

- `id` becomes `repo_name`.
- The result is a repository: `repo_name`, `default_branch`, `created_at`, and `upstream`.
  The old `Repo` had `id` (the name), `defaultBranch`, and `createdAt`.
- **Report:** a `null` check becomes a catch of `NotFoundError` that keeps the `null` branch.
- Code that called methods on the returned `Repo` passes `repo_name` to each method.

### `repo` → none

`store.repo({ id })` made no request.
Replace each `repo.<method>(options)` with the new method and `repo_name: id`.

Inside one function, use the repository name in place of the `Repo`.
When the project passes a `Repo` across its own exported functions, or stores it in a field, the change reaches each caller.

**Ask** in that case: "Keep your function signatures with a small local `Repo` type, or change each signature to take the client and the repository name?"
Recommend the local type when other code imports these functions.
Default: the local type.

- The local type holds the client, the name, and each field of the old `Repo` that the code reads, under the same names, such as `id` and `defaultBranch`.
  The language file shows it.
- The functions that returned a `Repo` build the local type from the result of `repos.get` or `repos.create`.
  Each other function keeps its signature.
- **Report:** with the other option, each function and field that held a `Repo` holds the repository name, and each caller changes.

### `listRepos` → `repos.list`

- The options keep their names: `cursor`, `limit`, and `q`.
- The result is a page of repositories ([sdk.md](sdk.md)).
  In each item, `url` becomes `repo_name`, and `baseRepo` becomes `upstream`.
  `repoId` holds an internal ID ([names.md](names.md)).

### `deleteRepo` → `repos.delete`

- `id` becomes `repo_name`.
- The result's `repoId` holds an internal ID; use `repo_name` ([names.md](names.md)).

### `updateRepo` → `repos.update`

The method is in `@pierre/storage` 1.18 betas only.
Read the operation's docs page for its fields.

## Git credentials

| Old | New | Changes |
| --- | --- | --- |
| `createGitCredential` | `repos.gitCredentials.create` | `repoId` becomes `repo_name` |
| `updateGitCredential` | `repos.gitCredentials.update` | `id` becomes `git_credential_id`; add `repo_name` |
| `deleteGitCredential` | `repos.gitCredentials.delete` | `id` becomes `git_credential_id`; add `repo_name` |

The old update and delete took only the credential ID.
Get the repository name where the code created the credential.
**Ask** when the code keeps only the credential ID: "Where can the code get the repository of each credential?"
Default: keep those calls on the old SDK, and report them.

## Branches and tags

| Old | New | Changes |
| --- | --- | --- |
| `listBranches` | `repos.branches.list` | the result is a page ([sdk.md](sdk.md)) |
| `createBranch` | `repos.branches.create` | `baseRef` and the deprecated `baseBranch` become `base_ref` |
| `deleteBranch` | `repos.branches.delete` | `name` becomes `target_branch`; the result's `name` becomes `target_branch` |
| `promote_ephemeral_branch` (Python) | `repos.branches.create` | pass `base_ref=base_branch`, `target_branch` (or `base_branch`), `base_is_ephemeral=True`, `target_is_ephemeral=False` |
| `listTags` | `repos.tags.list` | the result is a page |
| `createTag` | `repos.tags.create` | `name` stays `name`; `target` becomes `ref` |
| `deleteTag` | `repos.tags.delete` | `name` becomes `tag_name` |

## Commits

### `createCommit` and its builder → `repos.commits.create`

```ts
// @pierre/storage
const result = await store
  .repo({ id: name })
  .createCommit({ targetBranch: 'main', commitMessage: 'Update docs', author, expectedHeadSha: sha })
  .addFileFromString('README.md', '# Web')
  .addFile('logo.png', logoBytes)
  .deletePath('old.txt')
  .send()

// @pierre/sdk
const written = await pierre.repos.commits.create({
  repo_name: name,
  target_branch: 'main',
  commit_message: 'Update docs',
  author,
  expected_target_sha: sha,
  files: [
    { path: 'README.md', content: '# Web' },
    { path: 'logo.png', content: logoBytes },
    { path: 'old.txt', operation: 'delete' },
  ],
})
```

| Old | New |
| --- | --- |
| `targetRef` | `target_branch` |
| `expectedHeadSha` | `expected_target_sha` |
| `ephemeral`, `ephemeralBase` | `target_is_ephemeral`, `base_is_ephemeral` |
| `signal` | the request option `signal` |
| `addFile(path, source, { mode })` | `{ path, content: source, mode }` |
| `addFileFromString(path, text)` | `{ path, content: text }` |
| `addFileFromString(path, text, { encoding })`, not UTF-8 | `{ path, content: <text encoded to bytes> }` |
| `addFileFromBytes(path, bytes)` (Go) | `{ path, content: bytes }` |
| `deletePath(path)` | `{ path, operation: 'delete' }` |
| `send()` | the call itself |

- `files` can be an array or an iterable, also an async one.
  The SDK reads all entries before it sends any content.
- `content` can be a function.
  The SDK calls it when it sends that file, so only one file is open at a time.
- A builder in a loop becomes an array that the loop fills, or a generator.

The result is `{ commit, result }`:

| Old `CommitResult` field | New field |
| --- | --- |
| `commitSha`, `treeSha`, `targetBranch` | `commit.commit_sha`, `commit.tree_sha`, `commit.target_branch` |
| `packBytes`, `blobCount` | `commit.pack_bytes`, `commit.blob_count` |
| `refUpdate.branch`, `refUpdate.oldSha`, `refUpdate.newSha` | `result.target_branch`, `result.old_sha`, `result.new_sha` |

### Other commit methods

| Old | New | Changes |
| --- | --- | --- |
| `createCommitFromDiff` | `repos.commits.createFromDiff` | the options of `createCommit`, and `diff`; the same result |
| `restoreCommit` | `repos.commits.restore` | `targetCommitSha` becomes `base_ref`; `expectedHeadSha` becomes `expected_target_sha`; the same result as `createCommit` |
| `listCommits` | `repos.commits.list` | `branch` becomes `ref`; the result is a page; parse `date` where the code used a `Date` |
| `getCommit` | `repos.commits.get` | `sha` becomes `ref`; the result is `{ commit }` |

## Diffs and blame

### `getBranchDiff` and `getCommitDiff` → `repos.diff.get`

```ts
// @pierre/storage
const diff = await repo.getBranchDiff({ branch: 'feat' })

// @pierre/sdk
const { default_branch } = await pierre.repos.get({ repo_name: name })
const branchDiff = await pierre.repos.diff.get({ repo_name: name, ref: 'feat', base_ref: default_branch })
```

| Old | New |
| --- | --- |
| `getBranchDiff({ branch, base, ephemeral, ephemeralBase, paths })` | `{ ref, base_ref, ref_is_ephemeral, base_is_ephemeral, path }` |
| `getCommitDiff({ sha, baseSha, gitApplyCompatible, paths })` | `{ ref, base_ref, git_apply_compatible, path }` |

- Without `base`, the old branch diff compared the branch with the default branch.
  Without `base_ref`, `repos.diff.get` returns the diff of one commit.
  So when the old call has no `base`, pass the default branch as `base_ref`.
  Use the default branch that the code already knows, or read it from `repos.get`.
- The result has `sha` and `base_sha`, not `branch` and `base`.
  `files`, `filtered_files`, `stats`, and `merge_base_sha` keep their names.
- A file's `state` is now the Git status letter ([sdk.md](sdk.md)).

### `getBlame` → `repos.blame.get`

`ranges` becomes `range`.
In each line, `rawAuthorTime` and `rawCommitterTime` become `author_time` and `committer_time`.

## Files and archives

| Old | New | Changes |
| --- | --- | --- |
| `getFileStream` | `repos.files.get` | `headers.ifNoneMatch` and the other headers become params: `range`, `if_match`, `if_none_match`, `if_modified_since`, `if_unmodified_since`, `if_range`; remove `ephemeralBase`, which the server never read |
| `headFile` | `repos.files.metadata.get` | the same params; the result changes (table below) |
| `getArchiveStream` | `repos.archive.get` | `archivePrefix` becomes `archive: { prefix }` |
| `listFiles` | `repos.files.list` | the result is a page of `entries`; `paths` and `ref` are in `page.body` |
| `listFilesWithMetadata` | `repos.files.metadata.list` | the result is a page of `files`; `commits` and `ref` are in `page.body` |

- `repos.files.get` and `repos.archive.get` return the stream type of the language.
- With no `limit` and no `cursor`, `repos.files.list` and `repos.files.metadata.list` return the full listing in one page, and `has_more` is false.
  The old calls did the same, so a call with no `limit` needs no loop.
  With a `limit` or a `cursor`, each call returns one page: `repos.files.list` returns up to 1000 entries by default and 5000 at most, and `repos.files.metadata.list` returns up to 200 files by default and 1000 at most.
  The `limit` description names only this page default.
- A `304` is a result.
  A `412` or a `416` raises `APIError`; the old SDKs returned the response.
  **Report:** a check of these statuses becomes a catch of `APIError`.

| Old `headFile` field | New |
| --- | --- |
| `status` | `status` |
| `blobSha`, `lastCommitSha` | `headers.x_blob_sha`, `headers.x_last_commit_sha` |
| `size` | `headers.content_length`, as a number |
| `etag`, `acceptRanges`, `contentRange`, `contentType` | `headers.etag`, `accept_ranges`, `content_range`, `content_type` |
| `lastModified`, `rawLastModified` | `headers.last_modified`; parse it where the code needs a date |

## Search

`grep` becomes `repos.grep`.
`rev` becomes `ref`.
The nested keys take snake_case: `query: { pattern, case_sensitive }`.
Old Python passed `pattern` and `case_sensitive` at the top level; put them in `query`.

## Notes

| Old | New | Changes |
| --- | --- | --- |
| `getNote` | `repos.notes.get` | `sha` becomes `object_ref`; `ref` becomes `notes_ref` |
| `createNote`, `appendNote` | `repos.notes.create`, `repos.notes.append` | `sha` becomes `object_ref`; `ref` becomes `notes_ref`; `expectedRefSha` becomes `expected_notes_ref_sha`; the result's `targetRef` becomes `notes_ref` |
| `deleteNote` | `repos.notes.delete` | as `createNote`; remove `author` |
| `listNotesRefs` | `repos.notes.refs.list` | the result is a page of `refs`; `prefix` is in `page.body` |

## Merge and upstream

| Old | New | Changes |
| --- | --- | --- |
| `previewMerge` | `repos.previewMerge` | none |
| `merge` | `repos.merge` | `sourceBranch` becomes `source_ref`; the result's `source.branch` becomes `source.ref` |
| `pullUpstream` | `repos.upstream.pull` | remove `ref`; the result is `{ message }` |

**Report:** each `ref` that you remove from `pullUpstream` (`pull_upstream` in Python, `PullUpstream` in Go).
The option never had an effect: `repos.upstream.pull` syncs every branch and tag of the upstream.

## Remote URLs

```ts
// @pierre/storage
const url = await store.repo({ id: name }).getRemoteURL({ permissions: ['git:read'], ttl: 3600 })

// @pierre/sdk
const remote = await remoteUrl(pierre, { repo_name: name, scopes: ['git:read'], ttl: 3600 })
```

| Old | New |
| --- | --- |
| `getRemoteURL` | `remoteUrl` from `@pierre/sdk/git`, with `kind` omitted |
| `getEphemeralRemoteURL` | `kind: 'ephemeral'` |
| `getImportRemoteURL` | `kind: 'import'` |
| `permissions` | `scopes`, default `git:read` and `git:write` |
| `refPolicies` | `ref_policies`, default: the client's ref policies |
| `ops` | a last rule `{ pattern: '*', ops }` in `ref_policies`; if a last `*` rule exists, add the ops to it |
| `ttl` | `ttl`, in seconds |

The old URL token lived 365 days by default, or `defaultTTL`.
The new token lives 15 minutes by default and at most 1 hour.
The SDK rejects a longer `ttl`.

**Ask** when the old call gave no `ttl`, or a `ttl` over 3600: "How long must this URL work?"
Options:

1. 15 minutes, the new default: the code runs Git right after it makes the URL.
2. Up to 1 hour: `ttl: 3600`.
3. Longer, or a URL that stays in a Git config: use the credential helper (`pierre auth setup-git`), or make a new URL before each Git command.

Default: `ttl: 3600`.
When the code keeps the URL for longer, for example in a Git config, no new form is equal.
Then keep that call on the old SDK, and report it.

## Tokens

`generate_jwt` (Python) and `encodeRefsClaim` become `createToken` from `@pierre/sdk/auth`: `createToken(credentials, { scopes, repo, ttl, ref_policies })`.
It needs a private key.
A token from `createToken` or `remoteUrl` lives 15 minutes by default and at most 1 hour.
The token that the SDK signs for each request has its own default ([sdk.md](sdk.md)).
Python's old `generate_jwt` defaulted to 1 year.

**Ask** when the old code made a token that lives longer than 1 hour: "Use 1 hour or less, use the credential helper for Git, or sign the token with your own JWT library?"
Default: `ttl: 3600`.

## Webhooks

```ts
// @pierre/storage
const check = await validateWebhook(rawBody, headers, secret)
if (!check.valid) return reject()

// @pierre/sdk
let event: KnownWebhookEvent | UnknownWebhookEvent
try {
  event = await verifyWebhook({ payload: rawBody, headers, secret })
} catch {
  return reject()
}
```

- `verifyWebhook` from `@pierre/sdk/webhooks` returns the event, or raises `WebhookVerificationError` with `reason`: `missing_signature`, `malformed_signature`, `signature_mismatch`, `timestamp_out_of_range`, or `invalid_payload`.
  It raises a plain `PierreError` for a bad call, such as an empty secret.
- The old function returned `valid: false` for every failure, also for an empty secret.
  So the example rejects on every error, and the response stays the same.
  **Report:** a check of `valid` becomes a catch.
- The new function does not need the `X-Pierre-Event` header.
  It reads the event type from the payload.
- The event is a union on `type`: `push`, `repo.sync.started`, `repo.sync.succeeded`, `repo.sync.failed`, or `{ type: 'unknown', event_type, body }`.
  Its fields use wire names ([names.md](names.md) for `push`).
- `maxAgeSeconds` becomes `tolerance_seconds`, default 300.
  The old `0` turned off the timestamp check; the new `0` allows no age.
  **Ask** when the code sets `maxAgeSeconds: 0`: "Keep a timestamp check (recommended, 300 seconds), or set a large tolerance?"
  Default: 300.
- `validateWebhookSignature` and `parseSignatureHeader` have no equal; `verifyWebhook` does both.
- Python's `parse_push_event` has no equal; read the event that `verify_webhook` returns.
