# Visualizing the results

Step 10 of `SKILL.md`. The output is one self-contained HTML file: no build step, no external scripts, no external fonts, data inline. How the file is shared or published is up to the user and the environment.

Start from `templates/results.html`. It is data-driven: one `CONFIG` object and one `ROWS` array feed every view, so the views cannot disagree with each other.

## Contents

- Why this is a step of its own
- Rules
- Which chart for which kind of result
- Pitfalls

## Why this is a step of its own

When a campaign ends, someone asks for a headline and a chart. Two things in the method apply to that request and are easy to get wrong:

- **The obvious headline can be false.** In one campaign, four PRs made their functions 1.12x to 1.48x faster, and all four together changed a complete `fetch()` call by -1.5%, inside the noise. A chart of the functions alone invites the sentence "fetch() is 30% faster". The page must show the whole user-facing operation next to the parts.
- **Only one kind of number may be drawn.** Paired in-process numbers differed from standalone numbers in both directions (-69% paired against -32% standalone for one case, -18% against -23% for another). The chart shows the same standalone numbers as the PR bodies.

## Rules

1. **Show the whole user-facing operation.** Measure it standalone with its control, and draw it as its own row in grey, labelled "no measurable change" if its median is inside the control's range. This row is what makes the other rows believable.
2. **Only standalone numbers, each with its identical-code control.** Draw only rows that passed the resolvability test in step 9. Say in one sentence under the chart how the numbers were measured. Keep medians, pair ranges and controls in a table view and in tooltips, not in the main marks. A chart whose reading rule needs the reader to compare whiskers with control bands is too hard to read at first glance, and whisker-against-band is also the wrong test (the claim is the median, not a range of single pairs).
3. **Use the numbers from the PR bodies, unrounded.** Rounding -32.24% to -32.2% before computing the multiplier gave 1.47x on one page and 1.48x in the PR. Store the unrounded median and round only when you print.
4. **Label workloads, not functions,** unless the workload calls exactly one changed function. A row called "cookie parsing" that runs three functions, of which one changed, claims more than was measured. A label is a claim.
5. **One framing per chart: time or throughput.** The multiplier (1.48x) is the same in both framings and is the safest headline. The percentages are not (-32% time is +48% throughput). A page that shows both framings labels each chart with its framing.
6. **Use the framing the target ecosystem reads.** Check the project's README, its benchmark scripts and the titles of recent performance PRs: requests or operations per second, time per operation, bytes, or percent. In the Node.js ecosystem, throughput is usual.
7. **Grey is the baseline, one accent colour is the change, muted grey is a non-result.** A regression gets its own colour and the word "slower". The same colour means the same thing in every view. Every mark also has a direct value label, so no information is in colour alone.
8. **Show the status of each PR.** PRs get merged and closed after the page is made. Store a status per PR, render it, and update it before the page is shared again.
9. **Look at the rendered page once as a first-time reader** before you call it done. The first version in one campaign passed every data check and still failed its reader.

## Which chart for which kind of result

The kind of result picks the chart.

| What the campaign changed | Main chart | Why | Secondary views |
|---|---|---|---|
| CPU time of several independent workloads | Before/after paired bars per workload, baseline = 100, in the ecosystem's framing, multiplier as text under each pair | Reads in one second, and the baseline is visible in every row | Headline tiles, slope chart, table |
| One dominant change on one workload | One before/after pair, or a large tile with the absolute value (ns per operation, operations per second) | One number needs no axis | Absolute time per call from the repo's own benchmark tool |
| A superlinear path made linear (from the scaling scan) | Time against input size, base and change, both axes logarithmic, points at n, 4n and 16n | The win is a slope, not a percentage: small at realistic sizes, large at big ones | Table of the times and step ratios |
| Retained memory per instance | Paired bars in bytes per instance | The measurement is deterministic, so there are no ranges, and maintainers quote bytes | Percentage as text |
| Bundle size | Paired bars of compressed bytes (gzip and brotli), with the project's size budget as a reference line if it has one | Minified bytes mislead, and the budget line shows the constraint that matters | Table per entry point |
| A trade-off (one case faster, another slower) | Diverging bars around zero, one row per case | Shows the cost next to the win instead of averaging it away | Total and geometric mean as text |
| Latency percentiles, if the campaign measured them | Percentile bars (p50, p90, p99) base against change | An average hides a change in the tail | |
| The campaign itself, for the plan rather than for a PR | Step chart over experiment number: cumulative improvement of the kept changes, discarded experiments as grey marks | Shows what each keep contributed and how many attempts it took | The experiment log as a table |

## Pitfalls

- **Encoding.** A file with non-ASCII characters (the multiplication sign, a middle dot, a minus sign, arrows) showed broken characters when it was served without a charset. Declare `<meta charset="utf-8">`, and use HTML entities in markup and `\u` escapes in scripts as well.
- **The `hidden` attribute** only hides an element when a stylesheet says so. Outside an environment that ships that rule, hidden elements showed. The template styles `[hidden]` itself.
- **Tiles in an auto-fit grid** left the last tile alone on a new row. Use a column count that the number of tiles fills; the template computes it.
- **External resources.** A page that loads fonts or scripts from a network is not self-contained and looks different offline. The template uses system fonts.
