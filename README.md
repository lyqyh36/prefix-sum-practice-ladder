# prefix sum: the O(1) range-sum trick, the off-by-one traps, and a practice ladder from LC 303 to subarray-sum-equals-k

You get an array of 100,000 numbers and 100,000 queries. Each query asks for the sum of `nums[l..r]`. Nobody hands you a hint, and the brute-force loop is the most obvious thing in the world:

python
def range_sum(nums, l, r):
    return sum(nums[l:r+1])   # O(n) per query


That's roughly 10^10 additions in the worst case. On a normal judge with a 1–2 second limit, it dies. Not because the logic is wrong, but because every query re-adds elements that a previous query already added. Query `[0, 2]` and query `[2, 5]` both add `nums[2]`. Multiply that overlap by 100,000 queries and you've written a program that spends most of its life doing the same arithmetic twice.

Prefix sum is the fix, and it's one of the few techniques you can learn in ten minutes and then recognize instantly for the rest of your interview career. The catch is that almost everyone who "knows" it still writes the wrong index on the first attempt.

## The one identity that does all the work

Define `prefix[i]` as the sum of the first `i` elements:

text
prefix[i] = nums[0] + nums[1] + ... + nums[i-1]


Then the sum of `nums[l..r]` inclusive is:

text
prefix[r + 1] - prefix[l]


Two array reads, one subtraction, no matter how wide the range is. Build the table once in `O(n)`, answer every query in `O(1)`. Total cost drops from `O(n × Q)` to `O(n + Q)`.

That subtraction is worth staring at for a second, because it explains why the technique works at all. `prefix[r+1]` contains everything up to `r`. `prefix[l]` contains everything up to `l-1`. The second set is a subset of the first, so subtracting cancels the shared head and leaves exactly the slice you asked for. It's the same reason `sum(l, r)` works in SQL over a running total, and the same reason an accounting ledger can compute "revenue in Q3" by subtracting two cumulative figures.

Read that as: the hard part of prefix sum isn't the arithmetic. It's committing to one precise meaning for `prefix[i]` and never drifting from it.

## Building it in Python, and why the array is n+1 long

python
class NumArray:
    def __init__(self, nums):
        n = len(nums)
        self.prefix = [0] * (n + 1)          # prefix[0] = 0 is the sentinel
        for i in range(n):
            self.prefix[i + 1] = self.prefix[i] + nums[i]

    def sumRange(self, left, right):
        return self.prefix[right + 1] - self.prefix[left]


Three details in that code carry everything else.

**The array has length `n + 1`, not `n`.** The extra slot is `prefix[0] = 0`, the empty-sum sentinel. Without it, a query starting at `l = 0` needs a special case, because there is no `prefix[-1]` that means "nothing yet." With the sentinel, one formula covers every possible range, including the full array. It's the same trick you'll reuse when the hash-map version of this pattern initializes `{0: 1}` — a sentinel that makes the boundary case work without an `if`.

**`prefix[i]` is exclusive on the right.** It's the sum of `i` elements, not the sum through index `i`. Most indexing bugs come from mixing this up mid-solution.

**Use a 64-bit type in C++ and Java.** The Tencent write-up on this algorithm flags it directly: with large elements or a long array, the prefix sum overflows a 32-bit `int` (max about 2.1e9) fast. `long long` and `long` are not optional. Python users never hit it, which is exactly why Python users ship this bug to production in other languages.

## The two conventions, and how to stop mixing them

There are two indexing conventions living in the wild, and both are correct as long as you pick one and stay inside it:

- **0-indexed, half-open:** `prefix` has length `n + 1`, `prefix[0] = 0`, and the answer for `[l, r]` inclusive is `prefix[r + 1] - prefix[l]`.
- **1-indexed, inclusive:** the array is conceptually `a[1..n]`, `f[i] = a[1] + ... + a[i]`, `f[0] = 0`, and the answer for `[l, r]` is `f[r] - f[l - 1]`.

They produce identical results on valid ranges. Problems start when you build the table with one convention and query it with the other. A useful sanity check that costs nothing: after building, verify that the last prefix value equals the sum of the whole array, and that `range_sum(0, n-1)` matches a single pass with `sum()`.

| Symptom | Actual cause | Fix |
| --- | --- | --- |
| Wrong answer only when `l = 0` | No sentinel slot, so the left endpoint has no legal subtraction | Allocate `n + 1`, set `prefix[0] = 0` |
| Off by one element on every query | Querying with `prefix[r]` instead of `prefix[r + 1]` | Decide half-open vs inclusive once, write it in a comment |
| Correct on small tests, wrong on big ones | 32-bit overflow in the prefix array | Use 64-bit integers |
| Correct answers, time limit exceeded | Rebuilding the prefix table inside the query loop | Build once in `__init__`, answer in the method |
| Wrong after modifying the array | Prefix sums are static by design | Move to a Fenwick tree or segment tree |

The last row matters more than it looks. Prefix sums assume the data doesn't change. If queries and updates are interleaved, you've outgrown the technique.

