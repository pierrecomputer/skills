# Go

Use this file with [methods.md](methods.md) and [sdk.md](sdk.md).
They say what changes; this file shows how to write it.

## Module

- `go list -m github.com/pierrecomputer/sdk/packages/code-storage-go` gives the version of the old module that the project builds with.
  `go list -m -f '{{.Dir}}' github.com/pierrecomputer/sdk/packages/code-storage-go` gives the folder of its source.
- The module is `github.com/pierrecomputer/sdk/go`, with the package `pierre`.
  The packages `option`, `git`, `auth`, and `webhooks` are under the same path.
  `go list -m -versions github.com/pierrecomputer/sdk/go` lists its versions.
- The module is not published yet, so `go get` cannot download it.
  Until `go list` lists a version, do not move the Go code.
  Tell the user that the Go move waits for the first release of the Go SDK.
- The `go` directive in the `go.mod` of the module gives the oldest Go version that it needs.
  **Ask** when the project's `go` directive or toolchain is older: "Update Go to the version that the module needs?"
  Default: stop, and report that the move needs the update.
- After the import path changes, `go build ./... && go vet ./...` lists each remaining use.
  Run `go mod tidy` when no use of the old module remains.

## Client

```go
// code-storage-go
import storage "github.com/pierrecomputer/sdk/packages/code-storage-go"

store, err := storage.NewClient(storage.Options{Name: "acme", Key: pem})

// pierre
import (
	pierre "github.com/pierrecomputer/sdk/go"
	"github.com/pierrecomputer/sdk/go/option"
)

client, err := pierre.NewClient(option.WithOrg("acme"), option.WithPrivateKey(pem))
clientFromEnv, err := pierre.NewClient() // reads the PIERRE_* variables
```

| Old `storage.Options` field | New option |
| --- | --- |
| `Name` | `option.WithOrg` |
| `Key` | `option.WithPrivateKey` |
| `Token` | `option.WithToken` |
| `APIBaseURL` | `option.WithBaseURL` |
| `StorageBaseURL` | `option.WithGitHost` |
| `HTTPClient` | `option.WithHTTPClient` |
| `APIVersion`, `DefaultTTL` | none: remove them |

`NewClient` returns an error for a key that it cannot read.

An option that the code leaves out falls back to its `PIERRE_*` variable, then to the default.
An empty variable counts as not set.
`option.WithBaseURL("")` and `option.WithGitHost("")` give the default host of the organization, as an empty `APIBaseURL` and `StorageBaseURL` did.
They do not read `PIERRE_API_BASE_URL` or `PIERRE_GIT_HOST`.

## Types

| Old (`storage`) | New (`pierre`) |
| --- | --- |
| `*storage.Client` | `*pierre.Client` |
| `*storage.Repo` | `string`, the repository name, or a local `Repo` type (below) |
| `CommitSignature`, `NoteAuthor` | `pierre.GitIdentity`; wrap it in `pierre.Value` where the field is an `Opt` |
| `*CommitBuilder` | `[]pierre.CommitFile` |
| `CommitResult` | `*pierre.CommitWriteResponse` |
| `RefUpdate` | `pierre.RefUpdateResult` |
| `CommitInfo` | `pierre.Commit`; `Date` is a `string` |
| `GetBranchDiffResult`, `GetCommitDiffResult` | `*pierre.ReposDiffGetResponse` |
| `FileDiff`, `DiffFileState` | `pierre.FileDiff`; `State` is a `string`, a Git status letter |
| `WebhookPushEvent` | `*pierre.GitPushEvent`; `Repository.RepoName` is a `pierre.Opt[string]` |
| `WebhookValidation` | none: `webhooks.VerifyWebhook` returns the event or an error |
| `*storage.APIError` | `*pierre.APIError` |

## A local `Repo` type

Use this form when the user keeps the function signatures ([methods.md](methods.md), "`repo` → none").

```go
// Repo is a repository and the client that reaches it.
type Repo struct {
	Client        *pierre.Client
	ID            string
	DefaultBranch string
}

// FindRepo returns nil when the repository does not exist.
func FindRepo(ctx context.Context, client *pierre.Client, name string) (*Repo, error) {
	found, err := client.Repos.Get(ctx, pierre.ReposGetParams{RepoName: name})
	var notFound *pierre.NotFoundError
	if errors.As(err, &notFound) {
		return nil, nil
	}
	if err != nil {
		return nil, err
	}
	return &Repo{Client: client, ID: found.RepoName, DefaultBranch: found.DefaultBranch}, nil
}

// StartDraft creates a branch from the default branch.
func StartDraft(ctx context.Context, repo *Repo, branch string) error {
	_, err := repo.Client.Repos.Branches.Create(ctx, pierre.ReposBranchesCreateParams{
		RepoName:     repo.ID,
		BaseRef:      pierre.String(repo.DefaultBranch),
		TargetBranch: pierre.String(branch),
	})
	return err
}
```

