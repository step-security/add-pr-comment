# add-pr-comment
A GitHub Action which adds a comment to a pull request's issue.

This actions also works on [issue](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#issues),
[issue_comment](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#issue_comment),
[deployment_status](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#deployment_status),
[push](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push)
and any other event where an issue can be found directly on the payload or via a commit sha.

## Features

- Modify issues for PRs merged to main.
- By default will post "sticky" comments. If on a subsequent run the message text changes the original comment will be updated.
- Multiple sticky comments allowed by setting unique `message-id`s.
- Optional message overrides based on job status.
- Multiple posts to the same conversation optionally allowable.
- Supports a proxy for fork-based PRs. [See below](#proxy-for-fork-based-prs).
- Supports creating a message from a file path.

## Usage

Note that write access needs to be granted for the pull-requests scope.

```yaml
on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        with:
          message: |
            **Hello**
            🌏
            !
```

You can even use it on PR Issues that are related to PRs that were merged into main, for example:

```yaml
on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        with:
          message: |
            **Hello MAIN**
```

## Configuration options

| Input                    | Location | Description                                                                                                                                                                 | Required | Default                            |
| ------------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---------------------------------- |
| message                  | with     | The message you'd like displayed, supports Markdown and all valid Unicode characters.                                                                                       | maybe    |                                    |
| message-path             | with     | Path to a message you'd like displayed. Will be read and displayed just like a normal message. Supports multi-line input and globs. Multiple messages will be concatenated. | maybe    |                                    |
| message-success          | with     | A message override, printed in case of success.                                                                                                                             | no       |                                    |
| message-failure          | with     | A message override, printed in case of failure.                                                                                                                             | no       |                                    |
| message-cancelled        | with     | A message override, printed in case of cancelled.                                                                                                                           | no       |                                    |
| message-skipped          | with     | A message override, printed in case of skipped.                                                                                                                             | no       |                                    |
| status                   | with     | Required if you want to use message status overrides.                                                                                                                       | no       | {{ job.status }}                   |
| repo-owner               | with     | Owner of the repo.                                                                                                                                                          | no       | {{ github.repository_owner }}      |
| repo-name                | with     | Name of the repo.                                                                                                                                                           | no       | {{ github.event.repository.name }} |
| repo-token               | with     | Valid GitHub token, either the temporary token GitHub provides or a personal access token.                                                                                  | no       | {{ github.token }}                 |
| message-id               | with     | Message id to use when searching existing comments. If found, updates the existing (sticky comment).                                                                        | no       |                                    |
| refresh-message-position | with     | Should the sticky message be the last one in the PR's feed.                                                                                                                 | no       | false                              |
| allow-repeats            | with     | Boolean flag to allow identical messages to be posted each time this action is run.                                                                                         | no       | false                              |
| proxy-url                | with     | String for your proxy service URL if you'd like this to work with fork-based PRs.                                                                                           | no       |                                    |
| issue                    | with     | Optional issue number override.                                                                                                                                             | no       |                                    |
| update-only              | with     | Only update the comment if it already exists.                                                                                                                               | no       | false                              |
| GITHUB_TOKEN             | env      | Valid GitHub token, can alternatively be defined in the env.                                                                                                                | no       |                                    |
| preformatted             | with     | Treat message text as pre-formatted and place it in a codeblock                                                                                                             | no       |                                    |
| find                     | with     | Patterns to find in an existing message and replace with either `replace` text or a resolved `message`. See [Find-and-Replace](#find-and-replace) for more detail.          | no       |                                    |
| replace                  | with     | Strings to replace a found pattern with. Each new line is a new replacement, or if you only have one pattern, you can replace with a multiline string.                      | no       |                                    |

## Advanced Uses

### Proxy for Fork-based PRs

GitHub limits `GITHUB_TOKEN` and other API access token permissions when creating a PR from a fork. This precludes adding comments when your PRs are coming from forks, which is the norm for open source projects. To work around this situation I've created a simple companion app you can deploy to Cloud Run or another host to proxy the create comment requests with a personal access token you provide.

See this issue: https://github.community/t/github-actions-are-severely-limited-on-prs/18179/4 for more details.

Check out the proxy service here: https://github.com/mshick/add-pr-comment-proxy

**Example**

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        with:
          message: |
            **Howdie!**
          proxy-url: https://add-pr-comment-proxy-94idvmwyie-uc.a.run.app
```

### Status Message Overrides

You can override your messages based on your job status. This can be helpful
if you don't anticipate having the data required to create a helpful message in
case of failure, but you still want a message to be sent to the PR comment.

**Example**

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        if: always()
        with:
          message: |
            **Howdie!**
          message-failure: |
            Uh oh!
```

### Multiple Message Files

Instead of directly setting the message you can also load a file with the text
of your message using `message-path`. `message-path` supports loading multiple
files and files on multiple lines, the contents of which will be concatenated.

**Example**

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        if: always()
        with:
          message-path: |
            message-part-*.txt
```

### Find-and-Replace

Patterns can be matched and replaced to update comments. This could be useful
for some situations, for instance, updating a checklist comment.

Find is a regular expression passed to the [RegExp() constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/RegExp). You can also
include modifiers to override the default `gi`.

**Example**

Original message:

```
[ ] Hello
[ ] World
```

Action:

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        if: always()
        with:
          find: |
            \n\\[ \\]
          replace: |
            [X]
```

Final message:

```
[X] Hello
[X] World
```

Multiple find and replaces can be used:

**Example**

Original message:

```
hello world!
```

Action:

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        if: always()
        with:
          find: |
            hello
            world
          replace: |
            goodnight
            moon
```

Final message:

```
goodnight moon!
```

It defaults to your resolved message (either from `message` or `message-path`) to
do a replacement:

**Example**

Original message:

```
hello

<< FILE_CONTENTS >>

world
```

Action:

```yaml
on:
  pull_request:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v2
        if: always()
        with:
          message-path: |
            message.txt
          find: |
            << FILE_CONTENTS >>
```

Final message:

```
hello

secret message from message.txt

world
```

### Bring your own issues

You can set an issue id explicitly. Helpful for cases where you want to post
to an issue but for some reason the event would not allow the id to be determined.

**Example**

> In this case `add-pr-comment` should have no problem finding the issue number
> on its own, but for demonstration purposes.

```yaml
on:
  deployment_status:

jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - id: pr
        run: |
          issue=$(gh pr list --search "${{ github.sha }}" --state open --json number --jq ".[0].number")
          echo "issue=$issue" >>$GITHUB_OUTPUT
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - uses: step-security/add-pr-comment@v2
        with:
          issue: ${{ steps.pr.outputs.issue }}
          message: |
            **Howdie!**
```
