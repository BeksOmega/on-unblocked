# Find Unblocked Issues Action

This GitHub Action detects issues that were unblocked by the closing of another issue.

## Inputs

| Name    | Description                                                  | Required |
|---------|--------------------------------------------------------------|----------|
| `token` | `GITHUB_TOKEN` or a PAT with `issues:read` and `orgs:read`. | `true`   |

## Outputs

| Name         | Description                                                      |
|--------------|------------------------------------------------------------------|
| `issues`     | A JSON array of issue numbers that are now unblocked.            |
| `has_issues` | A boolean string (`"true"` or `"false"`) if any were found.      |

## Usage

```yaml
name: Handle Unblocked Issues

on:
  issues:
    types: [closed]

permissions:
  # For the labeling
  issues: write

jobs:
  # JOB 1: Find the issues
  detect-unblocked:
    runs-on: ubuntu-latest
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
        with:
          token: ${{ secrets.ORG_READ_TOKEN || secrets.GITHUB_TOKEN }}

  # JOB 2: Run generic actions for EACH unblocked issue
  process-unblocked-issues:
    needs: detect-unblocked
    if: needs.detect-unblocked.outputs.found == 'true'
    runs-on: ubuntu-latest
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
