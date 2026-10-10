# binary search python: a loop template that kills off-by-one bugs, the bisect module's real semantics, and "search on the answer" problems

You write a binary search in Python. It passes your test case. Then it loops forever on a two-element list, or returns the wrong index when the target appears three times, or you "just use `list.index()`" on a list of a million items and quietly turn an O(log n) solution into an O(n) one.

That last one is the most common version of the mistake, because Python makes it so easy. `if target in sorted_list` reads like a binary search. It isn't — it's a linear scan.

This piece covers the loop that actually holds up under edge cases, the two templates you need instead of one, what `bisect` really does (including the parts people get wrong), and how to recognize the problems where binary search isn't searching a list at all — plus what a structured pattern course like AlgoMonster covers on this specific topic and what it costs.

## What actually breaks

Three things go wrong, and they're distinct bugs:

1. **The infinite loop.** Usually `low = mid` or `high = mid` instead of `mid + 1` / `mid - 1`. When the range shrinks to two elements, `mid` becomes the same index forever.
2. **The missed element.** Usually `while low < high` in a template written for `<=`, or an off-by-one in the initial `high`. The element at the last remaining index never gets compared.
3. **The wrong duplicate.** You find *a* 4 in `[1, 2, 4, 4, 5]`, but the problem wants the *first* 4. A vanilla exact-match search returns whichever index the midpoint happened to land on.

Notice that the infinitely-looping version and the fast version are one character apart. That's the whole trap: binary search has no partial credit. It either terminates correctly on every input or it fails on the ones you didn't think to test.

## The loop template worth memorizing

python
def binary_search(arr: list[int], target: int) -> int:
    low, high = 0, len(arr) - 1
    while low <= high:                     # <= : low and high may point at the same element
        mid = (low + high) // 2            # // : integer division in Python 3
        if arr[mid] == target:
            return mid
        if arr[mid] < target:
            low = mid + 1                  # discard mid and everything left of it
        else:
            high = mid - 1                 # discard mid and everything right of it
    return -1


Three details carry all the weight:

- **`low <= high`, not `low < high`.** With `<=`, the loop keeps running while one candidate remains. Switch to `<` and you silently drop the last index — which is exactly the element in your one-element test case.
- **`//` not `/`.** In Python 3, `/` returns a float. `arr[mid]` with a float index raises `TypeError`, so you'll catch this one fast. But if you're translating from C or Java, this is the line that changes.
- **`mid + 1` and `mid - 1`.** This is the line that prevents the infinite loop, and it's safe because you already compared `arr[mid]` on the line above. In C++ or Java you'd write `low + (high - low) // 2` to avoid integer overflow on large indices. Python integers are arbitrary precision, so `(low + high) // 2` never overflows — but writing the subtraction form anyway is a habit that transfers when you switch languages mid-interview.

The precondition is non-negotiable: the input must be sorted. Binary search on an unsorted list doesn't fail loudly, it returns a plausible wrong answer.

## Two templates, not one

Most tutorials teach the exact-match loop and stop. That's why people freeze on "find the first element greater than or equal to target" or "find the minimum capacity that works." There's a second template — the boundary search — and it's the one that shows up more often in real interview questions.

|  | Exact match | Boundary / first-true |
| --- | --- | --- |
| **Loop condition** | `while low <= high` | `while low <= high` with an `ans` variable, or `while low < high` |
| **What you're asking** | "Is `target` at this index?" | "Is the first index satisfying `feasible(x)` here or to the left?" |
| **On a hit** | Return `mid` immediately | Record it, then keep searching left for an earlier one |
| **Returns** | Index of target, or `-1` | The boundary index, or `-1` if nothing is feasible |
| **Used for** | `target in sorted_list`-style lookups | First/last occurrence, `bisect` semantics, search on the answer |

The boundary version:

python
def first_true(arr, feasible) -> int:
    low, high = 0, len(arr) - 1
    ans = -1
    while low <= high:
        mid = (low + high) // 2
        if feasible(mid):      # monotonic: FFFF TTTT
            ans = mid          # mid is a candidate; remember it
            high = mid - 1     # look for an earlier True
        else:
            low = mid + 1      # mid is False; everything left of it is too
    return ans


