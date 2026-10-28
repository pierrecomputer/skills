# Python

Use this file with [methods.md](methods.md) and [sdk.md](sdk.md).
They say what changes; this file shows how to write it.

## Package

- The distribution is `pierre-sdk`, and the import is `pierre`.
  `requires-python` in its metadata gives the Python versions that it supports.
- The distribution is not published yet.
  Do not install a package named `pierre-sdk`, and do not move the Python code.
  Tell the user that the Python move waits for the first release of the Python SDK.
- Import the local functions from their modules: `from pierre.git import remote_url`, `from pierre.auth import create_token`, and `from pierre.webhooks import verify_webhook`.
  `import pierre` does not load these modules.
  A variable named `pierre` also hides the module, so use the `from` form.
- Find the project's Python version in `requires-python` of `pyproject.toml`, `.python-version`, `.tool-versions`, the CI configuration, or a `Dockerfile`.
  **Ask** when a file names a Python version outside the `requires-python` range of `pierre-sdk`, or when the project's `requires-python` allows one: "Update Python to a version that `pierre-sdk` supports?"
  Default: stop, and report that the move needs the update.
  When no file names a version, continue, and report the requirement.
- Read the installed version of `pierre-storage` with `pip show pierre-storage` in the project's environment, or from the lockfile, such as `uv.lock`, `poetry.lock`, or a `requirements` file.
  The source of that version is the `pierre_storage` folder that `pip show -f pierre-storage` lists.
- The old package brought `pyjwt` and `pydantic`.
  The new one does not.
  **Report:** when the code imports `jwt` or `pydantic` and the project does not declare it, add it to the project's dependencies.

## Client

The old SDK was async only.
Use `AsyncPierre`, so each `await` stays.

```python
# pierre-storage
from pierre_storage import GitStorage

store = GitStorage({"name": "acme", "key": private_key_pem})

# pierre-sdk
from pierre import AsyncPierre

async with AsyncPierre(org="acme", private_key=private_key_pem) as pierre:
    ...
async with AsyncPierre() as pierre:  # reads the PIERRE_* variables
    ...
```

| Old `GitStorageOptions` key | New keyword argument |
| --- | --- |
| `name` | `org` |
| `key` | `private_key` |
| `token` | `token` |
| `api_base_url` | `base_url` |
| `storage_base_url` | `git_host` |
| `api_version`, `default_ttl` | none: remove them |

- The client keeps one httpx client.
  `async with` closes it.
  The client has no `close()` method.
  For a client that lives as long as the process, create it once.
  To control the connections, pass your own `http_client=httpx.AsyncClient(...)`, and close it yourself.
- `Pierre` is the blocking client, with the same methods.
  **Report** each place where the code wraps one call in `asyncio.run`; `Pierre` is simpler there.
- A keyword argument that is `None` falls back to its `PIERRE_*` variable, then to the default.
  An empty variable counts as not set.
  An empty string is an error, but the old SDK used the default for it.
  Pass `value or None` where the old code passed a value that can be empty, such as `os.environ.get(...)`.

## Types

| Old (`pierre_storage`) | New |
| --- | --- |
| `GitStorage` | `AsyncPierre`, or `Pierre` in blocking code |
| `Repo` | `str`, the repository name, or a local `Repo` type (below) |
| `CommitSignature` | `GitIdentity` |
| the builder's files | `List[CommitFile]` |
| `CommitResult` | `CommitWriteResponse` |
| `RefUpdate` | `RefUpdateResult` |
| `CommitInfo` | `Commit` |
| `GetBranchDiffResult`, `GetCommitDiffResult` | `ReposDiffGetResponse` |
| `FileDiff` | `FileDiff`, with `state` as a Git status letter |
| `DiffFileState` | `str` |
| `WebhookPushEvent` | `PushWebhookEvent` |
| `WebhookValidationResult` | none: `verify_webhook` returns the event or raises |
| `ApiError` | `APIError` |

Import the result types from `pierre.types`, and the client and the error classes from `pierre`.

## A local `Repo` type

Use this form when the user keeps the function signatures ([methods.md](methods.md), "`repo` → none").

```python
from dataclasses import dataclass
from typing import Optional

from pierre import AsyncPierre, NotFoundError


@dataclass(frozen=True)
class Repo:
    pierre: AsyncPierre
    id: str
    default_branch: str


async def find_repo(pierre: AsyncPierre, name: str) -> Optional[Repo]:
    try:
        found = await pierre.repos.get(repo_name=name)
    except NotFoundError:
        return None
    return Repo(pierre, found["repo_name"], found["default_branch"])


async def start_draft(repo: Repo, branch: str) -> None:
    await repo.pierre.repos.branches.create(
        repo_name=repo.id, base_ref=repo.default_branch, target_branch=branch, timeout=180
    )
```

