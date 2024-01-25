# is-relevant-action

## What
A GitHub action that takes a list or lists of files and decides if any changes within
a PR's changes are relevant.

## Why
Whilst github does provide a way of only running actions if certain files change, 
out of the box it does not provide enough granularity to be as efficient as you might
want to be when running

## About
This action aims to be as concise, simple ad fast as possible and as such leverages a github
action's ability to run inline python.

## Usage
There are two ways of using this action, one which acts globally and also a further config that
can be passed which will apply further filtering to a list of keys. These can be used in 
conjunction with each other or alone.

### Global
Provide a plain old raw list of files to be included and files to be excluded.

```
name: Automated tests
on: [pull_request, workflow_dispatch]
jobs:
  includes-excludes-is-relevant-yes:
    runs-on: ubuntu-latest
    outputs:
      relevant: "${{ steps.is-relevant.outputs.relevant }}"
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Check relevance
        id: is-relevant
        uses: ./
        with:
          filenames: "/aaa/bbb/ccc,/aaa/xxx.foo,/zzz/xxx.foo"
          includes: |
            aaa/**
            **/*.foo
          excludes: |
            /zzz/**
      - name: Validate is-relevant output
        uses: ./.github/actions/validate-output
        with:
          relevant-expected: "\"yes\""
          relevant-actual: ${{ steps.is-relevant.outputs.relevant }}
          relevant-files-expected: "\"/aaa/bbb/ccc,/aaa/xxx.foo\""
          relevant-files-actual: ${{ steps.is-relevant.outputs.relevant-files }}
```

### Multiple
A list of keys can be passed (see the config arg passed below).
This may needed be because different jobs need to be run based upon different
sets for file changes. 

```
name: Automated tests
on: [pull_request, workflow_dispatch]
jobs:
  multi-is-relevant:
    runs-on: ubuntu-latest
    outputs:
      relevant: "${{ steps.is-relevant.outputs.relevant }}"
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Check relevance
        id: is-relevant
        uses: ./
        with:
          filenames: "/aaa/bbb/ccc,/aaa/xxx.foo,/zzz/xxx.foo"
          config: |
            {
              "A": { "includes": "/aaa/**" },
              "B": { "includes": "/bbb/**" },
              "C": { "includes": "/**/bbb/**"},
              "D": { "includes": "/zzzz/**" },
              "E": { "includes": "/zzzz/**,**/*.foo" }
            }
      - name: Validate is-relevant output
        uses: ./.github/actions/validate-output
        with:
          relevant-expected: '{"A": "yes", "B": "no", "C": "yes", "D": "no", "E": "yes"}'
          relevant-actual: ${{ steps.is-relevant.outputs.relevant }}
          relevant-files-expected: '{"A": "/aaa/xxx.foo,/aaa/bbb/ccc", "B": "", "C": "/aaa/bbb/ccc", "D": "", "E": "/aaa/xxx.foo,/zzz/xxx.foo"}'
          relevant-files-actual: ${{ steps.is-relevant.outputs.relevant-files }}
```

### Example usage
```

name: Automated tests
on:
  pull_request
jobs:
  list-changed-files:
    uses: ./.github/workflows/pr-get-changed-files.yml
    with:
      pr-number: ${{ github.event.number }}
    secrets: inherit

  is-relevant:
    runs-on: ubuntu-latest
    needs: list-changed-files
    outputs:
      relevant: ${{ steps.is-relevant.outputs.relevant }}
    steps:
      - name: Check relevance
        id: is-relevant
        uses: tubular-algorithms/is-relevant-action@v4
        with:
          filenames: ${{ needs.list-changed-files.outputs.filenames }}
          config: |
            {
              "core": { "excludes": "cypress/e2e/smoke/**,cypress/e2e/regression/**,**/*.test.js,storyshots/**"},
              "regression": {
                "includes": ".github/**,cypress/**",
                "excludes": "cypress/e2e/core/**,cypress/e2e/smoke/**"
              },
              "smoke": {
                "includes": ".github/**,cypress/**",
                "excludes": "cypress/e2e/regression/**,cypress/e2e/smoke/**"
              }
            }
      - name: Output results
        run: |
          echo '${{ steps.is-relevant.outputs.relevant }}' | \
            jq -r 'to_entries | map("\(.key) is relevant: \(.value)") | join("\n")'

  core-tests:
    if: fromJson(needs.is-relevant.outputs.relevant).core == 'yes'
    needs: [is-relevant]
    uses: ./.github/workflows/core.yml
    secrets: inherit

  regression-tests:
    if: fromJson(needs.is-relevant.outputs.relevant).regression == 'yes'
    needs: [is-relevant]
    uses: ./.github/workflows/regression.yml
    secrets: inherit

  smoke-tests:
    if: fromJson(needs.is-relevant.outputs.relevant).smoke == 'yes'
    needs: [is-relevant]
    uses: ./.github/workflows/smoke.yml
    secrets: inherit

  pr-conclusion:
    runs-on: ubuntu-latest
    needs:
      [
        core-tests,
        regression-tests,
        smoke-tests,
      ]
    if: always()
    steps:
      - name: Check for failure
        if: |
          contains(needs.*.result, 'failure')
        run: |
          echo 'Some checks have failed, exiting'
          exit 1
      - name: Output build pass message
        run: echo 'Well done your build has passed with flying colours (British spelling mandatory)!'


```

### A note on pattern matching
Glob pattern matching is supported as per the [Python glob specification](https://python.readthedocs.io/fr/latest/library/glob.html)