The mental model that makes this click: stop thinking about values and start thinking about a boolean array. For a sorted array and the condition `arr[i] >= 5`, the answers form `F F F T T T T` — all the `False`s on the left, all the `True`s on the right. Binary search finds the seam. Every "lower bound," "first occurrence," and "smallest capacity that works" problem is that same seam.

## Python already ships a binary search: `bisect`

For the boundary case, you usually don't need to write the loop at all. The standard library's `bisect` module implements it in C, and the boundary conditions are already correct — which is the practical argument for using it over hand-rolled code.

The two functions that matter:

python
import bisect

nums = [1, 2, 4, 4, 5]
bisect.bisect_left(nums, 4)    # 2  → first index where nums[i] >= 4
bisect.bisect_right(nums, 4)   # 4  → first index where nums[i] > 4
bisect.bisect(nums, 4)         # alias for bisect_right


Those return values are *insertion points*, not "found" indices. That's a deliberate design choice: `bisect` never calls `__eq__()`, only `__lt__()`, so it works on any type that supports ordering — including types where "equal" is ambiguous.

The classic duplicate-counting trick falls straight out of this:

python
count = bisect.bisect_right(nums, 4) - bisect.bisect_left(nums, 4)   # 2


Four things about `bisect` that are easy to miss:

- **`insort_left` / `insort_right` don't make insertion cheap.** The search is O(log n), but `list.insert` shifts elements, so the whole operation is O(n). The docs are explicit about this: the logarithmic search is dominated by the linear insertion step. For a sorted structure you insert into constantly, `bisect` on a list is the wrong tool.
- **The `key` parameter only exists in Python 3.10+.** It lets you bisect a list of records by an extracted key. One wrinkle: the key function is applied to elements but not to the value you're searching for, so you pass the pre-extracted key, not the record.
- **Not thread-safe.** Concurrent `bisect` calls on the same list, or a `bisect` running while another thread mutates the list, give undefined results. `insort_left` from multiple threads can leave the list unsorted.
- **`bisect` is a tool, not a replacement for understanding.** If an interviewer asks you to implement the loop, "I'd import bisect" is a fine production answer and a bad interview answer.

## Where binary search actually earns its keep

The list-search version is the easy one. The version that decides interviews is binary search on the answer: you're not searching an array at all, you're searching over a *range of possible answers*.

The recipe:

1. Define a monotonic predicate `feasible(x)` — a function that returns `False` for all values below some threshold and `True` for all values at or above it (or the reverse).
2. Set the search bounds to the range of possible answers, not to array indices.
3. Binary search for the seam.

The classic shape is "minimize the maximum." Given a set of tasks and a fixed number of workers/machines/days, find the smallest capacity that finishes the work. Check whether a capacity of `x` is enough → you get `FFFFTTTT` → binary search the seam.

python
def min_capacity(weights, days):
    def feasible(cap):                     # monotonic: small cap fails, large cap works
        used, current = 1, 0
        for w in weights:
            if current + w > cap:
                used += 1
                current = 0
            current += w
        return used <= days

    low, high = max(weights), sum(weights)  # bounds are answers, not indices
    while low < high:
        mid = (low + high) // 2
        if feasible(mid):
            high = mid                     # mid works; try smaller
        else:
            low = mid + 1                  # mid fails; must go bigger
    return low


Two things bite here. First, when you change what's being searched, the loop style often switches to the `low < high` form, where `low == high` is the answer — mixing the `<` loop with the `ans = -1` idiom from the previous template produces the missed-element bug. Pick one convention per function and stay in it.

Second, the bounds. Setting `low = 0` when every valid answer is at least `max(weights)` costs you log-factor iterations of nonsense. Setting `high` too low returns a wrong answer that still looks like a number.

## A bug checklist for your own solutions

Run through this when a binary search fails and the reason isn't obvious:

- Input actually sorted? A monotonic predicate actually monotonic? (An array like `[3, 1, 4, 1]` under "element > 4" is not monotonic, and no loop fix saves it.)
- `low <= high` vs `low < high` — do they match the rest of the template?
- `mid + 1` / `mid - 1`, or `mid`? If any branch assigns `mid` back to a bound without shrinking, you have an infinite loop.
- Does the function return on *every* exit path? A `while` loop that falls through returning `None` is a silent bug that only shows up on misses.
- Did you use `list.index()`, `in`, or `.count()` on the sorted list anywhere? Each is O(n) and erases the reason you wrote the search.
- Empty input. `len(arr) - 1` on `[]` gives `high = -1`, and the loop correctly skips — but only if you didn't do something before the loop.
- Integer division. `mid = (low + high) / 2` in Python 3 yields a float and breaks indexing.

## Practicing it as a pattern, not a lucky fix

Binary search is one of the patterns where "I've seen this before" carries less weight than "I know which of the two templates this is." That distinction is exactly what pattern-based prep platforms organize around. AlgoMonster builds its curriculum around 48 explicit coding patterns — binary search among them — with decision flowcharts that map problem-statement keywords to the right approach, template code in six languages, and roughly 231 mapped lessons and problems across the platform. The content is text and interactive diagrams rather than video, which is either a feature or a dealbreaker depending on how you learn.

Their binary search track walks the progression described above: vanilla search over a sorted array, then first occurrence / boundary search, then the monotonic-function framing where you stop thinking about values and start thinking about a boolean array, then binary search on the answer. There's also a Speedrun mode for timed review before an interview, and an AI assistant for when you want a hint rather than the editorial. The published curriculum is built to be worked through in about 4–8 weeks at 1–2 hours a day.

If you want to see whether the format fits before paying, the free tier gives you a slice of the patterns and problems:

👉 [Try AlgoMonster's free tier and poke at the binary search lessons](https://algo.monster?fpr=li69)

Worth setting expectations: the honest complaint about AlgoMonster in third-party reviews is problem volume. Roughly 231 mapped problems is a fraction of LeetCode's catalogue, and the platform is text-only with limited system design coverage. The pitch is pattern recognition, not reps — most people who use it pair it with LeetCode for volume.

## AlgoMonster plans and current pricing

There's no monthly plan. Every plan unlocks the same content; the difference is how long you keep it.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the format | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant and Speedrun; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; company questions updated monthly; bonus Foundation and Build an LLM courses; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | 12 months, cancel or upgrade anytime | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates and permanent access | $189 | One-time, billed once | [Get Pro Lifetime access](https://algo.monster?fpr=li69) |

Two caveats worth stating plainly. First, this pricing is promotional and has moved around a lot historically — older reviews cite everything from $99 to $459 for lifetime access, and community threads on AlgoMonster's own forum show people waiting for a $99 lifetime window to return. What you actually pay is whatever the checkout page shows when you get there, so check it before deciding. Second, both paid plans are currently shown at a discount off the listed price, and the total is calculated on the checkout page.

The break-even math is simple enough to do yourself: if the annual plan renews at $99 and lifetime is $189, lifetime wins if you'd renew twice. If you're prepping for one interview loop in the next two months, annual covers it.

## FAQ

**Does Python have a built-in binary search?**
Not as a single `binary_search()` function, but `bisect.bisect_left()` and `bisect.bisect_right()` implement exactly that boundary search in C. For "find the insertion point" or "count occurrences in a sorted list," use them. For interviews that ask you to implement the algorithm, write the loop.

**Why does my binary search loop forever?**
Almost always because a branch assigns `mid` to `low` or `high` without the `± 1`. When the range narrows to two adjacent indices, `mid` recomputes to the same value every iteration. Check both branches, not just the one you think is wrong.

**`bisect_left` or `bisect_right`?**
`bisect_left` gives the first index where the value could be inserted without breaking sorted order — i.e. the first element `>= x`. `bisect_right` gives the first index where elements are strictly `> x`. If duplicates are involved, that one position is the entire difference between "first occurrence" and "after the last occurrence."

**Do I need to buy anything to learn this pattern?**
No. The `bisect` documentation is free, and the two templates above cover most interview variants. A paid course earns its money on sequencing and feedback — knowing which pattern a problem statement is pointing at, and getting unstuck without reading a full editorial. That's a real gap if you're learning alone, and it's not worth paying for if you already recognize the pattern on sight.