## 2D prefix sums are inclusion–exclusion

The same idea extends to matrices, and the formula is where most people first meet inclusion–exclusion without being told that's what it is.

Build with a padding of one row and one column:

python
prefix[i][j] = matrix[i-1][j-1] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1]


Answer a rectangle query from `(r1, c1)` to `(r2, c2)`:

python
prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1]


The subtraction removes the strips above and to the left; the final addition puts back the corner that got subtracted twice. Four lookups, `O(1)`, regardless of how large the rectangle is. A LeetCode discussion thread on 1D and 2D prefixes walks through a full worked grid, and it's worth building one by hand once — the double-subtracted corner is much easier to remember after you've seen it go wrong.

## Difference arrays: prefix sums running in reverse

Flipping the loop gives you the inverse operation, and it solves a completely different problem: many range updates, one final read.

To add `x` to every element of `[a, b]`, don't touch the range. Instead:

python
diff[a] += x
if b + 1 < len(diff):
    diff[b + 1] -= x


Then run one prefix pass over `diff`, and every position holds its accumulated value. Each range update is `O(1)` instead of `O(n)`. Classic uses: LeetCode 1109 (Corporate Flight Bookings), where every booking is a range update over a seat range, and the USACO "difference array" problems where dozens of intervals overlap and you need the final values only.

Same trick in 2D: add `v` at `(x1, y1)`, subtract at `(x1, y2+1)` and `(x2+1, y1)`, add back at `(x2+1, y2+1)`, then take the 2D prefix. It's the corner logic from above, applied backwards.

## Prefix sum plus a hash map: the version interviewers actually ask

Static range-sum queries are a warm-up. The interview-shaped variant is counting subarrays.

"If `prefix[j] - prefix[i] = k`, then the subarray between `i` and `j` sums to `k`." Rearranged: for each position `j`, count how many earlier prefix values equal `prefix[j] - k`. That's a two-sum problem in disguise, and a hash map answers it in `O(n)`:

python
def subarray_sum(nums, k):
    counts = {0: 1}          # the sentinel again: the empty prefix
    running = 0
    total = 0
    for x in nums:
        running += x
        total += counts.get(running - k, 0)
        counts[running] = counts.get(running, 0) + 1
    return total


`{0: 1}` is the load-bearing line. Without it, subarrays starting at index 0 are invisible, because there's no earlier prefix value to pair with. This is the same sentinel idea as `prefix[0] = 0`, expressed in a dictionary instead of an array.

Related problems in the same family:

- **Contiguous Array (LC 525)** — treat `0` as `-1` and find the longest subarray summing to zero. Equal counts of 0s and 1s means a zero-sum range, and you keep the *earliest* index of each prefix value because you want maximum length.
- **Number of Sub-arrays With Odd Sum (LC 1524)** — count prefixes by parity instead of value.
- **Number of Ways to Split Array (LC 2270)** — compare a prefix sum to its complement; no hash map needed.
- **Subarray Sums Divisible by K** and its modulo cousins — count prefixes by remainder.

The general question is always the same: can the thing I care about be encoded as something additive over prefixes, and expressed as a difference of two prefixes? If yes, the implementation is usually ten lines.

## Where prefix sums stop working

Three failure modes, all of which come up in interviews as follow-ups.

**Updates.** Prefix sums are a snapshot. One element change invalidates every prefix after it. If the problem interleaves `update(i, val)` with `sumRange(l, r)`, you need a Fenwick tree or a segment tree, both `O(log n)` per operation. A 2020 paper on the prefix-sum problem states the trade-off plainly: flat arrays give `O(1)` queries with `O(n)` updates, while tree structures balance both at `O(log n)`, and `O(log n)` is optimal for the dynamic version when values fit in a machine word. Interviewers love asking this as "now what if the array changes?" — and the expected answer is a data structure switch, not a clever prefix hack.

**Overflow.** Covered above, but it costs people real submissions.

**Negative or unbounded ranges.** Prefix sums handle negatives fine; sliding window does not. If the values can be negative and the question is about a sum condition, that's a signal to reach for prefix sums plus a hash map rather than two pointers.

## A practice ladder that builds actual recognition

Doing ten prefix-sum problems in random order teaches you ten problems. Doing them in this sequence teaches you the pattern.

1. **Running Sum of 1d Array (LC 1480)** — build the accumulator once. Trivial on purpose.
2. **Range Sum Query - Immutable (LC 303)** — the sentinel, the `n + 1` array, the subtraction. This is the core pattern in its purest form.
3. **Number of Ways to Split Array (LC 2270)** — split-point logic. Compare prefix against its complement.
4. **Product of Array Except Self (LC 238)** — the same precompute-once thinking with multiplication instead of addition.
5. **Corporate Flight Bookings (LC 1109)** — difference array, range updates, single final pass.
6. **Contiguous Array (LC 525)** — prefix plus hash map, earliest-index bookkeeping.
7. **Subarray Sum Equals K (LC 560)** — the canonical hash-map variant.
8. **Number of Sub-arrays With Odd Sum (LC 1524)** — parity-counting prefixes.
9. **Range Sum Query 2D - Immutable (LC 304)** — inclusion–exclusion on a grid.
10. **CSES 1660 / 1661** — subarray sum and subarray sum II; harder counting variants on the same skeleton.

