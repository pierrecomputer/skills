# SDK rules for every call

Use this file with [methods.md](methods.md) and the file of your language.
This file gives the rules that apply to every call.
Names here use the TypeScript form.
Old Python names are the snake_case form, and old Go names are the PascalCase form.

## Names and results

- Each method takes one params object with `repo_name`, in place of a repository handle.
- Each option takes its wire name, in snake_case.
  [methods.md](methods.md) names each option that changes more, and [names.md](names.md) gives the deprecated wire names.
- Results use wire names in every language: `next_cursor`, not `nextCursor`.
- **Report:** each function of the project that returns an SDK result to its callers, because the shape of that result changes.
- Dates are RFC 3339 strings, and each value has one field.
  `rawDate`, `rawAuthorTime`, `rawCommitterTime`, `rawLastModified`, and `rawPushedAt` become `date`, `author_time`, `committer_time`, `last_modified`, and `pushed_at`.
  **Report:** where the code used the parsed value (a `Date`, a `datetime`, a `time.Time`), parse the string at that use.
- A diff entry's `state` is the Git status, the old `rawState`: `A`, `M`, `D`, `R`, `C`, or `T`.
  The old `state` held a word such as `added`.
  **Report:** compare the first letter where the code compared the word.
  A rename or a copy can carry a score, such as `R100`.
- An enum is open: a new value can arrive.
  Give each branch on an enum a default case.

## Request options

| Old | New |
| --- | --- |
| `ttl` on a call | the request option for the TTL of the token that the SDK signs for that request, in seconds: 1 to 3600, default 300 |
| `signal` on a call (TypeScript) | the request option `signal` |
| `refPolicies` on a call | the request option for ref policies |

The client option for ref policies sets the default of each call.
A client with a fixed `token` rejects ref policies.

## Errors

| Old | New |
| --- | --- |
| `ApiError` | `APIError`, or its subclass for the status: `BadRequestError` 400, `AuthenticationError` 401, `PermissionDeniedError` 403, `NotFoundError` 404, `ConflictError` 409, `RateLimitError` 429, `InternalServerError` 5xx |
| `RefUpdateError` | `RefGuardError` for a failed SHA guard, a `ConflictError` with `guard` and `current_sha`; `RefUpdateError` for another failed ref update, an `APIError` with `result` and `commit` |
| a plain error with a message | the class and its `code` |

- `APIError` has `status`, `code`, `title`, `detail`, `request_id`, `headers`, `body`, and `doc_url`.
- Another status, such as 412, 416, or 422, gives a plain `APIError`.
- The SDK picks `RefGuardError` when the code is `ref_guard_failed`, also when the body has `result`.
  Otherwise it picks `RefUpdateError` when the body has `result`, at any status.
- The old `reason` was the ref-update status.
  A stale guard, such as `expected_target_sha` (the old `expectedHeadSha`), now raises `RefGuardError`.
  Where the code read `reason` `conflict` after a guarded write, check `RefGuardError`.
  `guard` names the request field of the guard, and `current_sha` is the current commit of the ref; send it as the guard of the next attempt.
  For another failed ref update, read `result.status` where the code read `reason`, for example `conflict` for a patch that does not apply.
  `code` can be absent on a `RefUpdateError`, so do not use it in place of `reason`.
  `result` is a `RefUpdateResult`, or a `NotesWriteResult` for a notes write; both have `status`, so read it with no cast or decode.
- The old `RefUpdateError` also had `status`: a string label, such as `conflict`, from which the old SDK took `reason`.
  Change a check of the old `status` in the same way as a check of `reason` (above).
- The new `RefUpdateError` and `RefGuardError` are `APIError`s, and their `status` is the HTTP status, such as `409`.
  The old `RefUpdateError` was not an `ApiError`.
  **Report:** each check of the class `APIError` or of a number `status` that a failed ref update can reach, for example `status === 409` around a commit.
  The old check never matched a failed ref update, so the code that it guards, such as a retry, starts to run.
- A `404` has the code `repository_not_found` when the repository does not exist, or when the path does not match the token's `repo` claim.
  A missing object in the repository, such as a note, a ref, or a file, has the code `not_found`.
  **Report:** a check of a message such as "note not found" becomes a check of `NotFoundError` whose `code` is not `repository_not_found`.
- `MergeConflictError` is a `ConflictError` with `conflict_type`, `conflict_paths`, and `merge_base_sha`.
- On the HTTP API, a failed SHA guard is a 409 with the code `ref_guard_failed`, `guard`, and `current_sha`.
  A merge guard failure had the code `conflict` before; old code that compares the code with `conflict` after a merge guard needs the new code.
  A commit write, a restore, or a notes write keeps its `result`, and `result.old_sha` gives the current commit.
