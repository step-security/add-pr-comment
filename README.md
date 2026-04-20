[![StepSecurity Maintained Action](https://raw.githubusercontent.com/step-security/maintained-actions-assets/main/assets/maintained-action-banner.png)](https://docs.stepsecurity.io/actions/stepsecurity-maintained-actions)

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
- Supports [file attachments](#file-attachments) via GitHub Artifacts.
- Automatic [message truncation](#message-truncation) for oversized messages (e.g., large Terraform plans).

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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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
| delete-on-status         | with     | If specified and a comment exists and the status is matching the value of this option, the comment will be deleted                                                          | no       |                                    |
| refresh-message-position | with     | Should the sticky message be the last one in the PR's feed.                                                                                                                 | no       | false                              |
| allow-repeats            | with     | Boolean flag to allow identical messages to be posted each time this action is run.                                                                                         | no       | false                              |
| proxy-url                | with     | String for your proxy service URL if you'd like this to work with fork-based PRs.                                                                                           | no       |                                    |
| issue                    | with     | Optional issue number override.                                                                                                                                             | no       |                                    |
| update-only              | with     | Only update the comment if it already exists.                                                                                                                               | no       | false                              |
| GITHUB_TOKEN             | env      | Valid GitHub token, can alternatively be defined in the env.                                                                                                                | no       |                                    |
| preformatted             | with     | Treat message text as pre-formatted and place it in a codeblock                                                                                                             | no       |                                    |
| find                     | with     | Patterns to find in an existing message and replace with either `replace` text or a resolved `message`. See [Find-and-Replace](#find-and-replace) for more detail.          | no       |                                    |
| replace                  | with     | Strings to replace a found pattern with. Each new line is a new replacement, or if you only have one pattern, you can replace with a multiline string.                      | no       |                                    |
| attach-path              | with     | A file path or glob pattern for files to upload as artifacts and link in the comment. See [File Attachments](#file-attachments).                                            | no       |                                    |
| attach-name              | with     | Name for the uploaded artifact.                                                                                                                                             | no       | pr-comment-attachments             |
| attach-text              | with     | Markdown content for the attachment section. Always separated from the comment by a horizontal rule. Supports `%ARTIFACT_URL%` and `%ATTACH_NAME%` template variables.      | no       | (see [File Attachments](#file-attachments)) |
| truncate                 | with     | Truncation mode when the message exceeds the safe comment length. See [Message Truncation](#message-truncation).  

## Outputs

| Output            | Description                                                       |
| ----------------- | ----------------------------------------------------------------- |
| `comment-created` | `"true"` if a new comment was created, `"false"` otherwise.       |
| `comment-updated` | `"true"` if an existing comment was updated, `"false"` otherwise. |
| `comment-id`      | The numeric ID of the created or updated comment.                 |
| `artifact-url`    | If files were attached, the URL to download the artifact.         |
| `truncated`       | `"true"` if the message was truncated, `"false"` otherwise.      |
| `truncated-artifact-url` | If truncated in artifact mode, the URL to download the full message. |

### Using outputs in subsequent steps

```yaml
- uses: step-security/add-pr-comment@v3
  id: comment
  with:
    message: 'Hello world'

- name: Check outputs
  run: |
    echo "Comment created: ${{ steps.comment.outputs.comment-created }}"
    echo "Comment updated: ${{ steps.comment.outputs.comment-updated }}"
    echo "Comment ID: ${{ steps.comment.outputs.comment-id }}"
```

> **Tip:** By default, comments are "upsert" — a comment is created on the first run and updated on subsequent runs when matched by `message-id`. If you want this create-or-update behavior, you do not need to set `update-only`. Setting `update-only: true` skips comment creation entirely and only updates an existing comment. Use it when you specifically want no comment to appear unless one was already posted by a previous step or run.

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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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
      - uses: step-security/add-pr-comment@v3
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

### File Attachments

You can attach files to your PR comments by uploading them as GitHub Artifacts and embedding download links in the comment body. Files matching the `attach-path` glob are uploaded as a single artifact, and a markdown section with the download link is appended to your comment, separated by a horizontal rule.

> **Note:** Artifact download URLs require GitHub authentication and expire based on your repository's retention settings (default 90 days). Images will not render inline — they appear as download links. This is a GitHub platform limitation.

**Simple — attach a file with defaults**

```yaml
on:
  pull_request:
jobs:
  report:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v6
      - run: echo "Build output here" > report.txt
      - uses: step-security/add-pr-comment@v3
        with:
          message: |
            Build complete! See attached report.
          attach-path: report.txt
```

**Advanced — glob pattern, custom name, and custom text template**

```yaml
on:
  pull_request:
jobs:
  report:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v6
      - run: |
          mkdir -p coverage
          echo "line coverage: 85%" > coverage/summary.txt
          echo "<html>...</html>" > coverage/report.html
      - uses: step-security/add-pr-comment@v3
        with:
          message: |
            ## Coverage Report
            Tests passed with 85% line coverage.
          attach-path: coverage/*
          attach-name: coverage-report
          attach-text: '📎 [Download %ATTACH_NAME%](%ARTIFACT_URL%)'
```

The `attach-text` input supports two template variables:

| Variable         | Replaced with                      |
| ---------------- | ---------------------------------- |
| `%ARTIFACT_URL%` | The artifact download URL          |
| `%ATTACH_NAME%`  | The value of the `attach-name` input |

### Message Truncation

GitHub's API limits comment bodies to 65,536 characters. Messages that exceed this limit (common with large Terraform plans, verbose test output, etc.) would previously cause the action to fail with an "Argument list too long" or API error.

This action automatically truncates oversized messages to stay within a safe limit (61,440 characters, which includes a 4,096 character buffer). The `truncate` input controls what happens with the full message:

| Mode | Behavior |
| ---- | -------- |
| `artifact` (default) | The full, untruncated message is uploaded as a downloadable GitHub Artifact. The comment is truncated and a download link is appended. |
| `simple` | The comment is truncated and a notice is appended. No artifact is uploaded. |

If artifact upload fails (e.g., permissions, network issues), the action automatically falls back to simple truncation.

**Example — default artifact mode**

```yaml
on:
  pull_request:
jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v6
      - run: terraform plan -no-color > plan.txt
      - uses: step-security/add-pr-comment@v3
        with:
          message-path: plan.txt
```

If the plan output exceeds the safe limit, the comment will be truncated and end with:

> **This message was truncated.** [Download full message](https://github.com/...)

**Example — simple mode (no artifact)**

```yaml
- uses: step-security/add-pr-comment@v3
  with:
    message-path: plan.txt
    truncate: simple
```

The comment will be truncated and end with:

> **This message was truncated.**

**Using the truncation outputs**

```yaml
- uses: step-security/add-pr-comment@v3
  id: comment
  with:
    message-path: plan.txt
- name: Check if truncated
  if: steps.comment.outputs.truncated == 'true'
  run: |
    echo "Message was truncated"
    echo "Full message: ${{ steps.comment.outputs.truncated-artifact-url }}"
```

> **Tip:** For very large outputs like Terraform plans, prefer using `message-path` over the `message` input. The `message` input is passed via environment variables, which have OS-level size limits that can cause failures before the action even runs. File-based input via `message-path` avoids this entirely.


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

      - uses: step-security/add-pr-comment@v3
        with:
          issue: ${{ steps.pr.outputs.issue }}
          message: |
            **Howdie!**
```

### Delete on status

This option can be used if comment needs to be removed if a status is reached.

**Example**

> Here, a comment will be added on failure, but on a subsequent run,
> if the job reaches success status, the comment will be deleted.

```yaml
on:
  pull_request:
jobs:
  pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: step-security/add-pr-comment@v3
        if: always()
        with:
          message-failure: There was a failure
          delete-on-status: success
```