If you want a single behavioral goal: after this ladder, you should be able to read a problem statement mentioning "subarray", "range sum", or "number of ways to split" and know within about forty seconds whether the answer is a flat prefix array, a prefix array with a map, or a difference array. That identification step is what interviewers are actually testing, and it's a different skill from writing the loop.

One more piece of context worth knowing, because it explains why this idiom is everywhere: prefix sums are the canonical parallel algorithm. The standard formulation splits into an up-sweep that builds a tree of partial sums bottom-up and a down-sweep that pushes "sum of everything to my left" back down to the leaves — `O(n)` work and `O(log n)` span, originally described by Ladner and Fischer in 1977. The same structure underlies stream compaction, radix sort, and GPU scan primitives. Learning it as a LeetCode trick sells it short.

## Where a structured curriculum fits in

Prefix sum is not hard. The hard part is that it lives among about a dozen sibling patterns — two pointers, sliding window, binary search on the answer, interval handling, monotonic stacks — and interview problems rarely announce which one they want. Most people who stall out aren't missing knowledge; they're missing the mapping from problem statement to pattern.

That's the specific gap AlgoMonster is built around. It's a text-and-diagram curriculum organized by pattern rather than by problem count, put together by engineers from Google, and the lessons come with code templates plus practice problems on top of each concept. For a technique like prefix sum, the relevant value isn't "here's the formula" — plenty of free blog posts have the formula — it's the decision flowcharts for choosing a pattern, and a ladder of problems sequenced so each one exercises the same idea in a new disguise.

If you want to see how the prefix-sum material is presented before paying anything, the free tier gives you access to a limited subset of patterns and lessons, which is enough to judge whether the format works for you. 👉 [Start with the free plans and poke around the pattern lessons on AlgoMonster](https://algo.monster?fpr=li69).

### Current AlgoMonster plans

AlgoMonster's pricing page has three tiers and no monthly billing option. The listed prices move with sales, and the annual plan is currently shown as a promotional rate rather than the full $198/year list price:

| Plan | What's included | Price | Billing | Get access |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of content; enough to evaluate the teaching style | $0 | — | [Open the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; monthly-updated questions from top tech companies; member discount on live bootcamps; bonus CS fundamentals and build-an-LLM courses | $99/year (shown as $8.25/month billed annually) | Annual, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates; one payment, permanent access | $189 | One-time | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

Two things to check at checkout before you buy:

- **Discount depth varies.** The subscribe page itself has shown both $99/year and, on a deeper promotional snapshot, $69.99/year with lifetime at $89 — the sale ladder changes. Third-party discount pages also list a partner code (currently `SCHELLEY`) that stacks an extra 10% on top of an active sale, applied through the "Add promotion code" field. Codes like that rotate, so verify the total on the payment screen rather than trusting a blog post.
- **Not everything is bundled.** Specialized courses developed with external partners, such as Quant Interview Prep, are sold separately. The core pattern curriculum, the CS fundamentals course, and the LLM course are included.

### Is it the right fit for prefix-sum prep specifically?

Depends on what you're missing.

If your problem is that you can't write the formula, you don't need a subscription — the sections above are enough, and LeetCode 303 will confirm it in twenty minutes. If your problem is that you solve `Subarray Sum Equals K` after seeing the tag and then blank on `Contiguous Array` three weeks later, you're dealing with pattern recognition, and a structured curriculum with decision flowcharts and sequenced problems targets exactly that.

Reasonable objections, based on what third-party reviews consistently point out: the platform is entirely text and diagrams, with no video walkthroughs; the problem bank is far smaller than LeetCode's, so it's a curriculum rather than a practice archive; and system design plus behavioral content is introductory rather than senior-level, so candidates targeting staff-level loops will need another resource. Lifetime access is the plan that makes the pricing math work for most people, since it's a one-time $189 versus a recurring annual fee — roughly one and a half years of the current annual rate.

For a candidate prepping over the next two or three months, annual is the lower-commitment choice. For an engineer who expects to interview again in two or three years — which, anecdotally, is most of them — lifetime is straightforwardly cheaper. 👉 [Compare the current AlgoMonster plans and pricing here](https://algo.monster?fpr=li69).

## The short version

Prefix sums turn repeated interval work into one subtraction, and the entire technique rests on a single discipline: fix what `prefix[i]` means, allocate `n + 1` slots so the empty prefix exists, and never mix conventions mid-solution. Add a hash map and you can count subarrays with a target sum in linear time. Add a difference array and you can apply thousands of range updates in constant time each. The moment the array starts changing between queries, the technique is finished and you switch to a tree.

Everything else — the 2D inclusion–exclusion, the parity tricks, the modulo variants — is the same identity wearing a different hat.
