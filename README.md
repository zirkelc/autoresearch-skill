<div align='center'>
  <h1>Autoresearch</h1>
  <p align="center">An agent skill that runs a performance optimization campaign on a code repository.</p>
</div>

## What it does

The skill makes an agent improve the performance of an existing library through many small experiments. Each experiment is one change. The agent commits it, measures it against the previous commit, and keeps it only if the result is above a threshold that was measured on the same machine before the first experiment. Discarded experiments are recorded in a log with their measured result and commit hash.

A campaign produces:

- a branch with the kept commits,
- a plan with the calibration results and the notes for each experiment,
- a tab-separated log of all experiments, kept and discarded,
- pull requests for the kept changes, each created only after the user confirms it.

## Origin

The skill is based on [karpathy/autoresearch](https://github.com/karpathy/autoresearch). In that project, an agent edits a training script, trains a model for five minutes, reads the validation loss, keeps or discards the change, and repeats. The human edits the instructions in `program.md`, not the code.

This skill uses the same loop for a different task: making an existing library faster without changing its behaviour. It does not share code with karpathy/autoresearch.

The different task requires parts that the original does not need:

- **Measurement.** The original uses a fixed metric (validation loss after a fixed time). In a library, the effects are often 1% to 25% of the run time, and two runs of the same code can differ by more than that. The skill therefore compares two git revisions in one process, in alternation, and calibrates a noise threshold before the first experiment.
- **Behaviour checks.** A performance change must not change the output of the library. The skill records the current behaviour before the first experiment and checks it after every change.
- **Pull requests.** The kept changes go to a repository owned by other people. The skill includes rules for grouping, verifying and describing the changes for the maintainers.

## How it works

`SKILL.md` has ten steps:

0. **Scope.** Agree on the branch, the focus and the number of experiments.
1. **Explore.** Find how the repository builds, tests and benchmarks, which metric the maintainers accept, and which checks can run on this machine.
2. **Guard.** Record the current behaviour as hashes and commit them. The guard is not changed after this step.
3. **Harness.** Copy the runtime folder into the repository and write the benchmark cases. Check the harness with a deliberate slowdown and a deliberate wrong result.
4. **Calibrate.** Compare identical code with itself three times. The result is the noise floor. The threshold for keeping a change is about twice the noise floor, and at least 1%.
5. **Baseline and plan.** Profile the code, test how the run time grows with the input size, and write a list of candidates.
6. **Rules.** The rules that the loop follows.
7. **Loop.** Change, check, commit, measure. Measure again if the change clears the threshold. Keep the commit or reset it. Log the result.
8. **Stop.** After three experiments in a row that are not kept, try other parts of the code. Stop at the experiment budget.
9. **Package.** Group the kept commits into pull requests, verify each one on its own, and confirm each with the user before it is created.

### Three measurements

The skill measures a change in three ways. They give different results, and each one has a specific use:

| Measurement | What it shows | Used for |
|---|---|---|
| Full suite, both revisions in one process | Whether other cases changed | The summary numbers |
| One case, both revisions in one process | Whether the target case changed | The decision to keep or discard |
| One case, one revision per process | What a maintainer will measure | The numbers in a pull request |

The third measurement is reported together with a control run of identical code. If the control run varies as much as the measured effect, the pull request reports the second measurement and says so.

## Contents

```
autoresearch-skill/
├── SKILL.md                      the method, independent of the language
├── references/
│   ├── methodology.md            how the measurements work and how to report them
│   └── pr-packaging.md           how to turn the kept commits into pull requests
├── templates/
│   ├── plan.md                   campaign plan with calibration and experiment notes
│   ├── experiments.tsv           experiment log
│   ├── pr-body.md                pull request body
│   ├── gates.example.sh          runs the repository's checks in the right order
│   └── fetch-fixtures.sh         downloads external test inputs with checksums
└── runtimes/
    ├── README.md                 what a runtime must provide
    └── node-ts/                  harness for JavaScript and TypeScript on Node.js
        ├── harness.mts           shared code: config, git revisions, case loading
        ├── ab.mts                compares two git revisions in one process
        ├── solo.mts              measures one revision per process
        ├── guard.mts             records and checks the behaviour of the cases
        ├── differential.mts      compares the output of two revisions over generated inputs
        ├── scan.mts              measures each case at three input sizes
        ├── profile.mts           CPU profile per function, caller, line or directory
        ├── mem.mts               retained memory per instance
        ├── jitter.mts            checks that the machine is quiet enough to measure
        ├── quiet.sh              waits for a quiet machine, then runs a command
        ├── micro.example.mjs     compares two versions of one function
        ├── selftest.mts          tests the harness on a generated monorepo
        ├── cases.example.mts     example benchmark cases
        ├── fixtures.example.mts  example shared inputs
        ├── perf.config.example.json
        └── README.md
```

The `node-ts` folder is copied into the target repository. Other languages need a new runtime folder that follows `runtimes/README.md`.

## Install

```bash
npx skills add zirkelc/autoresearch-skill
```

## Use

```
/autoresearch
/autoresearch error paths, 10 experiments
```

You can also ask for it in plain words, for example "run a performance campaign on this repo". The skill asks for the scope and the budget before it changes anything.

A campaign takes several hours. Most of the time is measurement and, on a shared machine, waiting until the machine is quiet.

## Results

The skill has run on four repositories. After each campaign, the skill was updated with the problems found during that campaign.

### zod

18 experiments, 11 kept. 40% less time on the benchmark suite.

| Pull request | Description | Status |
|---|---|---|
| [#6316](https://github.com/colinhacks/zod/pull/6316) | Lazy `ZodError` construction, about 40% faster failing `safeParse` | Merged |
| [#6317](https://github.com/colinhacks/zod/pull/6317) | Reuse frozen default parse contexts, about 15% faster leaf parses | Closed |
| [#6318](https://github.com/colinhacks/zod/pull/6318) | Halve schema construction cost and keep instances in fast-properties mode | Merged |
| [#6319](https://github.com/colinhacks/zod/pull/6319) | Single-check fast path and in-place issue prefixing in the object JIT | Closed |

### linkedom

18 experiments, 8 kept. 2.02x faster on the benchmark suite.

| Pull request | Description | Status |
|---|---|---|
| [#335](https://github.com/WebReflection/linkedom/pull/335) | Create the event listeners map on first use | Open |
| [#336](https://github.com/WebReflection/linkedom/pull/336) | Cache the class token value and skip the token list while parsing | Open |
| [#337](https://github.com/WebReflection/linkedom/pull/337) | Walk and serialize the tree without intermediate arrays | Open |
| [#338](https://github.com/WebReflection/linkedom/pull/338) | Allocate the node end marker without symbol keys in the literal | Open |

### micromark

16 experiments, 7 kept.

| Pull request | Description | Status |
|---|---|---|
| [#234](https://github.com/micromark/micromark/pull/234) | Improve `subtokenize` performance | Open |
| [#235](https://github.com/micromark/micromark/pull/235) | Improve `splice` performance | Open |
| [#236](https://github.com/micromark/micromark/pull/236) | Improve HTML compile performance | Open |
| [#237](https://github.com/micromark/micromark/pull/237) | Improve performance of parsing without extensions | Open |
| [#238](https://github.com/micromark/micromark/pull/238) | Improve performance of the text and code text resolvers | Open |
| [#239](https://github.com/micromark/micromark/pull/239) | Improve attention resolver performance | Open |

### undici

12 experiments, 6 kept.

| Pull request | Description | Status |
|---|---|---|
| [#5901](https://github.com/nodejs/undici/pull/5901) | Avoid redundant request state in the `Request` constructor | Closed |
| [#5902](https://github.com/nodejs/undici/pull/5902) | Mask WebSocket frames without a mask array, four bytes per step | Closed |
| [#5903](https://github.com/nodejs/undici/pull/5903) | Check `ByteString` code units with a native scan | Open |
| [#5904](https://github.com/nodejs/undici/pull/5904) | Split each cookie pair once in `getCookies` | Open |

## License

MIT