## Calls

```go
// code-storage-go
repo, err := store.CreateRepo(ctx, storage.CreateRepoOptions{ID: name, DefaultBranch: "main"})
_, err = repo.CreateBranch(ctx, storage.CreateBranchOptions{BaseRef: "main", TargetBranch: "feat"})
same, err := store.Repo(storage.RepoOptions{ID: name}) // a handle, no request

// pierre
created, err := client.Repos.Create(ctx, pierre.ReposCreateParams{
	RepoName:      pierre.String(name),
	DefaultBranch: pierre.String("main"),
})
_, err = client.Repos.Branches.Create(ctx, pierre.ReposBranchesCreateParams{
	RepoName:     created.RepoName,
	BaseRef:      pierre.String("main"),
	TargetBranch: pierre.String("feat"),
}, option.WithTokenTTL(10*time.Minute))
```

- Each old method is the PascalCase form of its name in [methods.md](methods.md), except `FileStream`, `ArchiveStream`, `RemoteURL`, `EphemeralRemoteURL`, `ImportRemoteURL`, and `Config`.
- Each new method is the catalog path in PascalCase on the client, for example `client.Repos.GitCredentials.Create`.
- Each method takes `ctx`, a params struct named after the operation (`pierre.<Operation>Params`), and request options.
- An optional field is `pierre.Opt[T]`.
  Set it with `pierre.String`, `pierre.Int` (an `int64`), `pierre.Bool`, `pierre.Float`, or `pierre.Value`.
  Leave it zero to omit it.
  `pierre.Null[T]()` sends `null`.
  Read it with `Get()`, which returns the value and a `bool`.
- The old SDK read an empty string or a `nil` pointer as "not set".
  Leave the new field zero in those cases.
  `pierre.String("")` sends an empty value, so use a helper where a value can be empty:

  ```go
  // optional returns an unset field for an empty string.
  func optional(value string) pierre.Opt[string] {
  	if value == "" {
  		return pierre.Opt[string]{}
  	}
  	return pierre.String(value)
  }
  ```
- The params have no deprecated top-level fields.
  Use the current names in [names.md](names.md).
- The per-call options are `option.WithTokenTTL` (whole seconds, 1 second to 1 hour), `option.WithTimeout`, `option.WithMaxRetries`, `option.WithHeader`, `option.WithRefPolicies`, and `option.WithResponseInto`.
- To read a status, such as `201` or `200` from `Repos.Create`, pass `option.WithResponseInto(&res)` with `var res *http.Response`.

## Errors

```go
repo, err := client.Repos.Get(ctx, pierre.ReposGetParams{RepoName: name})
var notFound *pierre.NotFoundError
if errors.As(err, &notFound) {
	repo, err = nil, nil
}
```

The example keeps the `nil` result of `FindOne` for a missing repository.

- Each status family has its own type: `*pierre.BadRequestError`, `*pierre.AuthenticationError`, `*pierre.PermissionDeniedError`, `*pierre.NotFoundError`, `*pierre.ConflictError`, `*pierre.RateLimitError`, and `*pierre.InternalServerError`.
  Each one embeds `*pierre.APIError`.
- `*pierre.RefUpdateError` embeds `*pierre.APIError` directly.
  So `errors.As` for `*pierre.ConflictError` does not match it.
  Check `*pierre.RefUpdateError` first.
- `*pierre.RefGuardError` is a failed SHA guard, with the code `ref_guard_failed`, `Guard`, and `CurrentSHA`.
  It unwraps to a `*pierre.ConflictError`, so check it before `*pierre.ConflictError`.
- `*pierre.APIError` has `Status`, `Code`, `Title`, `Detail`, `RequestID`, `Headers`, `Body`, and `DocURL`.
  The old `Message`, `StatusText`, `Method`, and `URL` have no equal; use `Title` and `Detail`.
- The old `RefUpdateError.Reason` is the `Status` of the ref update, not `Code`.
  `Code` can be empty.
  `Result` is a decoded `pierre.RefUpdateResult`, and `Commit` is a `*pierre.WrittenCommit` or `nil`.
  `Body` holds the response as it arrived.
