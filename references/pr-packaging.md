# Packaging kept commits as PRs

Turn the linear branch of kept commits into a few independent PRs that reviewers can judge one at a time. Each PR must stand alone: it applies to the base without the others, it passes the tests alone, and its numbers come from measuring it alone.

## 0. Review the kept commits, and treat the review as measurement

Read the kept commits again before you group them. A campaign optimises for one small change at a time, so the series arrives with duplicated helpers, an unrolled loop in one of two places that need it, and forms that were convenient while experimenting.

The rule that makes this safe is short: **any edit after the last measurement invalidates that measurement.** A review suggestion that touches a hot path is an experiment, and it gets the same treatment: the isolation check first, then the branch verification in section 4, and the numbers in the body come from the branch, never from the campaign log. In one campaign the review found no bugs and still changed the results three ways. One cleanup was neutral. One moved a shared helper so that a second code path got the unrolled loop as well, which doubled that PR's effect and made the campaign log an understatement. One was slower and would have shipped as an improvement, and the isolation check is what caught it.

So: review, then re-measure, then group. Never review after the verification table is written.

## 1. Check each change against earlier work and the maintainers' principles

Do this before you group, because it can remove or reshape a change.

For every kept change, search open **and** closed PRs:

```sh
gh pr list --repo <owner/repo> --state all --search "<function name>" --limit 20
gh pr list --repo <owner/repo> --state all --search "<file or module name>" --limit 20
gh pr list --repo <owner/repo> --state all --search "perf in:title <area>" --limit 20
```

Read every result that touches the same code, and for the closed ones read why they were closed. An earlier closed attempt at the same idea predicts your PR's result better than any measurement. In one campaign, a PR repeated a PR that the code owner had closed 13 days earlier with "we already have a better version". The new PR used the same helper name and the same unrolled loop, and it was closed as a duplicate within hours. A search of open PRs only had not found the earlier one.

Then check each change against the principles recorded in the plan during step 1 of `SKILL.md` (the maintainers' stated rules from their reviews of earlier performance PRs, and the code owner's own earlier attempts). Drop a change that breaks one of them, or reshape it so it does not. Do not argue for it in the PR body: a guard and a test suite prove that the behaviour is unchanged, and they cannot show that a change respects a design rule. In the same campaign, a second PR was closed because it merged steps that the code keeps separate on purpose to follow a specification, although its behaviour was identical.

If a change survives only in a form the owner has already rejected, leave it out and mention it in the summary to the user instead.

## 2. Group the commits

Group kept commits by theme (the path or mechanism they touch), not by the order in which they were made. Good groups are the ones a maintainer can accept or reject as a unit, for example "error construction", "schema construction", "parse fast paths". Four PRs from eleven commits worked well. A group with a single strong commit is fine.

Order the PRs by how easy they are to accept: the largest and least controversial first.

Before you present the grouping, check it mechanically: the union of the groups must equal the list of kept commits. Print the difference if it is not empty. A grouping table that silently lists 7 of 8 kept commits looks complete to everyone reading it.

Present the grouping to the user before you create branches.

## 3. Build one branch per group

Check that the base branch did not move (`git fetch`). For each group:

```sh
git worktree add <path> -b <branch> <base>
cd <path> && git cherry-pick <commit> <commit> ...
```

Branch names: hyphenated (`perf-error-path`), not slash-separated, because a remote branch named `perf` blocks every `perf/*` ref.

Three things about the worktree itself, each of which costs ten confused minutes the first time:

- Put it **outside** the repository directory, so no test glob, formatter or type checker finds a second copy of the sources.
- It has no `node_modules`. A symlink to the main checkout's is enough, but `.gitignore` says `node_modules/`, and a pattern with a trailing slash matches a directory, which a symlink is not. So the symlink shows as untracked until you add it to `.git/info/exclude`.
- Submodules are empty in a new worktree, which silently disables any gate that lives in one (a conformance suite, a fixture corpus). A symlink to the main checkout's submodule directory works; afterwards `trash` the symlink and recreate the empty directory, or the next `git submodule` command in the main checkout is confused.

Resolve conflicts so that each branch contains only its own group's code:

- Drop scratch edits that leaked into commits during the loop (for example `.gitignore` lines).
- When a conflict hunk contains code from another group, keep only the part that belongs to this group.
- Afterwards, grep each branch for symbols introduced by the other groups. The count must be zero.

Run the repo's full test suite **before** you copy the harness into the branch. The ignore entries that keep formatters and linters away from `perf/` live in the campaign's harness commit, not on a PR branch, so a copied-in harness fails the format gate on its own plan and scripts.

Install dependencies, then regenerate every committed derived artifact (types, bundled output) and fold the result into the commit that causes it, rather than adding a "regenerate" commit on top. Only some branches will change generated files, and the campaign branch may never have built them at all.

Then run the full test suite and the guard on each branch.

The guard needs the harness, and the harness does not exist on a branch that starts from the base. Copy it in as untracked files instead of committing it:

```sh
git checkout <campaign-branch> -- perf && git reset -- perf
```

Remove those copies again before switching back to the campaign branch, which tracks the same paths. The A/B harness itself does not need this: it materialises both revisions with `git archive`, so it can compare any two revisions from the campaign checkout. Only what runs against the working tree (the guard, and any run while the PR branch is checked out) needs the copy.

## 4. Verify each branch alone

Run from the harness location (the campaign branch):

1. One noise-control run with identical code on both sides.
2. Two A/B runs of the base against the branch, focused on the cases the PR targets where the full suite cannot resolve them.
3. A standalone run per headline case, alternating whole processes (`solo.mts A B <case>` in node-ts), **twice, on different occasions, each next to an identical-code control** (`solo.mts A A <case>`, same settings). An effect is resolvable standalone when its median lies outside the control's pair range in both runs. One case moved from -12.6% to -17.7% between days with tight controls both times, so one run is one sample of the day.

**Every number in the body comes from step 3, measured next to its control.** The reason is not that paired numbers are inflated: across five changes in one campaign the standalone number came out lower twice and higher twice. It is that a maintainer builds one revision per process, so that is the number they will measure, and a reviewer who reproduces something else stops believing the rest of the PR. Two PRs of an earlier campaign were closed over that kind of credibility.

The control is what turns the number into evidence. Process-to-process spread differed by a factor of twenty between cases of the same campaign, from +-1.5% to +-33%, so the same command resolves a 32% effect in one case and cannot resolve 9% in another. Where the control's spread covers the effect, say so in the body, give the focused number and name the instrument: one real change measured -8.5% and -9.8% focused against controls of +0.2% and +0.8%, and no standalone run could separate it from its own control.

This is not theoretical. In the campaign that produced this rule, one PR had already been published claiming 1.17x for a case whose control turned out to span -19% to +18%. Both of its standalone measurements sat inside that range, the claim was not supported, and the body had to be corrected after the fact. Measure the control before the number goes into a body, not after.

Build the verification table from these runs, never from the campaign log: isolated effects differ from stacked ones.

An asymptotic keep or a trade-off must be re-measured in isolation before its body is written. Suite neutrality in a stacked run is not evidence: one campaign's trade-off measured +0.25% stacked and +1.21% alone, and another's paired gain did not reproduce standalone at all. Where the stacked number differs materially from the isolated one, give both with the reason. This happens in both directions: a PR measured alone can look larger because it has the untouched path to itself, and a trade-off can look cheap alone but cost several percent once the other PRs removed the work that hid it. Two bodies of the same campaign must not state two different numbers for the same effect without explaining why.

Optionally, run the repo's own benchmark on the base and on each branch as an external cross-check. Quote a figure only under the rules in `methodology.md` (elephants, or paired in-process references, with caveats stated).

## 5. Write the PR bodies

A maintainer reads the body to decide one thing: whether to review the change. Keep it short and factual.

1. **The repository's PR template comes first.** Use the current template of the repository, or of its organisation (`.github/pull_request_template.md`, or the organisation's `.github` repository), with all its sections in their order. Take it from the repository, not from a merged PR, which can show an old revision. Fill in the template, keep machine markers such as `<!--do not edit: pr-->`, and tick only what is true. Do not put a structure of your own above or around it. Some repositories check the template with a bot and flag every PR that removes it. If the repository has no template, use `templates/pr-body.md`.
2. **Write in ASD-STE100 Simplified Technical English.** Short sentences with one statement each. Active voice. Common words with one meaning. No rhetorical phrases, no emphasis for effect, no sentences that only introduce the next sentence. This applies to the body and to every comment on the PR.
3. **Default content**, placed in the template's sections:
   - What changed: one or two sentences.
   - Why it is faster: one or two sentences.
   - One number: the standalone result for the targeted workload, with the runtime version. Name the workload, not the whole library.
   - Which tests ran, and that the behaviour is unchanged.
   - Only if there is one: an observable difference a reviewer could notice, or an invariant the change introduces. One sentence each.
