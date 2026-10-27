---
name: pierre-migrate
description: Use when code calls the old Pierre Code Storage SDKs (`@pierre/storage`, `pierre-storage`, or the Go module `code-storage-go`) or deprecated Pierre HTTP routes and fields such as `/api/v1/repos/*`, and you move that code to `@pierre/sdk` or the current HTTP API.
metadata:
  from-typescript: "@pierre/storage 1.17.1"
  to-typescript: "@pierre/sdk 0.1.0-canary.3"
  from-python: "pierre-storage 1.17.1"
  to-python: "pierre-sdk 0.1.0.dev3, not published"
  from-go: "github.com/pierrecomputer/sdk/packages/code-storage-go v1.17.1"
  to-go: "github.com/pierrecomputer/sdk/go v0.1.0-canary.3, not published"
---

# Move to `@pierre/sdk` and the current HTTP API

This skill covers one move: from the old Code Storage SDKs and the deprecated HTTP routes and fields to `@pierre/sdk` and the current routes.
For the fields and behavior of a current operation, use the `pierre` skill or the docs index, `https://code.storage/docs/llms.txt`.
The `metadata` of this file names the old and new SDK versions that the facts and examples match.

## 1. Find what the project uses

Run these searches from the project root.
Read only the references for what you find.
Every path also uses [references/names.md](references/names.md).

| Search | The project uses | Read |
| --- | --- | --- |
| `rg -l '@pierre/storage'` | the TypeScript or JavaScript SDK | [methods.md](references/methods.md), [sdk.md](references/sdk.md), [typescript.md](references/typescript.md) |
| `rg -l 'pierre[-_]storage'` | the Python SDK | [methods.md](references/methods.md), [sdk.md](references/sdk.md), [python.md](references/python.md) |
| `rg -l 'code-storage-go'` | the Go SDK | [methods.md](references/methods.md), [sdk.md](references/sdk.md), [go.md](references/go.md) |
| `rg -n -e '/api/v1/repo' -e 'pull-upstream' -e 'branches/diff' -e '/base\b'` | HTTP requests to deprecated routes | [http.md](references/http.md) |

A project can use more than one of these.
Code that sends HTTP requests to current routes can still send deprecated fields.
Compare its parameters and fields with [names.md](references/names.md).

List each call site by file and line, with its old call and its new call.
Step 2 says when to show this list to the user.

## 2. Ask before you change behavior

The references mark two kinds of behavior change:

- **Ask:** no new behavior is equal to the old behavior.
  Collect these questions while you read the call sites.
  Then ask the user once, in one numbered list.
  Give the options for each question.
  Recommend the default that the reference gives, unless the code gives a reason for another option.
  In that case, name the reason, for example a comment or a call that follows.
  If the user does not answer, apply the default that the reference gives.
- **Report:** the reference gives the change that keeps the old behavior.
  Apply it without a question.

A change with no label is a rename or keeps the behavior exactly.
It needs no question and no report line.
Ask only about cases that the code has.

Put the call-site list of step 1 in the same message as the questions, as a table: file and line, old call, new call.
If there are no questions, show the table before your first edit.

## 3. Change the code

- For an SDK move, install the new SDK beside the old one.
  The language file names the package and says how to install it.
  Use the package manager of the project's lockfile.
  Without a lockfile, change only the manifest.
  Use the version that the user names.
  If the user names none, use the newest published version, and report it.
  Remove the old SDK in step 4.
- Change the imports and the client first.
  Then use the type checker or the compiler to list each remaining use, and change each use with the references.
- Name the new client as the docs do: `pierre` in TypeScript and Python, `client` in Go.
- Change only what the move needs.
  Do not refactor other code.
- Add no comment that describes the old SDK or the move, and no question or TODO.
  Put them in the report.
  Keep each existing comment, and change it only where it becomes false.
- Do not print, copy, or open a private key or a token.
- If the new API has no equal for an old call, keep that call and the old SDK, and report it.
- If no reference covers a case, read the old SDK source and the new SDK types, or the operation's docs page.
  The old source is in the installed package and in `https://github.com/pierrecomputer/sdk`.
  The OpenAPI document, `https://code.storage/openapi.json`, marks each deprecated route and field with its replacement.

## 4. Verify

1. Run the project's own checks: the type check or the build, the linter, and the tests.
   Tests that call the API need the credentials that the project reads, such as the `PIERRE_*` variables.
   If they are not set, ask the user to run those tests.
2. Run the searches of step 1 again.
   They must find only the calls that you kept and reported.
3. Remove the old SDK package when no use of it remains.
   The manifest and the lockfile must not name it.
4. Run the checks again.

## 5. Report

Give the user:

- the files that changed;
- each **Ask** item, with the user's answer or the default that you applied;
- each **Report** item, with its file and line;
- each call that you kept on the old SDK or on a deprecated route, and the reason;
- each check that you ran, and its result.
