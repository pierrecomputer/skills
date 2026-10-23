# Deprecated wire names

Use this file for HTTP requests and for SDK params.
`@pierre/sdk` uses the wire names: TypeScript and Python as they are, and Go in PascalCase with initialisms in capitals, for example `ExpectedTargetSHA`.
The TypeScript and Python types still accept some deprecated names.
Use the current name in each case.
The Go params do not have the deprecated names.

An old SDK option is usually the camelCase (TypeScript) or PascalCase (Go) form of its wire name.
[methods.md](methods.md) lists the exceptions, such as `id` and `paths`.
Find the wire name, then use this table.

## Request names

| Operation | Deprecated | Current |
| --- | --- | --- |
| `repos.create` | query `default_branch` | body `default_branch` |
| `repos.create` | `base_repo.sha` | `base_repo.ref` |
| `repos.create` | `base_repo` with an external `provider` (a sync) | `upstream`, see below |
| `repos.create` | `base_repo` with `provider: "code"` and `operation: "fork"` (a copy) | `base_repo: { name, ref, auth: { token } }`, see below |
| `repos.branches.create` | `base_branch` | `base_ref` |
| `repos.branches.delete` | body `name` or `target_branch` | query `target_branch` |
| `repos.branches.delete` | body `ephemeral` | query `ephemeral` |
| `repos.commits.create`, `repos.commits.createFromDiff` | `ephemeral` | `target_is_ephemeral` |
| `repos.commits.create`, `repos.commits.createFromDiff` | `ephemeral_base` | `base_is_ephemeral` |
| `repos.commits.create`, `repos.commits.createFromDiff`, `repos.commits.restore` | `expected_head_sha` | `expected_target_sha` |
| `repos.commits.restore` | `target_commit_sha` | `base_ref` |
| `repos.commits.get` | query `sha` | `ref` |
| `repos.commits.list` | query `branch` | `ref` |
| `repos.diff.get` | `sha`, `baseSha`, `gitApplyCompatible` | `ref`, `base_ref`, `git_apply_compatible` |
| `repos.grep` | `rev` | `ref` |
| `repos.merge` | `source_branch` | `source_ref` |
| `repos.notes.get` | query `sha`, `ref` | `object_ref`, `notes_ref` |
| `repos.notes.create`, `repos.notes.append` | `sha`, `ref`, `expected_ref_sha` | `object_ref`, `notes_ref`, `expected_notes_ref_sha` |
| `repos.notes.create` | `action: "append"` | the operation `repos.notes.append` |
| `repos.notes.delete` | body `sha` or `object_ref` | query `object_ref` |
| `repos.notes.delete` | body `ref` or `notes_ref` | query `notes_ref` |
| `repos.notes.delete` | body `expected_ref_sha` or `expected_notes_ref_sha` | query `expected_notes_ref_sha` |
| `repos.notes.delete` | body `action`, `note`, `author` | none; the delete commit has the default author |
| `repos.tags.create` | `target` | `ref` |
| `repos.upstream.pull` | `ref`, `after` | none; a pull syncs every branch and tag of the upstream |

A delete with query parameters sends no body.

**Report:** each `ref` or `after` that you remove from the body of an HTTP upstream pull.
Neither field ever had an effect.

### The sync form of `base_repo`

A `base_repo` with an external `provider`, such as `github` or `gitlab`, is a lasting sync.
Send it as `upstream`, and move the other fields:

| Deprecated | Current |
| --- | --- |
| `base_repo.provider`, `owner`, `name` | `upstream.provider`, `owner`, `name` |
| `base_repo.upstream_host` | `upstream.host` |
| `base_repo.auth.auth_type: "public"` | `upstream.access: "public"` |
| `base_repo.default_branch` | top-level `default_branch` |

A `base_repo` without `provider` is a one-time copy of a Code Storage repository: `{ name, ref, auth: { token } }`.

The old SDKs sent a copy with `provider: "code"`, `owner`, and `operation: "fork"`.
It is a copy, not a sync, so keep it in `base_repo`:

- Keep `name` and `auth.token`.
- Send `ref`, or the old `sha` as `ref`.
  Without `ref`, the copy takes the default branch of the source.
- Drop `provider`, `owner`, and `operation`.

`POST /api/repos` rejects an `upstream` with `provider: "code"` with `400`.

## Response names

| Operation | Deprecated | Current |
| --- | --- | --- |
| `repos.list` items, `repos.get` | `url` | `repo_name` |
| `repos.list` items, `repos.get` | `base_repo` | `upstream` |
| `repos.create` | `http_url` | `repo_name` |
| `repos.update` | `url` | `repo_name` |
| `repos.list` items, `repos.get`, `repos.create`, `repos.delete`, `repos.update` | `repo_id`; `repos.update` also `id` | `repo_name` |
| `repos.branches.delete` | `name` | `target_branch` |
| `repos.commits.create`, `repos.commits.createFromDiff`, `repos.commits.restore` | `result.branch` | `result.target_branch` |
| `repos.merge` | `source.branch` | `source.ref` |
| notes writes | `target_ref` | `notes_ref` |
| webhook `push` | `customer_id` | `org` |
| webhook `push` | `repository.url` | `repository.repo_name` |

`url` and `http_url` hold the repository name, so the value does not change.
`repo_id`, `id`, and `customer_id` hold internal IDs, not names.
An older webhook delivery can lack `org` and `repository.repo_name`.
Read `repository.url` when `repository.repo_name` is absent.
`customer_id` is not the organization name, so it cannot replace a missing `org`.

**Ask** when the code stores `repo_id`, `id`, or `customer_id`, or compares it with stored data: "Switch the stored key to `repo_name` (or `org`), or keep reading the deprecated field until you move the stored data?"
Default: keep the deprecated field, and report it.