## Calls

```python
# pierre-storage
repo = await store.create_repo(id=name, default_branch="main")
await repo.create_branch(base_ref="main", target_branch="feat", ttl=600)
same = store.repo(id=name)  # a handle, no request

# pierre-sdk
created = await pierre.repos.create(repo_name=name, default_branch="main")
await pierre.repos.branches.create(
    repo_name=created["repo_name"], base_ref="main", target_branch="feat", token_ttl=600
)
```

Each old method is the snake_case form of its name in [methods.md](methods.md), for example `find_one` and `get_branch_diff`.
The new method is the catalog path with snake_case segments, for example `pierre.repos.git_credentials.create`.

- Every argument is a keyword argument.
- The per-call options are `timeout` (seconds), `max_retries`, `extra_headers`, `token_ttl` (seconds), and `ref_policies`.
  No operation accepts `idempotency_key` today, so do not pass it.
- `None` changed meaning.
  The old SDK did not send an argument that was `None`.
  The new SDK sends `None` in a body field as `null`, and raises `PierreError` for `None` in a query or header argument.
  **Report:** remove each argument that is `None`.
  For a variable that can be `None`, pass `value if value is not None else NOT_GIVEN` (`from pierre import NOT_GIVEN`).
- Results are `TypedDict`s with the wire names.
- `pierre.with_raw_response.repos.create(...)` gives a `RawResponse` with `data`, `response`, `headers`, and `request_id`.
  Read a status, such as `201` or `200`, from `response.status_code`.

## Timeouts

The old SDK had fixed timeouts.
The new SDK has 60 seconds for each attempt.

**Report:** pass `timeout=180` to the calls that had 180 seconds: `create_branch`, `merge`, `preview_merge`, `create_tag`, `restore_commit`, the commit builder's `send`, and `create_commit_from_diff`.
The other old timeouts were 30 or 60 seconds, so they need no change.

## Errors

```python
from pierre import NotFoundError

try:
    repo = await pierre.repos.get(repo_name=name)
except NotFoundError:
    repo = None
```

The example keeps the `None` result of `find_one` for a missing repository.

| Old | New |
| --- | --- |
| `ApiError.status_code` | `APIError.status` |
| `ApiError.response` | `APIError.body`, `headers`, `request_id` |
| `RefUpdateError.reason` | for a stale guard, `RefGuardError` with `guard` and `current_sha`; else `result["status"]`, and `code` can be absent |
| `RefUpdateError.status`, a `str` | as `reason`; the new `status` is the HTTP status, an `int` ([sdk.md](sdk.md)) |
| `RefUpdateError.ref_update` | `result` |
| `httpx.HTTPStatusError` from a read | the `APIError` subclass for the status |
| `ValueError` and other plain errors with a message | the class and its `code` |

`result` has the type `Union[RefUpdateResult, NotesWriteResult]`.
Read `result["status"]` with no cast:

```python
from pierre import NotFoundError, RefGuardError, RefUpdateError


def is_stale_head(err: Exception) -> bool:
    return isinstance(err, RefGuardError) and err.guard == "expected_target_sha"


def is_patch_conflict(err: Exception) -> bool:
    return isinstance(err, RefUpdateError) and err.result["status"] == "conflict"


def is_missing_object(err: Exception) -> bool:
    return isinstance(err, NotFoundError) and err.code != "repository_not_found"
```

## Pagination

```python
async for repo in pierre.repos.list(limit=100):
    ...  # each item of every page

page = await pierre.repos.list()  # one page
page.data
page.next_cursor
```

With `Pierre`, use `for` in place of `async for`.

A loop that treats an error of the first page apart from a later error:

```python
first: Optional[AsyncPage[Commit, ReposCommitsListResponse]]
try:
    first = await pierre.repos.commits.list(repo_name=name, ref=branch)
except NotFoundError:
    first = None
commits = [] if first is None else [commit async for commit in first]  # each page
```

`AsyncPage` is in `pierre`, and `Commit` and `ReposCommitsListResponse` are in `pierre.types`.

## Commits

```python
await pierre.repos.commits.create(
    repo_name=name,
    target_branch="main",
    commit_message="Update docs",
    author={"name": "Bot", "email": "bot@example.com"},
    files=[
        {"path": "README.md", "content": "# Web"},
        {"path": "old.txt", "operation": "delete"},
    ],
)
```

- `content` can be `str`, `bytes`, a binary file, an iterable of `bytes`, an async iterable, or a function that returns one of them.
- The old builder removed a leading `/` from each path.
  The new SDK sends the path as it is.
  **Report:** remove a leading `/` with `path.lstrip("/")` where a path can have one.