- The old `RefUpdateError.Status` was a `string`, the label of `Reason`; read `Result.Status` in its place.
  The new `Status` is the `int` HTTP status of the embedded `*pierre.APIError`, so the compiler shows each old comparison with a string.
- The old `*storage.RefUpdateError` did not unwrap to `*storage.APIError`.
  Now `errors.As` for `*pierre.APIError` matches a `*pierre.RefUpdateError` and a `*pierre.RefGuardError` ([sdk.md](sdk.md), the **Report** on a number `status`).

```go
// isStaleHead reports a commit whose expected target SHA is not the branch head.
func isStaleHead(err error) bool {
	var guardErr *pierre.RefGuardError
	return errors.As(err, &guardErr) && guardErr.Guard == "expected_target_sha"
}

// isPatchConflict reports a diff that does not apply.
func isPatchConflict(err error) bool {
	var refErr *pierre.RefUpdateError
	return errors.As(err, &refErr) && refErr.Result.Status == "conflict"
}

// isMissingObject reports a missing note, ref, or file in a repository that exists.
func isMissingObject(err error) bool {
	var notFound *pierre.NotFoundError
	return errors.As(err, &notFound) && notFound.Code != "repository_not_found"
}
```

## Pagination

```go
page, err := client.Repos.List(ctx, pierre.ReposListParams{})
for repo, err := range page.All(ctx) {
	// each item of every page
}
```

`page.Data`, `page.NextCursor`, `page.HasNextPage()`, `page.GetNextPage(ctx)`, and `page.IterPages(ctx)` read one page at a time.

The list call returns the error of the first page, and `page.All` yields the error of a later page.
So a loop that treats an error of the first page apart from a later error keeps the two checks apart:

```go
page, err := client.Repos.Commits.List(ctx, pierre.ReposCommitsListParams{RepoName: name, Ref: pierre.String(branch)})
var notFound *pierre.NotFoundError
if errors.As(err, &notFound) {
	return nil, nil
}
if err != nil {
	return nil, err
}
var commits []pierre.Commit
for commit, err := range page.All(ctx) { // the first page and each next page
	if err != nil {
		return nil, err
	}
	commits = append(commits, commit)
}
```

## Commits

```go
client.Repos.Commits.Create(ctx, pierre.ReposCommitsCreateParams{
	RepoName:          name,
	TargetBranch:      "main",
	CommitMessage:     "Update docs",
	Author:            pierre.GitIdentity{Name: "Bot", Email: "bot@example.com"},
	ExpectedTargetSHA: pierre.String(sha),
	Files: []pierre.CommitFile{
		{Path: "README.md", Content: strings.NewReader("# Web")},
		{Path: "old.txt", Operation: "delete"},
	},
})
```

- `AddFile(path, r, opts)` becomes `{Path, Content: r, Mode}`, and `AddFileFromBytes` uses `bytes.NewReader`.
- `Open func() (io.ReadCloser, error)` in place of `Content` opens a file only when the SDK sends it.
- `FileSeq iter.Seq[pierre.CommitFile]` replaces `Files` for a sequence.
- The SDK calls `Open` and reads `Content` on its own goroutines.
- `Create` returns the error that the old `Err()` and `Send` returned.

## Local functions

```go
url, err := git.RemoteURL(ctx, client, git.RemoteURLParams{RepoName: name, TTL: time.Hour, Kind: git.KindEphemeral})
token, err := auth.CreateToken(ctx, client, auth.TokenParams{Scopes: []string{"git:read"}, Repo: name})
event, err := webhooks.VerifyWebhook(webhooks.VerifyWebhookParams{Payload: body, Header: r.Header, Secret: secret})
```

- `Kind` is `git.KindDefault`, `git.KindEphemeral`, or `git.KindImport`.
- `RefPolicies` take `option.RefPolicy{Pattern, Ops}`.
  `Ops` is a `[]string`, so a `storage.Ops` value becomes a `[]string` of the op strings, such as `"no-push"` for `storage.OpNoPush` ([sdk.md](sdk.md)).
- `storage.WebhookValidationOptions{MaxAgeSeconds: n}` becomes `Tolerance: pierre.Value(time.Duration(n) * time.Second)` in `webhooks.VerifyWebhookParams`.
  The old `MaxAgeSeconds: 0` was the default, 300 seconds, so leave `Tolerance` unset for it.
  A negative `MaxAgeSeconds` turned off the check: follow the `maxAgeSeconds` **Ask** in [methods.md](methods.md).
