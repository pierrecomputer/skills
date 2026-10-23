# HTTP API

Use this file for code that sends HTTP requests to the Code Storage API directly.
The public OpenAPI document is `https://code.storage/openapi.json`.
A deprecated route in it has `deprecated: true` and an `x-deprecation-message` that names the current route.

## Routes

Each `/api/v1/repos/<path>` route moves to `/api/repos/{repo_name}/<path>`.
The method, the query, the body, the response, and the scopes do not change, except in the table below.

- `{repo_name}` must equal the token's `repo` claim exactly.
  Percent-encode a `/` in it as `%2F`, for example `/api/repos/team%2Fweb/branches`.
- Another value returns `404` with the code `repository_not_found`.
- The token does not change.
  The old route read the repository from the same `repo` claim.
- **Ask** when the code that builds the request does not have the repository name: "Pass the repository name to this code, or read it from the token's `repo` claim?"
  Recommend to pass the name.
  Default: decode the token's payload, and read `repo`.
  Do not log or print the token.

| Deprecated route | Current route | Other change |
| --- | --- | --- |
| `GET /api/v1/repos` | `GET /api/repos` | none |
| `POST /api/v1/repos` | `POST /api/repos` | none |
| `GET /api/v1/repo` | `GET /api/repos/{repo_name}` | none |
| `PATCH /api/v1/repo` | `PATCH /api/repos/{repo_name}` | none |
| `DELETE /api/v1/repos/delete` | `DELETE /api/repos/{repo_name}` | none |
| `POST /api/v1/repos/pull-upstream`, `POST /api/repos/{repo_name}/pull-upstream` | `POST /api/repos/{repo_name}/upstream/pull` | remove the body fields `ref` and `after` ([names.md](names.md)) |
| `DELETE /api/v1/repos/base`, `DELETE /api/repos/{repo_name}/base` | `DELETE /api/repos/{repo_name}/upstream` | none |
| `GET /api/v1/repos/branches/diff`, `GET /api/repos/{repo_name}/branches/diff` | `GET /api/repos/{repo_name}/diff` | see "Branch diff" |
| `POST /api/v1/repos/git-credentials` | `POST /api/repos/{repo_name}/git-credentials` | remove the body field `repo_id` |
| `PUT /api/v1/repos/git-credentials` | `PUT /api/repos/{repo_name}/git-credentials/{git_credential_id}` | the body field `id` moves to the path |
| `DELETE /api/v1/repos/git-credentials` | `DELETE /api/repos/{repo_name}/git-credentials/{git_credential_id}` | the body field `id` moves to the path; no body |
| `DELETE /api/v1/repos/tags` | `DELETE /api/repos/{repo_name}/tags/{tag_name}` | the body field `name` moves to the path; no body |
| `POST /api/v1/repos/notes` with `"action": "append"` | `POST /api/repos/{repo_name}/notes/append` | remove `action` |

Percent-encode a tag name in the path, so a `/` is `%2F`.

The old credential update and delete needed only the credential ID.
The old SDKs signed those requests with the `repo` claim `org`, not a repository name.
The current routes also need the repository name in the path and in the token's `repo` claim.
**Ask** when the code keeps only the credential ID: "Where can the code get the repository of each credential?"
Default: keep those requests on the old route, and report them.

`POST /api/repos` takes the new repository's name from the `repo` claim.
The optional body field `repo_name` must equal the claim, or the request returns `400` with the code `repo_name_mismatch`.

For deprecated parameters and fields on any route, use [names.md](names.md).

## Branch diff

`GET /diff` replaces the branch diff.
Send `branch` as `ref`, `base` as `base_ref`, `ephemeral` as `ref_is_ephemeral`, and `ephemeral_base` as `base_is_ephemeral`.

Without `base`, the branch diff compared the branch with the repository's default branch.
Without `base_ref`, `/diff` returns the diff of one commit.
So when the old request has no `base`, send the default branch as `base_ref`.
Use the default branch that the code already knows.
Otherwise, read `default_branch` from `GET /api/repos/{repo_name}`.

The response has `sha` and `base_sha`, the resolved commits.
It does not have `branch` and `base`.
Code that reads those fields must use the values that it sent.

## Errors

Error responses are problem details (`application/problem+json`) with `type`, `title`, `status`, `code`, `detail`, and `instance`.
Branch on `code`, not on the text of `detail` or `error`.
`error` is a deprecated copy of `detail`.
**Report:** each check of message text that you replace.

- When the docs do not name the code of a failure, branch on the status, and report it.
- A `404` has the code `repository_not_found` when the repository does not exist, or when the path does not match the token's `repo` claim.
  Do not treat it as a missing branch, tag, or note.
- A missing object in the repository, such as a note, a ref, or a file, gives a `404` with the code `not_found`.
- **Report:** a check of a message such as "note not found" becomes a check of `404` with a `code` that is not `repository_not_found`.

## Tokens

The authentication page, `https://code.storage/docs/platform/authentication.md`, gives the scopes and the `repo` claim of each operation.
