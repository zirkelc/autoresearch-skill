<div align='center'>
  <h1>Autoresearch</h1>
  <p align="center">A performance campaign your agent runs on its own. One small change at a time, each measured against a bar it calibrated, kept or discarded, logged either way.</p>
</div>

## Why

Performance work fails in a particular way. A change looks obviously faster, the benchmark moves three percent, and nobody can say whether that was the change or the machine. Then twenty changes land together, the suite is four percent faster, and no one knows which one paid for it or which one made it worse.

This skill makes an agent work the other way round. It builds an instrument first, measures how much that instrument lies, and only then starts changing code. Every experiment is one small change, committed before it is measured, kept only when two runs clear a bar that was measured rather than guessed. Everything discarded stays in the log with its number and its hash.

The result of a run is a branch of kept commits, a plan, a tab-separated log of every experiment including the failures, and a set of pull requests whose numbers a maintainer can reproduce.

## Origin

The idea and the name come from [karpathy/autoresearch](https://github.com/karpathy/autoresearch): give an agent a real experiment loop, a metric and a time budget, and let it run unattended. There, the agent edits one training script, trains for five minutes, reads `val_bpb`, keeps or discards, and repeats a hundred times overnight. The human does not touch the Python. The human iterates on `program.md`, the markdown that programs the agent, which that project calls "essentially a super lightweight skill".

This is that loop pointed at a different problem: making an existing library faster without changing what it does. It is not a fork and shares no code. What it takes is the shape of the thing, one change per experiment, a number that decides, a log that keeps the failures, and a human who edits the instructions rather than the diff.

Three things change once the domain does, and they are why this is twenty-five files of method and tooling instead of one:

**The metric is the hard part.** nanochat hands you `val_bpb` after a fixed five minutes, and it is comparable across any change the agent can make. A library gives you wall-clock time, where the effects worth finding are 1 to 25% and two runs of *identical code* can differ by more than that. So most of this skill is the instrument: two git revisions loaded into one process and timed in strict alternation, the delta taken as the median of per-iteration ratios, both load orders combined to cancel load-order bias, a noise floor measured before anything is changed, and a machine probe before and after every run.

**Correctness is not free.** A faster model is still a model. A faster library that returns different answers is a bug. So a characterisation guard is recorded and committed before the first experiment and never edited again, risky changes get a differential run over generated inputs, and the repo's own gates run on every commit.

**The output belongs to someone else.** A campaign ends in pull requests that a maintainer has to be willing to accept, which is a different bar from a better number. That brings its own rules: the reported figure comes from the measurement a reviewer will reproduce, the invariants a change relies on are named as costs, and the repo's conventions win over the skill's.

## How it works

Ten steps in `SKILL.md`. Nine of them are things to do, and the tenth, sitting in the middle, is the page of rules the loop runs under. The nine:

1. **Scope.** Which branch, which focus, how many experiments. Nothing starts before this.
2. **Explore.** How the repo builds, tests and benchmarks. Which metric its maintainers actually accept, from their merged PRs. Which gates cannot run on this machine at all.
3. **Guard.** Record current behaviour as hashes over the benchmark inputs, commit it, and never touch it again. A change that moves the guard is not an optimisation.
4. **Harness.** Copy in the runtime folder, wire up the cases, then prove the instrument with two canaries: a deliberate slowdown that only the right cases may notice, and a deliberate wrong answer that at least one gate must catch.
5. **Calibrate.** Run the comparison with identical code on both sides, three times, and write down what it reports. That is the noise floor. The keep bar is about twice it, never below one percent.
6. **Baseline and plan.** Profile, scan for superlinear paths, and write a ranked candidate list.
7. **Loop.** Change, gate, commit, measure, measure again if it cleared, keep or `git reset --hard HEAD~1`. Log the row either way.
8. **Stop.** Three non-kept experiments in a row means diversify, not finish. Then the budget.
9. **Package.** Group the kept commits into independent PRs, verify each one alone, and confirm every PR with the human before it is created.

### Three instruments, three jobs

The part that took three campaigns to get right. The same change measured three ways gives three different numbers, by up to a factor of two, in both directions:

| Instrument | Answers | Used for |
|---|---|---|
| Full suite, paired | Did anything else move? | The two summary numbers |
| Focused on one case, paired | Did the targeted case move? | Every per-case keep or discard |
| Standalone, one revision per process | What will a maintainer measure? | The number a PR is allowed to print |

Each is an honest measurement of something different. A paired run holds both revisions in one process, which makes it precise and makes the library's own objects polymorphic in shared code. A standalone run has none of that and is what someone else will get when they build each side separately. A standalone number is only evidence next to an identical-code control, because process-to-process spread differs by a factor of twenty between cases.

## What ships

```
SKILL.md                     the method, runtime-agnostic
references/methodology.md    why pairing, minima, noise floors and controls, and how to report them
references/pr-packaging.md   turning kept commits into PRs a maintainer will read
templates/                   plan, experiment log, PR body, fixture fetcher, gate script
runtimes/node-ts/            a working harness for JavaScript and TypeScript on Node
```

The node-ts runtime is the part you copy into a target repo:

| File | Purpose |
|---|---|
| `ab.mts` | Paired timing of two git revisions, both load orders, median of paired ratios |
| `solo.mts` | One revision per process, alternating, for the number a PR reports |
| `guard.mts` | Characterisation guard that can only ever gain cases, never rewrite one |
| `differential.mts` | Both revisions over generated inputs and named scenarios, one process each |
| `scan.mts` | Each input shape at n, 4n and 16n, to find superlinear paths |
| `profile.mts` | CPU profile per function, per caller, per line, or per area |
| `mem.mts` | Retained bytes per instance |
| `jitter.mts`, `quiet.sh` | Is the machine quiet enough to measure, and wait until it is |
| `micro.example.mjs` | Throwaway check of one function, to reject a candidate before it costs an experiment |
| `selftest.mts` | Builds a synthetic monorepo and checks the tree logic in seconds |

Another runtime is a folder that meets the contract in `runtimes/README.md`. Fourteen capabilities, each one there because a campaign needed it.

## Install

```bash
git clone https://github.com/zirkelc/autoresearch-skill.git ~/Developer/autoresearch-skill
ln -s ~/Developer/autoresearch-skill ~/.claude/skills/autoresearch
```

The directory under `~/.claude/skills` has to be called `autoresearch`, which is the skill's own name.

## Use

```
/autoresearch
/autoresearch error paths, 10 experiments
```

Or just say what you want: "run a performance campaign on this repo", "find performance wins in the parser". The skill asks for the scope and the budget before it touches anything, and it stops for confirmation before any pull request is created.

Expect it to take a while. A campaign is measurement-bound, not thinking-bound: budget about two and a half times the experiment count in measurement runs, plus the waiting for a quiet machine, which on a shared machine can be most of the session.

## Where it has run

Four campaigns on real repositories, 64 experiments, 32 kept, 18 pull requests:

| Project | Experiments | Kept | Result |
|---|---|---|---|
| [zod](https://github.com/colinhacks/zod) | 18 | 11 | -40% on the suite. 4 PRs, 2 merged |
| [linkedom](https://github.com/WebReflection/linkedom) | 18 | 8 | 2.02x on the suite. 4 PRs open |
| [micromark](https://github.com/micromark/micromark) | 16 | 7 | 6 PRs open |
| [undici](https://github.com/nodejs/undici) | 12 | 6 | 4 PRs open |

Half of the experiments were discarded, which is the point. The log of what did not work is the part that makes the rest believable.

Every campaign changed the skill afterwards, from its own defects: the estimator was wrong for the first two (a quotient of two minima instead of a median of paired ratios), the harness could not see a monorepo until the third, and the fourth found that a warning fired on a third of clean cases and that a published speed-up was not supported by its own control.

## History

This skill lived in [zirkelc/skills](https://github.com/zirkelc/skills) until it grew past the size of a directory in a shared repository. Its history moved with it, so the reasoning behind each rule is still in the commit that added it. Commit hashes referenced inside older commit messages point at the original repository, because splitting the history rewrote them here.

## License

MIT
