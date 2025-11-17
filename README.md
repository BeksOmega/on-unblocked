# Find Unblocked Issues Action

This GitHub Action detects issues that were unblocked by the closing of another issue.

## Inputs

| Name    | Description                                                  | Required |
|---------|--------------------------------------------------------------|----------|
| `token` | `GITHUB_TOKEN` or a PAT with `issues:read`. | `false`  |

## Outputs

| Name         | Description                                                      |
|--------------|------------------------------------------------------------------|
| `issues`     | A JSON array of issue numbers that are now unblocked.            |
| `has_issues` | A boolean string (`"true"` or `"false"`) if any were found.      |

## Usage

**Note:** When using the default `GITHUB_TOKEN`, you must grant the appropriate permissions to the job that runs this action. See the example below.

```yaml
name: Handle Unblocked Issues

on:
  issues:
    types: [closed]

jobs:
  # JOB 1: Find the issues
  detect-unblocked:
    runs-on: ubuntu-latest
    permissions:
      issues: read
    outputs:
      # Pass the JSON array to the next job
      matrix: ${{ steps.finder.outputs.issues }}
      found: ${{ steps.finder.outputs.has_issues }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Find Unblocked Issues
        id: finder
        uses: BeksOmega/on-unblocked@v1

  # JOB 2: Run generic actions for EACH unblocked issue
  process-unblocked-issues:
    needs: detect-unblocked
    if: needs.detect-unblocked.outputs.found == 'true'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    strategy:
      matrix:
        # Parse the JSON array from the previous job
        issue-number: ${{ fromJson(needs.detect-unblocked.outputs.matrix) }}
    steps:
      - name: Log Information
        run: echo "Processing Unblocked Issue $ISSUE_NUMBER"
        env:
          ISSUE_NUMBER: ${{ matrix.issue-number }}

      - name: Label as Ready
        uses: actions/github-script@v7
        with:
          script: |
             await github.rest.issues.addLabels({
               owner: context.repo.owner,
               repo: context.repo.repo,
               issue_number: ${{ matrix.issue-number }},
               labels: ['ready-for-dev']
             });
```