- `VerifyWebhook` returns a `webhooks.Event`.
  Use a type switch on `*pierre.GitPushEvent`, `*pierre.GitSyncStartedEvent`, `*pierre.GitSyncSucceededEvent`, `*pierre.GitSyncFailedEvent`, and `*webhooks.UnknownEvent`.
  A rejection is a `*pierre.WebhookVerificationError` with `Reason`.
- A bad call, such as an empty secret, returns a plain `*pierre.PierreError`.
  The old `ValidateWebhook` returned `Valid: false` for it.
  To keep the old response, answer every error as a rejection:

```go
func handleWebhook(w http.ResponseWriter, r *http.Request, secret string, onPush func(repo string)) {
	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, "cannot read body", http.StatusBadRequest)
		return
	}
	event, err := webhooks.VerifyWebhook(webhooks.VerifyWebhookParams{Payload: body, Header: r.Header, Secret: secret})
	if err != nil {
		http.Error(w, err.Error(), http.StatusUnauthorized)
		return
	}
	if push, ok := event.(*pierre.GitPushEvent); ok {
		repo, ok := push.Repository.RepoName.Get()
		if !ok {
			repo = push.Repository.URL
		}
		onPush(repo)
	}
	w.WriteHeader(http.StatusNoContent)
}
```

## Files

- `Repos.Files.Get` and `Repos.Archive.Get` return an `io.ReadCloser`.
  Close it.
  Read the status and the headers with `option.WithResponseInto`.
- `Repos.Files.Metadata.Get` returns `*pierre.HeadersResult` with `Status` and `Headers`.
  Each header is a string, so parse `Headers.ContentLength` with `strconv.ParseInt`.

## Test mocks

A test that mocks the old client or a `Repo` changes with the code.
The project can have its own interface for the client ([sdk.md](sdk.md), "Your own type for the client").
An interface cannot hold the namespace fields of `*pierre.Client`, such as `Repos.Commits`.
So give each namespace that the code calls its own interface, with the methods that the code calls in the signatures of the SDK.
The namespace value, such as `client.Repos.Commits`, implements it.

```go
// commitsAPI is the part of client.Repos.Commits that the store calls.
type commitsAPI interface {
	Create(ctx context.Context, params pierre.ReposCommitsCreateParams, opts ...option.RequestOption) (*pierre.CommitWriteResponse, error)
	List(ctx context.Context, params pierre.ReposCommitsListParams, opts ...option.RequestOption) (*pierre.Page[pierre.Commit, pierre.ReposCommitsListResponse], error)
}

// Store keeps records in a repository.
type Store struct {
	commits commitsAPI
}

// NewStore returns a store that calls the client.
func NewStore(client *pierre.Client) *Store {
	return &Store{commits: client.Repos.Commits}
}
```

- Where the old interface held methods of several namespaces, the struct holds one interface for each namespace.
- Keep the signature of each exported constructor.
  A test sets the struct fields.
- **Report:** each exported interface that changes, because each type that implements it changes.

A fake:

```go
type fakeCommits struct {
	written []string
}

func (f *fakeCommits) Create(ctx context.Context, params pierre.ReposCommitsCreateParams, opts ...option.RequestOption) (*pierre.CommitWriteResponse, error) {
	for _, file := range params.Files {
		f.written = append(f.written, file.Path)
	}
	return nil, &pierre.RefUpdateError{
		APIError: &pierre.APIError{PierreError: &pierre.PierreError{Message: "409 conflict"}, Status: 409},
		Result:   pierre.RefUpdateResult{Status: pierre.RefUpdateResultStatusConflict},
	}
}

func (f *fakeCommits) List(ctx context.Context, params pierre.ReposCommitsListParams, opts ...option.RequestOption) (*pierre.Page[pierre.Commit, pierre.ReposCommitsListResponse], error) {
	return &pierre.Page[pierre.Commit, pierre.ReposCommitsListResponse]{}, nil
}
```

- A test builds the store with the fake in the field: `&Store{commits: &fakeCommits{}}`.
- A mock of the old commit builder becomes a mock of `Create`.
  Read `params.Files`, or `params.FileSeq` when the code passes a sequence.
- Make each error as the SDK does: the type for the status, with `Status` and `Code` on the embedded `*pierre.APIError`, for example `&pierre.NotFoundError{APIError: &pierre.APIError{PierreError: &pierre.PierreError{Message: "404 not_found"}, Status: 404, Code: "not_found"}}`.
  Set `PierreError`, because `Error()` reads its `Message`.
- A `*pierre.Page` with `HasMore` false never gets a next page, so a fake can return a `Page` with only `Data` and `Body`.
