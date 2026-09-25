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
SKILL.md                     the method, independent of the language
references/methodology.md    how the measurements work and how to report them
references/pr-packaging.md   how to turn the kept commits into pull requests
templates/                   plan, experiment log, pull request body, scripts
runtimes/node-ts/            the harness for JavaScript and TypeScript on Node.js
```

The node-ts runtime is copied into the target repository:

| File | Purpose |
|---|---|
| `ab.mts` | Compares two git revisions in one process |
| `solo.mts` | Measures one revision per process, for the numbers in a pull request |
| `guard.mts` | Records and checks the behaviour of the benchmark cases |
| `differential.mts` | Compares the output of two revisions over generated inputs |
| `scan.mts` | Measures each case at three input sizes to find superlinear code |
| `profile.mts` | CPU profile per function, caller, line or directory |
| `mem.mts` | Retained memory per instance |
| `jitter.mts`, `quiet.sh` | Check that the machine is quiet enough to measure |
| `micro.example.mjs` | Template to compare two versions of one function before an experiment |
| `selftest.mts` | Tests the harness on a generated monorepo |

Other languages need a new runtime folder. `runtimes/README.md` lists what a runtime must provide.

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

The skill has run on four repositories:

| Project | Experiments | Kept | Result | Pull requests |
|---|---|---|---|---|
| [zod](https://github.com/colinhacks/zod) | 18 | 11 | 40% less time on the benchmark suite | [#6316](https://github.com/colinhacks/zod/pull/6316), [#6317](https://github.com/colinhacks/zod/pull/6317), [#6318](https://github.com/colinhacks/zod/pull/6318), [#6319](https://github.com/colinhacks/zod/pull/6319) (2 merged, 2 closed) |
| [linkedom](https://github.com/WebReflection/linkedom) | 18 | 8 | 2.02x faster on the benchmark suite | [#335](https://github.com/WebReflection/linkedom/pull/335), [#336](https://github.com/WebReflection/linkedom/pull/336), [#337](https://github.com/WebReflection/linkedom/pull/337), [#338](https://github.com/WebReflection/linkedom/pull/338) (open) |
| [micromark](https://github.com/micromark/micromark) | 16 | 7 | | [#234](https://github.com/micromark/micromark/pull/234), [#235](https://github.com/micromark/micromark/pull/235), [#236](https://github.com/micromark/micromark/pull/236), [#237](https://github.com/micromark/micromark/pull/237), [#238](https://github.com/micromark/micromark/pull/238), [#239](https://github.com/micromark/micromark/pull/239) (open) |
| [undici](https://github.com/nodejs/undici) | 12 | 6 | | [#5901](https://github.com/nodejs/undici/pull/5901), [#5902](https://github.com/nodejs/undici/pull/5902), [#5903](https://github.com/nodejs/undici/pull/5903), [#5904](https://github.com/nodejs/undici/pull/5904) (open) |

After each campaign, the skill was updated with the problems found during that campaign.

## History

The skill was first part of [zirkelc/skills](https://github.com/zirkelc/skills). It moved to this repository with its commit history. Commit hashes mentioned in older commit messages refer to the original repository.

## License

MIT