- For text in another encoding, pass `text.encode(encoding)`.

## Local functions

- `await remote_url(pierre, repo_name=name, ttl=3600)` with `AsyncPierre`; no `await` with `Pierre`.
- `create_token(pierre, scopes=[...], repo=name, ttl=900)` is always synchronous.
  It needs a client with a private key.
- `verify_webhook(payload=raw_body, headers=request.headers, secret=secret)` returns the event dict.
  `payload` is the raw body as `str` or `bytes`.

## Files

`repos.files.get` and `repos.archive.get` return an `httpx.Response` with an open stream.
Read it with `await res.aread()` or `async for chunk in res.aiter_bytes()`, then call `await res.aclose()`.
The conditional and range headers are keyword arguments, such as `range=` and `if_none_match=`, in place of the old `headers` dict.

## Dates and enums

- Python 3.10 does not parse a final `Z` with `datetime.fromisoformat`.
  There, use `datetime.fromisoformat(value.replace("Z", "+00:00"))`.
- `DiffFileState` and `GitFileMode` members become strings, for example `"A"` and `"100644"`.

## Test mocks

A test that mocks the old client or a `Repo` changes with the code.
When the project has its own type for the client ([sdk.md](sdk.md), "Your own type for the client"), write it as a `Protocol` with the path of each method that the code calls:

```python
from typing import Awaitable, List, Protocol

from pierre import AsyncPage
from pierre.types import Commit, CommitFile, CommitWriteResponse, GitIdentity, ReposCommitsListResponse


class StoreCommits(Protocol):
    async def create(
        self,
        *,
        repo_name: str,
        author: GitIdentity,
        commit_message: str,
        target_branch: str,
        files: List[CommitFile],
    ) -> CommitWriteResponse: ...

    def list(
        self, *, repo_name: str, ref: str
    ) -> Awaitable[AsyncPage[Commit, ReposCommitsListResponse]]: ...


class StoreRepos(Protocol):
    @property
    def commits(self) -> StoreCommits: ...


class StoreClient(Protocol):
    @property
    def repos(self) -> StoreRepos: ...
```

- Name only the arguments that the code passes.
  The SDK method takes more keyword arguments, and `AsyncPierre` still fits the `Protocol`.
- A list method of `AsyncPierre` returns an `AsyncPaginator`.
  Type it as `Awaitable[AsyncPage[...]]` when the code awaits a page, or as `AsyncIterable[Item]` when the code uses `async for` on the call.
- A namespace is a `@property` in the `Protocol`, so a fake can hold it in a plain attribute.

A fake:

```python
import httpx

from pierre import RefUpdateError


async def no_next_page(cursor: str) -> AsyncPage[Commit, ReposCommitsListResponse]:
    raise AssertionError("no next page")


class FakeCommits:
    def __init__(self) -> None:
        self.written: List[str] = []

    async def create(
        self,
        *,
        repo_name: str,
        author: GitIdentity,
        commit_message: str,
        target_branch: str,
        files: List[CommitFile],
    ) -> CommitWriteResponse:
        self.written.extend(file["path"] for file in files)
        raise RefUpdateError(409, {"result": {"status": "conflict"}})

    async def list(
        self, *, repo_name: str, ref: str
    ) -> AsyncPage[Commit, ReposCommitsListResponse]:
        body: ReposCommitsListResponse = {"commits": [], "has_more": False}
        return AsyncPage(
            data=body["commits"],
            has_more=False,
            next_cursor=None,
            body=body,
            response=httpx.Response(200),
            fetch=no_next_page,
        )


class FakeRepos:
    def __init__(self) -> None:
        self.commits = FakeCommits()


class FakeClient:
    def __init__(self) -> None:
        self.repos = FakeRepos()
```

- A mock of the old commit builder becomes a mock of `repos.commits.create`.
  `files` can be an iterable or an async iterable, so read it as the code passes it.
- Make each error from the status and the body of the response, with the class that the SDK picks ([sdk.md](sdk.md)): `NotFoundError(404, {"code": "not_found"})`, or `APIError(412, {"code": "precondition_failed"})` for a status with no subclass.
  The old `RefUpdateError(message, status="conflict")` becomes `RefUpdateError(409, {"result": {"status": "conflict"}})`.
- A failed SHA guard is a `RefGuardError`: `RefGuardError(409, {"code": "ref_guard_failed", "guard": "expected_target_sha", "current_sha": sha})`.
- With `Pierre`, the methods are not `async`, and a list method returns a `Page` with the same arguments.

## Find the remaining uses

Run the project's type checker if it has one, such as `mypy` or `pyright`.
Otherwise search for `pierre_storage`, `GitStorage`, and `create_client`, and run the tests.
