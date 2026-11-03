---
name: pierre
description: Use when you integrate with Pierre through its HTTP API, TypeScript, Python, or Go SDKs, or the Pierre CLI. Covers Code Storage repositories, Git, branches, and commits.
---

# Pierre

Read the current docs before you write an integration or run a command.
The docs own routes, authentication, scopes, request fields, and examples.

## Find the current interface

1. Check the project's package manifest and lockfile for the SDK package and version.
2. For CLI work, read `pierre --version` and the relevant command's `--help`.
3. If the docs MCP server is connected, discover its tools and search for the operation or guide.
4. Otherwise, fetch the docs index.
5. Select the relevant links from the index, then fetch those Markdown pages.
6. Read the authentication page and the operation's required scopes before a request.

Docs MCP server: `https://code.storage/docs/mcp`.
Docs index: `https://code.storage/docs/llms.txt`.
Pages accept a `.md` suffix, for example `https://code.storage/docs/repos/webhooks.md`.
Use Markdown pages instead of HTML pages or a full-site download.
The docs MCP server supplies reference material; it does not run Pierre operations.

For an SDK call, read that SDK's reference as well as the HTTP operation.
If a capability is absent from the docs or installed client, report that gap before proposing a call.

## Old Code Storage SDKs

The old SDKs are `@pierre/storage`, `pierre-storage`, and the Go module `code-storage-go`.
The rules below do not apply to them.
When a project uses one, suggest the `pierre-migrate` skill to move the code to the current SDK.
To keep the old SDK, read its README and its source to find each call.

## Map catalog operations

These rules apply to the current Pierre SDKs in TypeScript, Python, and Go, and to the `pierre` CLI.
They do not establish that a package, method, or command is released.
Check the installed version first.

- A catalog ID names a resource and an operation, for example `repos.branches.create`.
- Collections use plural nouns. Child resources nest under their parent.
- Singletons use singular nouns, for example `meta`, `repos.upstream`, and `repos.diff`.
- Standard verbs are `create`, `get`, `list`, `update`, and `delete`.
- Read a named action's docs when those verbs do not describe its effect.
- For HTTP, look up the method, path, and parameter locations in the operation's reference page.
- Do not derive an HTTP path or method from the catalog ID. Singular and plural paths can share a namespace.

### SDKs

- The method path is the catalog ID on the client.
  TypeScript and Python call it with one params object or keyword arguments.
  Go calls the same path in PascalCase, with a `context.Context` and a params struct.
- TypeScript and Python request and response fields keep the HTTP wire names, such as `repo_name`.
- Go uses exported PascalCase fields with JSON tags that match the wire names. Read the generated parameter types.
- Client configuration follows each language's convention; it has no HTTP wire names.
- Date fields remain RFC 3339 strings.
- An SDK parameter can carry authentication context, such as the repository of the token, instead of an HTTP body field.
- A local helper, such as the token or the Git remote URL helper, makes no HTTP call. Its docs give its name and import in each language.
- A method name with a suffix, such as `AndWait`, makes several HTTP calls. Read its docs.
- Read a helper's docs instead of inventing a route for it.

### CLI

- A command joins the resource path with `:` and puts the method after a space, with kebab-case segments: `repos.gitCredentials.create` is `pierre repos:git-credentials create`.
- Flags are the wire names, with hyphens in place of underscores.
- A parent resource is a flag, such as `--repo-name` (short form `-R`). Inside a clone with a Code Storage remote, `--repo-name` defaults to that repository.
- A command has at most one positional argument: the resource it acts on. Prefer the full flag form from its docs.
- `--json` prints the HTTP response body.
- `pierre api <catalog-id>` calls any operation. Pass fields with `-F key=value`.
- The exit code comes from the error: 2 for a usage error, 3 for an authentication or permission error, 4 for a resource that does not exist, and 1 for any other error.
- The CLI has no Git commands. After `pierre auth setup-git`, use plain Git.
- When the installed CLI's `--help` differs from these rules, follow `--help`.
