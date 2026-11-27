# runlab verdict

**Did this change help?** A pull-request check that answers with numbers: the difference
between the change and its baseline over seeds, its 95 % confidence interval and the verdict.
It fails when the change misses what you expected of it, and `refused` (a result that fails
an integrity check) always fails.

An example check:

| | A vs B | metric | A − B [95% CI] | p | seeds (A, B) | verdict | why |
|---|---|---|---|---|---|---|---|
| ✅ | `pr-41-3f9c2ab` vs `base-41-3f9c2ab` | val_acc | +0.0123 [+0.0041, +0.0205] | 0.006 | 5, 5 | a_better | expected better: met |

The logic is `runlab verdict` in [runlab-client](https://github.com/omer-nachman/runlab-python)
(MIT, on PyPI); this action installs it and runs it.

## Use it

1. Commit the spec to your default branch as `.runlab/verdict.yml`:

   ```yaml
   paths: ["train/**", "configs/**"]   # other changes need no verdict
   project: my-model
   checks:
     - a: "pr-{pr}-{sha7}"             # this pull request's seeds
       b: "base-{pr}-{sha7}"           # the base branch's seeds, from the same job
       metric: val_acc
       expect: better                  # better | not_worse (with margin) | different
   ```

2. Sign in to RunLab with GitHub once, as the repository's owner:

   ```bash
   pip install runlab-client && runlab login
   ```

3. Give the workflow these permissions (a job's own `permissions:` replaces the workflow's,
   so list all four there):

   ```yaml
   permissions:
     contents: read
     checks: write          # the check run
     pull-requests: write   # the pull-request comment
     id-token: write        # keyless sign-in to RunLab
   ```

   and add the step after the one that trains both arms' seeds (each seed prints `RESULT:`
   lines, see below):

   ```yaml
         - uses: omer-nachman/runlab-action@v1
           with:
             results: |
               out/pr/*.txt --arm pr-{pr}-{sha7}
               out/base/*.txt --arm base-{pr}-{sha7}
   ```

No key and no secret: the job signs in with the short-lived token GitHub gives it, which names
the repository and is minted for RunLab only; RunLab accepts it as the RunLab user bound to the
repository owner's GitHub account, for this action's calls only. An organization's repository
passes a key instead: `runlab-key: ${{ secrets.RUNLAB_KEY }}`.

A whole workflow, both arms trained in the same job (same runner, same budget per seed):

```yaml
name: verdict
on: pull_request
permissions:
  contents: read
  checks: write          # the check run
  pull-requests: write   # the pull-request comment
  id-token: write        # keyless sign-in to RunLab
jobs:
  verdict:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - name: Seeds of both arms
        env:
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
        run: |
          git worktree add ../base "$BASE_SHA"
          mkdir -p out/pr out/base
          for seed in 1 2 3 4 5; do
            python train.py --seed "$seed" > "out/pr/$seed.txt"
            (cd ../base && python train.py --seed "$seed") > "out/base/$seed.txt"
          done
      - uses: omer-nachman/runlab-action@v1
        with:
          results: |
            out/pr/*.txt --arm pr-{pr}-{sha7}
            out/base/*.txt --arm base-{pr}-{sha7}
```

Each seed prints one line per metric; other lines are skipped:

```text
RESULT: <arm> <metric>=<number> seed=<n>
```

From Python, `print(runlab_client.result_line("any", "val_acc", acc, seed=seed))` writes one;
`--arm` replaces the arm, so the training script need not know which arm it is.

## What keeps the check honest

| Rule | Why |
|---|---|
| The spec is read from the pull request's **base** commit | a pull request cannot loosen the test it is judged by; one that adds or changes the spec is judged by the old one, or gets a neutral check when there was none |
| `refused` and `insufficient` (fewer than 2 seeds in an arm) always fail | a verdict that cannot be trusted never passes |
| `{sha7}` in the arm names | each push gets its own pair of arms, compared once; a second look at a pair warns `peeking` |
| An unknown key in the spec is an error | a typo never turns a check off |
| `paths` | a pull request that changes none of them gets a neutral check and records nothing |

The check is only as trustworthy as the job that runs the seeds: it runs in the pull
request's own workflow. Pull requests from forks get no token or secrets in GitHub Actions, so
they get a notice and no verdict.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `runlab-key` | empty: keyless | a RunLab key, from a secret; needed for an organization's repository |
| `results` | none | files to record, one per line: `GLOB [--arm A] [--project P]`; `{pr}`, `{sha}`, `{sha7}`, `{branch}` are filled in |
| `spec` | `.runlab/verdict.yml` | the spec's path |
| `spec-from` | `base` | `head` reads the checked-out spec instead, to try a new one |
| `post` | `true` | `false` writes only the job summary |
| `fail` | `true` | `false`: a missed `expect` fails the check run but not the step |
| `url` | `https://runlab.tools` | your RunLab URL |
| `token` | `github.token` | for the Checks and Issues APIs |
| `python-version` | `3.12` | empty uses the runner's python |
| `version` | `0.4.0` | the runlab-client version; the ref in `uses:` pins it |

Outputs: `conclusion` (success, failure or neutral), `failed`, `checks`, `url` (the check
run's).

## Spec keys

| Key | Default | Meaning |
|---|---|---|
| `paths` | every path | globs: `*` within a folder, `**` across folders, `dir/` everything under it |
| `project` | none | the project of every row that names none |
| `a`, `b` | required | the change's arm and the baseline's |
| `metric` | required | the metric to test |
| `expect` | `better` | `better` passes on `a_better`; `not_worse` also on `equivalent` or `non_inferior` and needs `margin`; `different` on `a_better` or `b_better` |
| `margin` | none | the smallest difference that matters, in the metric's units |
| `better` | `auto` | `higher`, `lower`, or `auto` from the metric's name |
| `alpha` | 0.05 | the test's level; the interval is 1 − alpha |
| `missing` | `failure` | the row's result when an arm has no results: `failure` or `neutral` |

Full reference: "Pull-request check" in the
[runlab-client README](https://github.com/omer-nachman/runlab-python#pull-request-check-runlab-verdict).

## License

MIT