- `APIConnectionError` means that no response came. `APITimeoutError` is its subclass for a timeout.
- **Report:** replace each check of message text with a check of the class and `code`.

## Retries and timeouts

The old SDKs did not retry.
The new SDK retries twice, with backoff, on a connection error, 408, 429, 5xx, and 409 with the code `repository_thawing` or `sync_in_progress`.
It retries only `GET`, `HEAD`, `PUT`, and `DELETE` requests.
Each attempt has a 60-second timeout.

The SDK never retries these operations: `repos.create`, `repos.update`, `repos.archive.get`, `repos.grep`, `repos.merge`, `repos.branches.create`, `repos.commits.create`, `repos.commits.createFromDiff`, `repos.commits.restore`, `repos.notes.create`, `repos.notes.append`, `repos.tags.create`, `repos.gitCredentials.create`, and `repos.upstream.pull`.
A retry loop around one of them stays as it is.

A retried `DELETE` can get `404` when the first attempt worked and its response was lost.
**Report** each delete whose `404` the code treats as a failure.

**Ask** when the code wraps another SDK call in its own retry loop: "Keep your loop and turn off SDK retries for that call, or remove your loop?"
Default: keep the loop, and set the call's max retries to 0.

**Ask** when the code sends large commits or reads large archives: "Can these calls take more than 60 seconds? If so, which timeout do you want?"
Default: keep 60 seconds.

## Pagination

A list call returns a page: `data` holds the items, with `has_more`, `next_cursor`, and `body`.
The old result had a named array (`repos`, `branches`, `commits`, `tags`, `refs`, `entries`, `files`) and `nextCursor` and `hasMore`.
The other fields of the body are in `page.body`, for example `paths` of `repos.files.list` and `commits` of `repos.files.metadata.list`.
Replace a cursor loop with the iteration of your language.
A call that read one page still reads one page.

A loop can treat an error of the first page apart from an error of a later page, for example a `404` on the first page that means an empty list.
Then make the list call inside the old error handling: the call gives the first page.
After the handling, iterate over that page.
The iteration gives the items of that page and of each next page, and an error of a later page comes from the iteration.
The file of your language shows it.

## Older versions of the old SDK

The references describe the old versions that the `metadata` of `SKILL.md` names.
An older version can lack an option, a method, or a result field.
It can also ignore an option that the code passes, for example through a type cast or an untyped object.
So when the project installs an older version, compare each call of the project with the source of the installed version, and list each difference.
The installed package holds that source.

A library that other projects install can declare a version range, such as `^1.9.0`.
Its users get the newest version that the range allows, not the version in the library's lockfile.
For a library, also compare each call with that newest version.

**Ask** when the installed version did not send an option that the code passes: "Version <version> did not send `<option>`, so the call ignored it. Send it now, or leave it out?"
Default: send it, because the code asks for it.
When the newest version that a library's range allows sends the option, the library's users already get that behavior.
Then send it with no question, and report it.

**Report** each other difference that reaches a call of the project, with the installed version.

## Your own type for the client

A project can describe the old client, or a `Repo`, with its own type, so that a test can give a fake in place of the client: a TypeScript object type, a Python `Protocol`, or a Go interface.
The type and each fake change with the code.

- Give the type the shape of the new client: the path of each method that the code calls, such as `repos.commits.create`, the params with wire names, and the result fields that the code reads.
  A chain of old calls becomes one call, for example `repo().createCommit().send()` becomes `repos.commits.create` with `files`.
- Keep the name of the type, and only the methods that the code calls.
- Use the params and result types of the new SDK where the type holds a whole params object or result.
- The real client must still fit the type.
  The type checker shows it where the code passes the real client.
- Change each fake to the new type.
  The file of your language shows a fake, in "Test mocks".

## A library's public API

A library that other projects install can name old SDK types in its public API: a re-export, such as `GitStorage` or `CommitSignature`, the params or the result of an exported function, an exported option type, or an exported type for the client.
The move changes that API for each user of the library.

- Change each old type to its new type in the table of your language file.
  A `Repo` follows the `repo` **Ask** in [methods.md](methods.md).
- Keep the names of the exported functions, options, and fields.
- Change each example of the old client in the library's README and docs.
- **Ask** when the library re-exports an old SDK name: "Export the new names, such as `Pierre` and `GitIdentity`, or keep each old name as an alias of its new type?"
  Default: the new names.
  An old name on a type with a new shape hides the change from your users.
- **Report:** each export whose type changes.
  A release with this change breaks the code of the library's users.