4. **Everything else only if the user asks for it**: the campaign and methodology preamble, the verification table with pair ranges and controls, the paired numbers, reproduction commands, a link to the harness, links to companion PRs. These stay in the plan. If a maintainer asks how a number was measured, answer in a comment.
5. **Never put harness sources in a body.**

**Never tick a DCO or CLA checkbox.** It is a declaration by a person about their own work, and an agent cannot make it. Leave it unticked and say so when you present the PR.

Write bodies to files and pass them with `--body-file`. Inline heredocs break backticks and template literals. Placeholder PR numbers such as `#1 #2` link to, and notify, the old issues 1 and 2 of the target repository, so add cross-references only after the PRs exist.

## 6. Confirm and create, one PR at a time

Before any of this touches a remote, grep the plan, the log and the cases for private names, paths and hosts. A link to the campaign branch in a PR body or comment makes it public, and a plan written during the campaign names the downstream repo that motivated the work. One campaign published a private repository's name eight times that way. This is a blocking check, not a tidy-up.

Check the base's own CI before you open anything. When a PR shows failing jobs, compare the failing set with the base's last run: a failure that also fails on the base is not yours, and saying so in the body saves the maintainer the same investigation. Occasionally the comparison finds a real bug in their CI, which is worth its own issue.

On a **first** contribution to a repository, GitHub shows no checks at all until a maintainer approves the workflow runs. That looks exactly like broken CI, and the natural reactions (push again, ask what is wrong) are both wrong. Say so when you hand the PR over, and wait.

The base moves between the verification and the creation, sometimes by hours. Check what changed before you open anything:

```sh
git diff --stat <measured-base> origin/main          # do the touched files or the invariants overlap?
git merge-tree --write-tree origin/main <branch>     # does it still merge cleanly? no working tree touched
```

No overlap means the branch can stay on the base it was measured against, and the body can say which commit that was. An overlap in the touched files, or in the files an invariant depends on, means re-verify on the new base rather than rebase and hope.

`gh repo fork <owner>/<repo> --clone=false` is the form that works when you only need the fork; adding `--remote=false` to it failed.

Put decision items (the ideas that would change behaviour, from the plan) into the body of the PR they relate to, as an open question for the maintainers, with the behaviour risk named.

For each PR, show the user the title, the branch, the commits, the verification table and the body. Wait for confirmation. Then push with an explicit refspec and create the PR:

In zsh, write the refspec with braces (`"${b}:refs/heads/${b}"`): `$b:refs/heads/...` is read as a history modifier and pushes the wrong ref. In general, prefer a script file over an inline shell loop, because the traps live in the quoting.

```sh
git push <remote> <branch>:refs/heads/<branch>
gh pr create --repo <owner/repo> --base <base> --head <fork-owner>:<branch> --title "..." --body-file <file>
```

If the user wants a staging round, create the PRs on their fork first, then transfer them upstream: create the upstream PRs, update the bodies with the real numbers and head refs, and close the fork PRs with a link to the upstream PR.

If `gh pr create` fails with an API error, check with `gh pr view <branch> --repo <owner/repo>` whether the PR exists before you retry.

While a PR waits, the base moves. Re-verify before you nudge it, and re-check the invariants from its body against current base: work that lands after you open a PR can add exactly the state your change assumed nobody would add, which turns a correct change into a broken one without touching its diff.

When a scan or a profile finds the same bug in a dependency, treat that as a separate campaign in miniature: clone the dependency, run its own gates, compare two built copies of it (with an identical-copy control), and ask before creating a branch or a fork there.

Afterwards, keep the campaign branch, the plan and the log. They are the reference when a reviewer asks about a discarded alternative, and when later PRs need a rebase after earlier ones land.
