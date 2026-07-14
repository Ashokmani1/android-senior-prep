# Coding Rounds & Company Questions

This kit is built around architecture, modularization, system design, and leadership — the rounds
fundamentals don't win (see [the kit's premise](../index.md)). But Senior/Lead loops at several
companies also run a **DSA-lite coding round**: short, practical problems, not LeetCode-hard,
usually solvable in 15–20 minutes with clean code and the right edge cases called out.

This page does two things:

1. **Maps every question from real loops** to the module that already answers it, so you're not
   hunting.
2. **Solves the coding problems that genuinely have no home elsewhere in this kit** — worked in
   Kotlin, with the complexity and the edge case an interviewer is listening for.

---

## Company round map

| Company | Question | Where it's answered |
|---|---|---|
| Booking.com | Implement `findViewById` | [Worked below ↓](#implement-findviewbyid) |
| Booking.com | Group mutual anagrams | [Worked below ↓](#group-mutual-anagrams) |
| Booking.com | Classify four sides as square / rectangle / neither | [Worked below ↓](#square-rectangle-or-neither) |
| Booking.com | Delta-encode a sequence | [Worked below ↓](#delta-encoding) |
| Booking.com | Common elements across three arrays (with duplicates) | [Worked below ↓](#common-elements-in-three-arrays) |
| Booking.com | `ConcurrentModificationException` twist | [Worked below ↓](#concurrentmodificationexception) |
| Booking.com | Design a hotel list + detail screen, what APIs, layout | [System Design framework](../system-design/framework.md) · [Clean Architecture](../architecture/clean-architecture.md) — treat it as a paged list screen ([Instagram feed prompt](../system-design/questions.md#2-design-instagrams-feed)) + a detail screen backed by a single "get by id" endpoint |
| Booking.com | Fragments & Activity lifecycle, Views, Layouts | [M1 Activity](../deep-dive/01-activity.md) · [M2 Fragment](../deep-dive/02-fragment.md) · [M3 View System](../deep-dive/03-view-system.md) |
| Booking.com | Background tasks — AsyncTask, Service, IntentService | [M16 Services](../deep-dive/16-services.md) · [M17 WorkManager](../deep-dive/17-workmanager.md) |
| Booking.com | Busiest day from check-in/check-out dates | [Worked below ↓](#busiest-day-interval-sweep) |
| Booking.com | Elements repeated more than N times | [Worked below ↓](#elements-repeated-more-than-n-times) |
| Booking.com | Classify a review as positive/negative/neutral from word lists | [Worked below ↓](#sentiment-from-word-lists) |
| Spotify | Design Search feature architecture | [System Design framework](../system-design/framework.md) |
| Spotify | Design a disk-based cache (cross-platform, secure, opaque, 100k+ objects) | [Client-side caching layer](../system-design/questions.md#6-design-a-client-side-caching-layer) — for the specific constraints, see [note below ↓](#disk-based-cache-constraints) |
| Spotify | Linked List Cycle | [Worked below ↓](#linked-list-cycle) |
| Spotify | Palindrome Linked List | [Worked below ↓](#palindrome-linked-list) |
| PhonePe | Minimum meeting rooms | [Worked below ↓](#minimum-meeting-rooms) |
| PhonePe | Anagram strings | [Worked below ↓](#group-mutual-anagrams) |
| Paytm | String rotation check + rotation count | [Worked below ↓](#string-rotation) |
| Paytm | Pangram check | [Worked below ↓](#pangram-check) |
| Paytm | Design Search feature architecture | [System Design framework](../system-design/framework.md) |
| Paytm | Sort an array of 0s, 1s, 2s | [Worked below ↓](#sort-0s-1s-2s-dutch-national-flag) |
| Paytm | Abstract vs Interface | [Kotlin fundamentals](../kotlin/fundamentals.md#abstract-class-vs-interface) |
| Paytm | Android memory questions | [M27 Memory](../deep-dive/27-memory.md) |
| Meesho | SOLID principles | [Architecture Q&A](../architecture/questions.md) |
| Meesho | Dagger, why DI, alternatives, build your own | [M14 Dagger](../deep-dive/14-dagger.md) · [DI across modules](../modularization/di-across-modules.md) |
| Meesho | MVVM vs MVP, why not RxJava observables | [Patterns: MVVM · MVI · UDF](../architecture/patterns.md) |
| Meesho | Multi-module benefits, dependency/abstraction handling | [Modularization strategy](../modularization/strategy.md) |
| Meesho | `val` vs `const` | [Kotlin fundamentals](../kotlin/fundamentals.md#const-val-vs-val) |
| Meesho | `inline` keyword | [Kotlin idioms](../kotlin/idioms.md#inline-noinline-crossinline) |
| Meesho | `lateinit` vs `lazy` | [Kotlin fundamentals](../kotlin/fundamentals.md#lateinit-vs-lazy) |
| Rakuten | Does changing device/app language re-download a language split | [App Bundle — language splits](../deep-dive/36-distribution.md#language-splits-and-a-runtime-locale-change) |
| Rakuten | A→B→C→D — which activity can be `singleInstance`, and why | [Launch modes](../deep-dive/01-activity.md#the-four-launch-modes) |
| Rakuten | Can a `data class` have an empty constructor | [Kotlin fundamentals](../kotlin/fundamentals.md#can-a-data-class-have-an-empty-constructor) |
| Rakuten | `List` vs `MutableList` vs `ImmutableList` vs `PersistentList` stability in Compose | [Compose Basics — stability](../deep-dive/25-compose-basics.md#list-vs-mutablelist-vs-immutablelist-vs-persistentlist) |
| Rakuten | Recomposition scope: which functions recompose when a hoisted value changes | [Worked below ↓](#tracing-recomposition-scope) |
| Rakuten | Backing properties in Kotlin | [Kotlin fundamentals](../kotlin/fundamentals.md#backing-fields-backing-properties) |
| Rakuten | `inline` lambda with a bare `return` swallows a later call — fix it, and what `noinline` changes | [Worked below ↓](#non-local-return-swallowing-a-later-call) |
| Rakuten | Normal function vs `inline` function, why declare a function `inline` | [Kotlin idioms](../kotlin/idioms.md#inline-noinline-crossinline) |
| Rakuten | `@Inject` vs `@AssistedInject` | [M14 Dagger](../deep-dive/14-dagger.md#inject-vs-assistedinject) |
| Rakuten | Trace a recursive function using postfix `n--` as the argument | [Worked below ↓](#postfix-decrement-in-a-recursive-call) |
| Rakuten | `LiveData` vs `MediatorLiveData`; cold vs hot `Flow` | [MediatorLiveData](../deep-dive/05-viewmodel-livedata.md#b7-mediatorlivedata) · [Cold vs hot Flow](../deep-dive/13-flow.md#cold-vs-hot) |
| Rakuten | Exception handling in coroutines | [M12 Coroutines — Part F](../deep-dive/12-coroutines.md#part-f-exception-handling) |
| Rakuten | Stateful vs stateless composables | [Compose Basics — state hoisting](../deep-dive/25-compose-basics.md#part-e-state-hoisting-unidirectional-data-flow) |
| Rakuten | `launch` vs `async` — which to use for background work | [Worked below ↓](#launch-vs-async-for-background-work) |
| Rakuten | Upload 30 images, but only 5 concurrently | [Worked below ↓](#throttle-concurrent-uploads) |

!!! tip "Why most of this table is links, not answers"
    A Senior/Lead loop reuses the same fundamentals every other round asks — the differentiator
    isn't a second copy of the answer, it's not having to search for it twice. Everything above
    that already has a home stays there so it's kept in sync with the module's Interview Q&A
    section; only the genuinely new problems (mostly the DSA warm-ups) get worked out below.

---

## Worked coding problems

Each solution favors clarity and the standard-library idiom over cleverness — that's what a 20-minute
round is actually grading. State the brute force, name the complexity, then reach for the better one.

### Implement `findViewById`

**The ask:** without using the framework, implement the tree lookup `findViewById` performs. Tests
whether you understand the view tree is just a tree, not magic.

```kotlin
open class MiniView(val id: Int)

class MiniViewGroup(id: Int) : MiniView(id) {
    val children = mutableListOf<MiniView>()

    fun addView(child: MiniView) { children += child }
}

fun MiniView.findViewById(target: Int): MiniView? {
    if (id == target) return this
    if (this is MiniViewGroup) {
        for (child in children) {
            child.findViewById(target)?.let { return it }
        }
    }
    return null
}
```

Depth-first, O(V) worst case where V is the number of views — real Android does the same DFS over
`ViewGroup.mChildren`, which is why a deeply nested layout makes repeated `findViewById` calls
measurably slower (one more reason View Binding / Compose skip the lookup entirely — see
[M6 Binding](../deep-dive/06-binding.md)).

### Group mutual anagrams

Two words are mutual anagrams if their letters — sorted — are identical. Group by that signature.

```kotlin
fun groupAnagrams(words: List<String>): List<List<String>> =
    words.groupBy { it.toCharArray().sorted().joinToString("") }.values.toList()
```

O(n · k log k) for n words of max length k. A single-word anagram *check* is the same idea without
the grouping:

```kotlin
fun isAnagram(a: String, b: String): Boolean =
    a.length == b.length && a.toCharArray().sorted() == b.toCharArray().sorted()
```

`sorted()` comparison is simplest to say out loud; a 26-length count array gets you O(n) instead of
O(n log n) if the interviewer pushes on complexity.

### Square, rectangle, or neither

Given four side lengths (unordered), classify the shape they could form.

```kotlin
enum class Shape { SQUARE, RECTANGLE, NEITHER }

fun classify(sides: List<Int>): Shape {
    require(sides.size == 4)
    val (a, b, c, d) = sides.sorted()
    if (a != b || c != d) return Shape.NEITHER   // need two matching pairs
    return if (a == c) Shape.SQUARE else Shape.RECTANGLE
}
```

Sorting turns "do these four form two equal pairs" into a linear check — the trick worth saying
out loud, since the brute-force pairwise comparison is uglier and easier to get wrong.

### Delta encoding

First element as-is; every subsequent element as the difference from its predecessor.

```kotlin
fun deltaEncode(nums: List<Int>): List<Int> =
    nums.mapIndexed { i, n -> if (i == 0) n else n - nums[i - 1] }

fun deltaDecode(deltas: List<Int>): List<Int> {
    val result = mutableListOf<Int>()
    var running = 0
    for ((i, d) in deltas.withIndex()) {
        running = if (i == 0) d else running + d
        result += running
    }
    return result
}
```

O(n) both ways — the follow-up is usually "why would you do this at all," and the answer is
compressibility: monotonic or slow-changing sequences (timestamps, sensor readings) delta-encode
to mostly-small numbers that compress far better than the raw values.

### Common elements in three arrays

Arrays may contain duplicates; return elements present in all three (each once).

```kotlin
fun commonElements(a: List<Int>, b: List<Int>, c: List<Int>): List<Int> {
    val setB = b.toHashSet()
    val setC = c.toHashSet()
    return a.toHashSet().filter { it in setB && it in setC }
}
```

O(a + b + c) with hash sets. If the arrays are pre-sorted, a three-pointer sweep does it in the
same complexity with O(1) extra space instead of O(n) — worth mentioning as the follow-up trade.

### `ConcurrentModificationException`

**The ask:** why does removing from an `ArrayList` while iterating with a `for` loop throw, and how
do you fix it?

```kotlin
val list = mutableListOf(1, 2, 3, 4, 5, 6)

// Throws CME: the for-each's Iterator checks a modCount snapshot on every next(),
// and list.remove() bumps modCount without the iterator knowing.
for (n in list) {
    if (n % 2 == 0) list.remove(n)   // CRASH on next hasNext()/next() call
}
```

```kotlin
// Fix 1 — mutate through the iterator itself; it updates modCount in lockstep.
val it = list.iterator()
while (it.hasNext()) {
    if (it.next() % 2 == 0) it.remove()
}

// Fix 2 — build a new list instead of mutating in place.
val filtered = list.filterNot { it % 2 == 0 }

// Fix 3 — iterate a snapshot copy if you must mutate the original inside the loop.
for (n in list.toList()) {
    if (n % 2 == 0) list.remove(n)
}
```

The twist interviewers like: removing the **second-to-last** element sometimes *doesn't* throw,
because `hasNext()` can return `false` before the final `next()` call exposes the stale `modCount` —
a good answer names this as "CME is best-effort detection, not a guarantee," not a bug in the JDK.

### Busiest day — interval sweep

Given a list of `(checkIn, checkOut)` day pairs, find the single busiest day. This is the same shape
as the "merge intervals" family, solved with a difference array instead of merging.

```kotlin
fun busiestDay(bookings: List<Pair<Int, Int>>): Int {
    val delta = sortedMapOf<Int, Int>()
    for ((checkIn, checkOut) in bookings) {
        delta[checkIn] = (delta[checkIn] ?: 0) + 1
        delta[checkOut] = (delta[checkOut] ?: 0) - 1   // guest leaves on checkout day
    }
    var running = 0
    var best = 0
    var bestDay = -1
    for ((day, change) in delta) {
        running += change
        if (running > best) { best = running; bestDay = day }
    }
    return bestDay
}
```

O(n log n) for the sorted sweep over distinct days instead of O(n · range) for a naive per-day
counter — the thing to call out is exactly which day a checkout "frees" (does the guest still
occupy the room on the checkout day itself?), since that boundary condition is what the
interviewer is actually testing.

### Elements repeated more than N times

```kotlin
fun repeatedMoreThan(nums: List<Int>, times: Int): List<Int> =
    nums.groupingBy { it }.eachCount()
        .filterValues { it > times }
        .keys.toList()
```

`groupingBy { }.eachCount()` is the idiomatic Kotlin replacement for "build a `HashMap<Int, Int>`
and increment manually" — same O(n) complexity, no manual bookkeeping.

### Sentiment from word lists

Given positive-word and negative-word sets, score a free-text review.

```kotlin
enum class Sentiment { POSITIVE, NEGATIVE, NEUTRAL }

fun classify(review: String, positives: Set<String>, negatives: Set<String>): Sentiment {
    val words = review.lowercase().split(Regex("\\W+")).filter { it.isNotBlank() }
    val score = words.sumOf { w -> if (w in positives) 1 else if (w in negatives) -1 else 0 }
    return when {
        score > 0 -> Sentiment.POSITIVE
        score < 0 -> Sentiment.NEGATIVE
        else -> Sentiment.NEUTRAL
    }
}
```

O(n) in review length with O(1) set lookups. The interesting follow-up is what happens with
negation ("not good") — a pure bag-of-words scorer gets this wrong, which is the right moment to
say "this is why production sentiment uses a model, not a word list" rather than over-engineer the
toy version.

### Linked list cycle

Floyd's tortoise-and-hare — two pointers, no extra memory.

```kotlin
class Node(val value: Int) { var next: Node? = null }

fun hasCycle(head: Node?): Boolean {
    var slow = head
    var fast = head
    while (fast?.next != null) {
        slow = slow?.next
        fast = fast.next?.next
        if (slow === fast) return true   // referential identity — same node object
    }
    return false
}
```

O(n) time, O(1) space — the reason this beats a `HashSet<Node>` of visited nodes (also O(n) time
but O(n) space). Note `===`, not `==`: you're asking "is this the same node," not "do these nodes
have equal content" (see the `==` vs `===` section in [Kotlin fundamentals](../kotlin/fundamentals.md)).

### Palindrome linked list

Find the middle, reverse the second half in place, compare both halves.

```kotlin
fun isPalindrome(head: Node?): Boolean {
    if (head == null || head.next == null) return true

    var slow = head
    var fast = head
    while (fast?.next != null) {
        slow = slow!!.next
        fast = fast.next?.next
    }

    var prev: Node? = null
    var curr = slow
    while (curr != null) {
        val next = curr.next
        curr.next = prev
        prev = curr
        curr = next
    }

    var left = head
    var right = prev
    while (right != null) {
        if (left!!.value != right.value) return false
        left = left.next
        right = right.next
    }
    return true
}
```

O(n) time, O(1) extra space — the in-place reversal is the whole trick over the O(n)-space "copy
into an array and two-pointer compare" answer; mention you're mutating the list's structure
(restorable by reversing again) as the tradeoff for the space win.

### Minimum meeting rooms

Given `[start, end)` intervals, find the minimum number of rooms needed to hold all meetings
concurrently — the same reasoning that would size a "busiest concurrent bookings" widget.

```kotlin
fun minMeetingRooms(intervals: List<IntArray>): Int {
    val starts = intervals.map { it[0] }.sorted()
    val ends = intervals.map { it[1] }.sorted()

    var rooms = 0
    var maxRooms = 0
    var s = 0
    var e = 0
    while (s < starts.size) {
        if (starts[s] < ends[e]) {
            rooms++; s++
        } else {
            rooms--; e++
        }
        maxRooms = maxOf(maxRooms, rooms)
    }
    return maxRooms
}
```

O(n log n) for the two sorts + linear sweep, O(n) space for the two arrays — beats the O(n²)
pairwise-overlap check and is the same difference-array idea as [busiest day](#busiest-day-interval-sweep)
above, just phrased as a running counter instead of a map.

### String rotation

Is `b` a rotation of `a`, and if so, by how many positions?

```kotlin
fun isRotation(a: String, b: String): Boolean =
    a.length == b.length && (a + a).contains(b)

fun rotationCount(original: String, rotated: String): Int {
    if (!isRotation(original, rotated)) return -1
    return (original + original).indexOf(rotated)   // left-rotation offset
}
```

The `s + s` trick is the whole answer: every rotation of `a` is a substring of `a + a`, so the
problem collapses to a single substring search — O(n) with Kotlin's built-in `indexOf` (KMP under
the hood) instead of an O(n²) manual character-shift comparison.

### Pangram check

```kotlin
fun isPangram(s: String): Boolean {
    val lower = s.lowercase()
    return ('a'..'z').all { it in lower }
}
```

O(n + 26) — reads clean with `all { }`; a `BitSet`/`Int` bitmask of seen letters is the O(n)
one-pass alternative if the interviewer wants to avoid the `in` scan repeating over `lower`.

### Sorting an array — algorithms and complexity

The generic "sort this array, what's the time complexity" warm-up is really asking you to place a handful of algorithms on the same table, not to hand-roll one from scratch:

| Algorithm | Time (avg) | Time (worst) | Space | Stable? | Notes |
|---|---|---|---|---|---|
| Bubble / Selection sort | O(n²) | O(n²) | O(1) | Bubble: yes · Selection: no | Never the real answer; only correct as "the naive baseline before I optimize" |
| Insertion sort | O(n²) | O(n²) | O(1) | Yes | Actually good for small/near-sorted input — this is why hybrid sorts fall back to it below a size threshold |
| Merge sort | O(n log n) | O(n log n) | O(n) | Yes | Guaranteed n log n, but not in-place — the extra O(n) buffer is the tradeoff |
| Quicksort | O(n log n) | **O(n²)** | O(log n) | No | Fastest in practice (cache-friendly, in-place) but worst case degrades on already-sorted/adversarial input without a good pivot strategy (median-of-three, random pivot) |
| Kotlin's `sort()`/`sorted()` | O(n log n) | O(n log n) | O(n) worst-case | Yes | **Timsort** — a hybrid: merge sort overall, insertion sort on small runs, exploits already-sorted subsequences. This is what you actually call; the four above are what you explain when asked why |

```kotlin
// What you'd actually write in production — Kotlin's stdlib is Timsort under the hood.
val sorted = nums.sorted()          // new List, O(n log n)
nums.sort()                          // in-place on a MutableList/Array, same algorithm

// A from-scratch quicksort, if asked to implement one:
fun quicksort(arr: IntArray, lo: Int = 0, hi: Int = arr.lastIndex) {
    if (lo >= hi) return
    val pivot = arr[(lo + hi) / 2]
    var i = lo; var j = hi
    while (i <= j) {
        while (arr[i] < pivot) i++
        while (arr[j] > pivot) j--
        if (i <= j) { arr[i] = arr[j].also { arr[j] = arr[i] }; i++; j-- }
    }
    quicksort(arr, lo, j)
    quicksort(arr, i, hi)
}
```

The complexity answer that actually scores: name the naive O(n²) family, then Timsort/merge/quick at O(n log n), state *why* comparison-based sorting can't beat O(n log n) (the decision-tree lower bound), and mention that if the input is bounded integers (not arbitrary comparable values), counting sort/radix sort break that bound at O(n + k) — which is exactly the trick behind [Sort 0s, 1s, 2s](#sort-0s-1s-2s-dutch-national-flag) below, a 3-value special case of counting sort done in one pass.

### Sort 0s, 1s, 2s (Dutch national flag)

Single pass, three pointers, no counting sort / extra array.

```kotlin
fun sortColors(nums: IntArray) {
    var low = 0
    var mid = 0
    var high = nums.lastIndex
    while (mid <= high) {
        when (nums[mid]) {
            0 -> { nums[low] = nums[mid].also { nums[mid] = nums[low] }; low++; mid++ }
            1 -> mid++
            2 -> { nums[high] = nums[mid].also { nums[mid] = nums[high] }; high-- }
        }
    }
}
```

O(n) time, O(1) space, single pass — the reason it beats "count zeros/ones/twos then overwrite"
(also O(n) but two passes) is that it's the answer the interviewer actually wants when they say
"can you do it in one pass."

### Remove duplicates from an array in Kotlin

**The ask:** Remove duplicate values from a given array in Kotlin while keeping the original order.

```kotlin
fun <T> Array<T>.removeDuplicates(): Array<T> {
    return this.distinct().toTypedArray()
}

// Or manual Set-based approach for primitive arrays:
fun IntArray.removeDuplicates(): IntArray {
    val seen = LinkedHashSet<Int>()
    for (num in this) {
        seen.add(num)
    }
    return seen.toIntArray()
}
```

*Complexity:* O(n) time complexity and O(n) space complexity. Using `distinct()` or `LinkedHashSet` retains the insertion order.

### Write a function (Higher-Order Function) that returns a function

**The ask:** Write a higher-order function that generates and returns another function, illustrating closures.

```kotlin
fun multiplier(factor: Int): (Int) -> Int {
    return { number -> number * factor } // returns a function enclosing 'factor'
}

fun main() {
    val double = multiplier(2)
    val triple = multiplier(3)
    println(double(5)) // prints 10
    println(triple(5)) // prints 15
}
```

*Interview angle:* This demonstrates **closures** in Kotlin. The returned lambda captures the parameter `factor` from its enclosing scope and retains it even after `multiplier` completes execution.

### Postfix decrement in a recursive call

**The ask (as given, Java-flavored):**

```java
void s(int n) {
    if (n == 0) { System.out.print("text"); return; }
    System.out.print(n);
    s(n--);
}
s(3);
```

!!! note "This doesn't compile as literal Kotlin"
    Kotlin function parameters are implicitly `val` — `n--` on a parameter is a compile error (`val cannot be reassigned`). The trap only exists in a language (Java, or a Kotlin version with `n` copied into a local `var`) where the parameter is mutable. Worth naming out loud if an interviewer hands you this in Kotlin syntax — it's really testing postfix-decrement evaluation order, not Kotlin specifically.

**The trace:** `s(3)` prints `3`, not `0`. Every recursive call sees `n == 3` again, so the base case is never reached — this is **infinite recursion**, terminating only in a `StackOverflowError`. The reason: `n--` is a **postfix** decrement — the *value of the expression* is `n` **before** decrementing; only the *side effect* (writing the decremented value back to `n`) happens after. That side effect lands in the current call's local `n`, which is about to be discarded — the value actually passed to the next `s(...)` call is the pre-decrement value, `3`, every single time.

```
s(3): n=3 → print "3" → argument n-- evaluates to 3 (then this frame's n becomes 2, irrelevant)
  s(3): n=3 → print "3" → same thing again
    s(3): n=3 → print "3" → ...
      ... forever → StackOverflowError
```

**The fix:** use **prefix** decrement, `s(--n)` — its expression value *is* the already-decremented value, so each call actually receives a smaller `n`:

```java
void s(int n) {
    if (n == 0) { System.out.print("text"); return; }
    System.out.print(n);
    s(--n);   // prints the DECREMENTED value → 3, 2, 1, "text"
}
```

Complexity aside, this is a "do you actually understand postfix vs prefix evaluation order, not just the syntax" question — a very cheap trap that turns a 4-line function into an unbounded stack growth bug.

### Throttle concurrent uploads

**The ask:** 30 images queued for upload; the server (or your own bandwidth budget) should only see **5 in flight at once**.

```kotlin
suspend fun uploadAll(images: List<Uri>, maxConcurrent: Int = 5): List<UploadResult> = coroutineScope {
    val semaphore = Semaphore(maxConcurrent)
    images.map { uri ->
        async {
            semaphore.withPermit { uploadOne(uri) }   // suspends here until a permit is free
        }
    }.awaitAll()   // fails fast: one upload throwing cancels the rest
}
```

All 30 `async` coroutines are **launched** immediately — that's cheap, they're just suspended coroutine objects, not threads. Each one blocks at `semaphore.withPermit { }` until one of the 5 permits frees up, so at most 5 are actually inside `uploadOne` at any moment; the instant one finishes, the next queued coroutine acquires the freed permit and starts. `awaitAll()` gives fail-fast structured-concurrency behavior: if any upload throws, the rest are cancelled instead of silently finishing after you've already reported failure.

!!! note "Why `Semaphore`, not `chunked(5)`"
    A naive `images.chunked(5).forEach { chunk -> chunk.map { async { uploadOne(it) } }.awaitAll() }` also caps concurrency at 5, but it **stalls at each chunk boundary** — it waits for the slowest of each batch of 5 before starting the next 5, even if 4 of them finished early. `Semaphore` keeps the pipeline saturated: a new upload starts the instant *any* slot frees, not the instant the whole batch does. Same O(n) work, strictly better throughput under uneven upload times.

    `Dispatchers.IO.limitedParallelism(5)` (see [M12 Coroutines — Dispatchers](../deep-dive/12-coroutines.md#custom-dispatchers-and-limitedparallelism)) is the *dispatcher-level* version of the same idea — reach for it when you're bounding thread/dispatcher usage; reach for `Semaphore` when you're bounding an **application-level** resource limit (e.g. "the server accepts 5 concurrent uploads per client") that's independent of which dispatcher runs the work.

### Tracing recomposition scope

**The ask:** `A()` calls `B()`; `B()` calls `C()` and `D()`. A `userName` value is read directly in `A` and threaded down as a parameter through `B` to `D` (`C` never sees it). When `userName` changes, which functions recompose?

```kotlin
@Composable
fun A(userName: String) {
    Text("Hello $userName")   // A reads userName directly → A is invalidated on change
    B(userName)
}

@Composable
fun B(userName: String) {
    C()             // no dependency on userName
    D(userName)
}

@Composable
fun C() { /* ... */ }

@Composable
fun D(userName: String) { Text(userName) }
```

**Answer: `A`, `B`, and `D` recompose. `C` is skipped.**

- `A` reads `userName` in its own body, so `A`'s recompose scope is directly invalidated when it changes — `A` always re-runs.
- Because `A` re-runs, it re-invokes `B(userName)` with the new value. `B`'s parameter is a `String` (stable) but its *value* changed, so `B` is **not** equal to last time → skipping doesn't apply → `B` recomposes.
- `B` re-invokes `C()` — `C` takes no parameters affected by the change, and (assuming it's otherwise stable/skippable) its call site sees no changed, unstable input → the runtime calls `skipToGroupEnd()` and **`C`'s body never re-runs**, cutting off recomposition to its entire subtree.
- `B` also re-invokes `D(userName)` — the parameter changed, so `D` is not skipped and recomposes.

The general rule this exercises: recomposition isn't "the whole subtree re-runs because a parent changed" — it propagates only through scopes whose actual inputs changed (or that directly read the invalidated state); any composable in between whose call-site arguments are unchanged and stable is skipped, and skipping a scope skips everything *inside* it too. See [Compose Basics — Recomposition](../deep-dive/25-compose-basics.md#part-c-recomposition) and [Compose Runtime Internals — Recomposition & skipping](../deep-dive/41-compose-internals.md#4-recomposition-skipping) for the mechanics behind why `C` is never even entered.

### Non-local return swallowing a later call

**The ask:**

```kotlin
inline fun message(a: () -> Unit) { a.invoke() }

fun main() {
    message { print(1); return }
    message { print(2) }
}
```

**Output:** `1` — and nothing else. `main()` returns immediately after printing `1`; the second `message { print(2) }` call is never reached.

**Why:** `message` is `inline`, so its body — and the lambda passed to it — is spliced directly into the call site. The bare `return` inside the first lambda is therefore a **non-local return**: because the lambda physically lives inside `main()` after inlining, `return` exits `main()` itself, not just the lambda. That's exactly the mechanism [Kotlin idioms — `inline`/`noinline`/`crossinline`](../kotlin/idioms.md#inline-noinline-crossinline) covers: non-local return is only possible because the lambda is inlined.

**To print `2` as well**, use a **labeled return** to make the first `return` local to the lambda instead of non-local to `main`:

```kotlin
fun main() {
    message { print(1); return@message }   // local return — exits only this lambda
    message { print(2) }                   // now reached: prints "2"
}
```

`return@message` stops the first lambda at that point (same effect as falling off the end of the block) without unwinding `main`, so execution continues to the next statement.

**What `noinline` changes:** marking the parameter `noinline` —

```kotlin
inline fun message(noinline a: () -> Unit) { a.invoke() }
```

— stops the lambda from being physically spliced into the call site; it becomes a real allocated `Function0` object, the same as a parameter to a non-inline function. Because it's no longer inlined, a **bare `return` inside it is no longer legal** — the compiler rejects it (`'return' is not allowed here`), forcing you to either use `return@message` or restructure. So `noinline` doesn't fix the output by itself; it removes the *capability* that caused the bug in the first place (non-local return), turning a silent logic error into a compile-time error that points you at the labeled-return fix.

### `launch` vs `async` for background work

For a plain "go do this in the background, I don't need the result" task — an analytics ping, a DB write, a fire-and-forget network call — prefer **`launch`**. It's the simpler builder (`Job`, no result to manage), and an uncaught exception surfaces immediately through structured concurrency to the parent/`CoroutineExceptionHandler`, which is the behavior you want for work nobody is waiting on: a failure should be visible, not silently discarded.

Reach for **`async`** only when you need a *result* and intend to `await()` it — typically two or more independent pieces of work you want running concurrently before combining their results. The trap: `async` **defers** its exception to `await()` — if you never call `await()` on a `Deferred`, a thrown exception can sit unobserved (and, depending on the job hierarchy, may not surface until something else notices). Using `async` for work you don't plan to await is a common anti-pattern; it buys you nothing over `launch` and weakens error visibility.

See [M12 Coroutines — Part F: Exception handling](../deep-dive/12-coroutines.md#part-f-exception-handling) for the full `launch` vs `async` exception-surfacing contract.

---

## Disk-based cache constraints

Spotify's version of the [client-side caching layer](../system-design/questions.md#6-design-a-client-side-caching-layer)
prompt adds concrete constraints worth naming explicitly if asked:

- **Platform-independent, opaque values (`ByteArray`), fixed 32-byte keys** → a simple two-file
  design (append-only data log + an index mapping key → `(offset, length)`) generalizes across
  platforms better than reusing a JVM-specific serialization format; keep the cache's on-disk
  format free of Android-specific types.
- **Persistent, 100k+ objects, configurable 10 MB–1 GB** → an index kept fully in memory (32-byte
  key → offset/length is small per entry) with the blobs on disk; evict by LRU using an
  in-memory doubly-linked list + hash map (the same structure as an [LRU cache](../system-design/questions.md#6-design-a-client-side-caching-layer)),
  persisting the LRU order periodically so a crash doesn't require a full rebuild.
- **Secure** → encrypt values at rest (AES-GCM, key from Android Keystore) — the index (keys +
  offsets) can stay unencrypted since keys are opaque hashes, but never write plaintext values to
  disk. See [M24 Security](../deep-dive/24-security.md) for the Keystore-backed pattern.
- **Opaque** → the cache must not need to understand what's stored — no per-type serialization
  logic in the cache layer itself; callers own encode/decode.

This is a good prompt to state the two axes explicitly (LRU eviction *and* size-bound eviction can
disagree — a cache under its object-count limit can still be over its byte-size limit) before
diving into data structures.
