# tag-generator

## What is it?

`tag-generator` is a reusable GitHub Action responsible for creating the final Git tag and publishing a GitHub Release after the deployment flow has completed successfully.

This action is designed to run at the end of a pipeline, after previous jobs such as release validation, quality assurance, snapshot retrieval, infrastructure deployment, and application deployment have finished successfully.

The `tag-generator` does not validate the Pull Request title format or extract the release version.  
Those responsibilities should be handled by a previous action, such as `release-checker`.

The main goal of this action is to ensure that Git tags and GitHub Releases are created only after the full release/deployment workflow succeeds.

---

## What does it do?

The `tag-generator` action performs the following steps:

1. Checks out the repository.
2. Configures Git using the GitHub Actions bot user.
3. Validates that required inputs were provided.
4. Checks whether the Git tag already exists.
5. Creates an annotated Git tag.
6. Pushes the tag to the remote repository.
7. Saves the release notes into a file.
8. Publishes a GitHub Release using the provided tag, title, and release notes.

---

## When should this action run?

This action should run at the end of the workflow.

Recommended flow:

```text
Pull Request merged into main
        │
        ▼
release-checker
        │
        ├── Validate required label
        ├── Validate Pull Request title
        └── Extract release version
        │
        ▼
Quality Assurance
        │
        ▼
Get Snapshot Tag
        │
        ▼
Deployment Jobs
        │
        ▼
tag-generator
        │
        ├── Create Git tag
        └── Publish GitHub Release
```


## How to use

The `tag-generator` action can be used as a local composite action inside repositories that follow the internal GitHub Actions structure:

```text
.github
├── actions
│   └── tag-generator
│       └── action.yml
│
└── workflows
    └── deploy-to-prod.yml
```

The actions folder contains the reusable action configuration, while the workflows folder contains the pipeline that calls the action.

- Action Configuration:

The tag-generator action must be placed inside:

 ```
        .github/actions/tag-generator/action.yml
```

-  Workflow Configuration:

The tag-generator action can be used in two different workflow scenarios:

1. When the workflow is triggered by a Pull Request event.

```yaml
name: Deploy to Prod Environment

on:
  pull_request:
    types:
      - closed

jobs:
  release-checker:
    name: Release Checker
    runs-on: ubuntu-latest

    if: github.event.pull_request.merged == true && github.event.pull_request.base.ref == 'master'

    outputs:
      version: ${{ steps.release-checker.outputs.version }}
      release-title: ${{ steps.release-checker.outputs.release-title }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Check release information
        id: release-checker
        uses: ./.github/actions/release-checker
        with:
          pr-title: ${{ github.event.pull_request.title }}
          pr-labels: ${{ toJson(github.event.pull_request.labels.*.name) }}

  tag-generator:
    name: Tag Generator
    runs-on: ubuntu-latest

    needs:
      - release-checker

    if: github.event.pull_request.merged == true && github.event.pull_request.base.ref == 'master'

    permissions:
      contents: write
      pull-requests: read

    steps:
      - name: Generate tag and release
        uses: ./.github/actions/tag-generator
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          version: ${{ needs.release-checker.outputs.version }}
          release-title: ${{ needs.release-checker.outputs.release-title }}
          release-notes: ${{ github.event.pull_request.body }}
```

2. When the workflow is triggered by a push event:

```yaml
name: Deploy to Prod Environment

on:
  push:
    branches:
      - master

jobs:
  get-pr-context:
    name: Get Pull Request Context
    runs-on: ubuntu-latest

    permissions:
      contents: read
      pull-requests: read

    outputs:
      pr-title: ${{ steps.pr.outputs.pr-title }}
      pr-body: ${{ steps.pr.outputs.pr-body }}
      pr-labels: ${{ steps.pr.outputs.pr-labels }}

    steps:
      - name: Get Pull Request information
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPOSITORY: ${{ github.repository }}
          COMMIT_SHA: ${{ github.sha }}
        run: |
          PR_DATA=$(gh api \
            -H "Accept: application/vnd.github+json" \
            "/repos/$REPOSITORY/commits/$COMMIT_SHA/pulls")

          PR_COUNT=$(echo "$PR_DATA" | jq length)

          if [[ "$PR_COUNT" -eq 0 ]]; then
            echo "ERROR: No Pull Request was found for commit $COMMIT_SHA."
            echo "This workflow must be triggered by a merge commit associated with a Pull Request."
            exit 1
          fi

          PR_TITLE=$(echo "$PR_DATA" | jq -r '.[0].title')
          PR_BODY=$(echo "$PR_DATA" | jq -r '.[0].body // ""')
          PR_LABELS=$(echo "$PR_DATA" | jq -c '.[0].labels | map(.name)')

          echo "Pull Request title: $PR_TITLE"
          echo "Pull Request labels: $PR_LABELS"

          echo "pr-title=$PR_TITLE" >> "$GITHUB_OUTPUT"
          echo "pr-labels=$PR_LABELS" >> "$GITHUB_OUTPUT"

          {
            echo "pr-body<<EOF"
            echo "$PR_BODY"
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

  release-checker:
    name: Release Checker
    runs-on: ubuntu-latest

    needs:
      - get-pr-context

    outputs:
      version: ${{ steps.release-checker.outputs.version }}
      release-title: ${{ steps.release-checker.outputs.release-title }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Check release information
        id: release-checker
        uses: ./.github/actions/release-checker
        with:
          pr-title: ${{ needs.get-pr-context.outputs.pr-title }}
          pr-labels: ${{ needs.get-pr-context.outputs.pr-labels }}

  tag-generator:
    name: Tag Generator
    runs-on: ubuntu-latest

    needs:
      - get-pr-context
      - release-checker

    permissions:
      contents: write
      pull-requests: read

    steps:
      - name: Generate tag and release
        uses: ./.github/actions/tag-generator
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          version: ${{ needs.release-checker.outputs.version }}
          release-title: ${{ needs.release-checker.outputs.release-title }}
          release-notes: ${{ needs.get-pr-context.outputs.pr-body }}

```