  # Coding & Algorithms — Lead .NET Interview Prep

> Comprehensive C# solutions with full explanations for the most common algorithm problems in Lead .NET Software Engineer interviews. Each problem includes approach walkthrough, complexity analysis, and complete working code.

---

## Table of Contents

| # | Problem | Key Technique | Time Complexity |
|---|---------|--------------|-----------------|
| 1 | [Two Sum](#1-two-sum) | HashMap | O(n) |
| 2 | [Product of Array Except Self](#2-product-of-array-except-self) | Left / Right Pass | O(n) |
| 3 | [Merge Intervals](#3-merge-intervals) | Sort + Sweep | O(n log n) |
| 4 | [Longest Substring Without Repeating Characters](#4-longest-substring-without-repeating-characters) | Sliding Window | O(n) |
| 5 | [LRU Cache](#5-lru-cache) | Dictionary + DLL | O(1) |
| 6 | [Top K Frequent Elements](#6-top-k-frequent-elements) | Min-Heap | O(n log k) |
| 7 | [Binary Search](#7-binary-search) | Binary Search | O(log n) |
| 8 | [BFS — Breadth-First Search](#8-bfs--breadth-first-search) | Queue | O(V + E) |
| 9 | [DFS — Depth-First Search](#9-dfs--depth-first-search) | Stack / Recursion | O(V + E) |
| 10 | [Tree Traversal](#10-tree-traversal) | Recursive / Iterative | O(n) |
| 11 | [Valid Parentheses](#11-valid-parentheses) | Stack | O(n) |
| 12 | [Maximum Subarray — Kadane's](#12-maximum-subarray--kadanes-algorithm) | Kadane's DP | O(n) |
| 13 | [Coin Change](#13-coin-change-dynamic-programming) | Bottom-up DP | O(n × amount) |
| 14 | [Climbing Stairs](#14-climbing-stairs-dp--fibonacci) | Fibonacci DP | O(n) |
| 15 | [Find All Anagrams](#15-find-all-anagrams-in-a-string) | Sliding Window | O(n) |
| B1 | [Algorithm Patterns Summary](#bonus-algorithm-patterns-summary) | Reference | — |
| B2 | [C# Specific Tips](#bonus-c-specific-tips) | Reference | — |

---

## 1. Two Sum

### Problem Statement

Given an array of integers `nums` and an integer `target`, return the **indices** of the two numbers that add up to `target`. Exactly one solution always exists; you may not use the same index twice.

### Examples

```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]       (2 + 7 = 9)

Input:  nums = [3, 2, 4], target = 6
Output: [1, 2]       (2 + 4 = 6)

Input:  nums = [3, 3], target = 6
Output: [0, 1]
```

### Approach

**Brute Force — O(n²):** Try every pair `(i, j)` where `j > i`. Simple but slow.

**HashMap — O(n):** In a single pass, store each number and its index in a dictionary. For each element `nums[i]`, compute `complement = target - nums[i]`. If the complement is already in the dictionary, you have the answer. Otherwise, store the current number. Because you only look backward (at already-stored elements), the same index is never used twice.

Step-by-step for `[2, 7, 11, 15], target = 9`:
```
i=0: complement = 9 - 2 = 7  → not in dict → store {2:0}
i=1: complement = 9 - 7 = 2  → found at index 0 → return [0, 1]
```

### Complexity

| Approach | Time | Space |
|---|---|---|
| Brute Force | O(n²) | O(1) |
| HashMap (optimal) | O(n) | O(n) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Two Sum — find two indices whose values sum to target.
/// </summary>
public class Solution01_TwoSum
{
    // ── Approach 1: Brute Force — O(n²) time, O(1) space ──────────────
    public int[] TwoSumBruteForce(int[] nums, int target)
    {
        for (int i = 0; i < nums.Length; i++)
        {
            for (int j = i + 1; j < nums.Length; j++)
            {
                if (nums[i] + nums[j] == target)
                    return new[] { i, j };
            }
        }
        return Array.Empty<int>(); // guaranteed solution exists per problem
    }

    // ── Approach 2: HashMap — O(n) time, O(n) space ───────────────────
    public int[] TwoSumHashMap(int[] nums, int target)
    {
        // key = number value, value = its array index
        var seen = new Dictionary<int, int>();

        for (int i = 0; i < nums.Length; i++)
        {
            int complement = target - nums[i];

            // If the complement was stored in a previous iteration, done
            if (seen.TryGetValue(complement, out int prevIndex))
                return new[] { prevIndex, i };

            // Store current number so future iterations can find it
            seen[nums[i]] = i;
        }

        return Array.Empty<int>();
    }
}
```

### Follow-up Questions

- What if there can be zero or multiple solutions? (change return type to `List<int[]>`)
- What if the array is sorted? (use two-pointer O(n) time, O(1) space)
- What if we need the values instead of indices? (trivial with the HashMap approach)
- Can you solve it with O(1) space? (two-pointer on sorted copy — but then you lose original indices)

---

## 2. Product of Array Except Self

### Problem Statement

Given an integer array `nums`, return an array `result` where `result[i]` equals the product of all elements **except** `nums[i]`. Must run in O(n) time **without using division**.

### Examples

```
Input:  [1, 2, 3, 4]
Output: [24, 12, 8, 6]
  (result[0] = 2*3*4=24, result[1] = 1*3*4=12, ...)

Input:  [-1, 1, 0, -3, 3]
Output: [0, 0, 9, 0, 0]
```

### Approach

**Key insight:** For each index `i`, the product except self = (product of all elements LEFT of i) × (product of all elements RIGHT of i).

**Two-pass algorithm:**
1. **Left pass** — fill `result[i]` with the product of all elements to the *left* of `i`. `result[0] = 1` (no elements to the left).
2. **Right pass** — multiply `result[i]` by a running right-product. Traverse from right to left; `rightProduct` accumulates the product of all elements to the *right* of `i`.

Example with `[1, 2, 3, 4]`:
```
After left pass:  result = [1, 1, 2, 6]
                            ↑  ↑  ↑  ↑
                           ()  1  1*2 1*2*3

rightProduct starts at 1:
i=3: result[3] = 6 * 1 = 6,  rightProduct = 1*4 = 4
i=2: result[2] = 2 * 4 = 8,  rightProduct = 4*3 = 12
i=1: result[1] = 1 * 12 = 12, rightProduct = 12*2 = 24
i=0: result[0] = 1 * 24 = 24

Final: [24, 12, 8, 6]  ✓
```

### Complexity

| | Time | Space |
|---|---|---|
| Two-pass | O(n) | O(1) extra (output array not counted) |

### C# Solution

```csharp
using System;

/// <summary>
/// Product of Array Except Self — O(n) time, O(1) extra space.
/// No division allowed.
/// </summary>
public class Solution02_ProductExceptSelf
{
    public int[] ProductExceptSelf(int[] nums)
    {
        int n = nums.Length;
        int[] result = new int[n]; // output array (not counted as extra space per problem)

        // ── Left Pass ────────────────────────────────────────────────────
        // result[i] = product of everything to the LEFT of index i
        result[0] = 1; // nothing to the left of index 0
        for (int i = 1; i < n; i++)
        {
            result[i] = result[i - 1] * nums[i - 1];
        }

        // ── Right Pass ───────────────────────────────────────────────────
        // Multiply result[i] by product of everything to the RIGHT of index i
        int rightProduct = 1; // running product from the right
        for (int i = n - 1; i >= 0; i--)
        {
            result[i] *= rightProduct;   // combine left and right products
            rightProduct *= nums[i];     // extend running right product
        }

        return result;
    }
}
```

### Follow-up Questions

- How would you handle if division were allowed? (compute total product, divide by `nums[i]` — but breaks on zeros)
- What if the array contains zeros? (the two-pass approach handles zeros correctly without any special casing)
- Can you generalize to "product of all except a window of size k"?

---

## 3. Merge Intervals

### Problem Statement

Given an array of intervals `[start, end]`, merge all overlapping intervals and return the non-overlapping result.

### Examples

```
Input:  [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]

Input:  [[1,4],[4,5]]
Output: [[1,5]]       (touching intervals are considered overlapping)

Input:  [[1,4],[2,3]]
Output: [[1,4]]       (nested interval)
```

### Approach

1. **Sort** intervals by start time — O(n log n).
2. Initialize the result list with the first interval.
3. For each subsequent interval, compare its start against the **end of the last merged interval**:
   - If `interval.start <= last.end`: overlapping — extend `last.end = max(last.end, interval.end)`.
   - Otherwise: no overlap — append as a new interval.

```
Sorted: [1,3], [2,6], [8,10], [15,18]

Start: merged = [[1,3]]
[2,6] → 2 <= 3 → extend: [[1,6]]
[8,10] → 8 > 6 → append: [[1,6],[8,10]]
[15,18] → 15 > 10 → append: [[1,6],[8,10],[15,18]]
```

### Complexity

| | Time | Space |
|---|---|---|
| Sort + sweep | O(n log n) | O(n) output |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Merge Intervals — sort by start, then greedily merge overlaps.
/// </summary>
public class Solution03_MergeIntervals
{
    public int[][] Merge(int[][] intervals)
    {
        if (intervals.Length <= 1) return intervals;

        // Sort by start time using a custom comparison
        Array.Sort(intervals, (a, b) => a[0].CompareTo(b[0]));

        var merged = new List<int[]>();
        merged.Add(intervals[0]); // seed with the first interval

        for (int i = 1; i < intervals.Length; i++)
        {
            int[] current = intervals[i];
            int[] last    = merged[^1]; // index-from-end operator (C# 8+)

            if (current[0] <= last[1])
            {
                // Overlapping or touching: extend the end of the last interval
                last[1] = Math.Max(last[1], current[1]);
            }
            else
            {
                // Gap: add as a new separate interval
                merged.Add(current);
            }
        }

        return merged.ToArray();
    }
}
```

### Follow-up Questions

- Insert Interval: given sorted non-overlapping intervals + one new interval, merge efficiently (O(n) — no sort needed).
- Meeting Rooms: given meeting intervals, can one person attend all? (check if any overlaps exist)
- Meeting Rooms II: how many rooms needed? (use min-heap on end times)

---

## 4. Longest Substring Without Repeating Characters

### Problem Statement

Given string `s`, find the length of the **longest substring** without repeating characters.

### Examples

```
Input:  "abcabcbb"
Output: 3             ("abc")

Input:  "bbbbb"
Output: 1             ("b")

Input:  "pwwkew"
Output: 3             ("wke")

Input:  ""
Output: 0
```

### Approach

**Sliding Window with HashSet:**
- Maintain a window `[left, right]` that contains only unique characters.
- Expand `right` by adding `s[right]` to the window.
- If `s[right]` already exists in the window, shrink from `left` until the duplicate is removed.
- Track maximum window size.

**Optimized with Dictionary (O(n) guaranteed):**
- Store each character's *last seen index* in a dictionary.
- When a duplicate is found, jump `left` directly past the previous occurrence — no inner while loop needed.

### Complexity

| Approach | Time | Space |
|---|---|---|
| HashSet sliding window | O(n) amortized | O(min(n, charset)) |
| Dictionary (jump) | O(n) strict | O(min(n, charset)) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Longest Substring Without Repeating Characters — sliding window.
/// </summary>
public class Solution04_LongestSubstring
{
    // ── Approach 1: HashSet sliding window ───────────────────────────────
    public int LengthOfLongestSubstring(string s)
    {
        var charSet = new HashSet<char>(); // chars currently in the window
        int left   = 0;
        int maxLen = 0;

        for (int right = 0; right < s.Length; right++)
        {
            // Shrink from left until duplicate s[right] is removed
            while (charSet.Contains(s[right]))
            {
                charSet.Remove(s[left]);
                left++;
            }

            charSet.Add(s[right]);
            maxLen = Math.Max(maxLen, right - left + 1);
        }

        return maxLen;
    }

    // ── Approach 2: Dictionary — jump left pointer directly ─────────────
    public int LengthOfLongestSubstringOptimized(string s)
    {
        var lastSeen = new Dictionary<char, int>(); // char → last index seen
        int left   = 0;
        int maxLen = 0;

        for (int right = 0; right < s.Length; right++)
        {
            // If char was seen inside the current window, jump left past it
            if (lastSeen.TryGetValue(s[right], out int prevIdx) && prevIdx >= left)
                left = prevIdx + 1;

            lastSeen[s[right]] = right; // update last seen index
            maxLen = Math.Max(maxLen, right - left + 1);
        }

        return maxLen;
    }
}
```

### Follow-up Questions

- What if you need to return the actual substring, not just its length? (track the start index when `maxLen` is updated)
- Longest Substring with At Most K Distinct Characters? (use Dictionary with frequency count; shrink when `dict.Count > k`)
- Minimum Window Substring? (classic sliding window — track how many distinct chars are satisfied)

---

## 5. LRU Cache

### Problem Statement

Design a data structure for a **Least Recently Used (LRU) Cache** supporting:
- `get(key)` — return value if key exists, else -1. Marks key as recently used.
- `put(key, value)` — insert or update. If capacity is exceeded, evict the **least recently used** key first.

Both operations must run in **O(1)** time.

### Examples

```
LRUCache(2)         // capacity = 2
put(1, 1)           // cache = {1=1}
put(2, 2)           // cache = {1=1, 2=2}
get(1) → 1          // cache = {2=2, 1=1}  (1 is now MRU)
put(3, 3)           // evict 2 (LRU), cache = {1=1, 3=3}
get(2) → -1         // 2 was evicted
put(4, 4)           // evict 1 (LRU), cache = {3=3, 4=4}
get(1) → -1
get(3) → 3
get(4) → 4
```

### Approach

**Dictionary + Doubly Linked List:**
- `Dictionary<int, Node>` provides O(1) lookup by key.
- **Doubly linked list** maintains usage order: head = most recently used, tail = least recently used.
- Use **dummy head and tail sentinel nodes** to simplify edge cases (no null checks on boundary operations).

Operations:
- `get(key)`: look up in dictionary, move node to head → O(1).
- `put(key, value)`: if exists, update + move to head; if new, add at head. If over capacity, remove the node just before the tail (LRU) and delete from dictionary → O(1).

```
State after put(1,1) and put(2,2):
HEAD ↔ [key=2] ↔ [key=1] ↔ TAIL
        MRU                  LRU

After get(1):
HEAD ↔ [key=1] ↔ [key=2] ↔ TAIL
```

### Complexity

| Operation | Time | Space |
|---|---|---|
| `get` | O(1) | — |
| `put` | O(1) | — |
| Overall | — | O(capacity) |

### C# Solution

```csharp
using System.Collections.Generic;

/// <summary>
/// LRU Cache — O(1) get and put using Dictionary + Doubly Linked List.
/// Sentinel head/tail nodes eliminate null-check edge cases.
/// </summary>
public class LRUCache
{
    // ── Doubly Linked List Node ──────────────────────────────────────────
    private class DllNode
    {
        public int     Key;
        public int     Value;
        public DllNode Prev;
        public DllNode Next;

        public DllNode(int key = 0, int value = 0)
        {
            Key   = key;
            Value = value;
        }
    }

    private readonly int                    _capacity;
    private readonly Dictionary<int, DllNode> _cache; // O(1) key → node lookup

    // Dummy sentinels: _head.Next = MRU node, _tail.Prev = LRU node
    private readonly DllNode _head;
    private readonly DllNode _tail;

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _cache    = new Dictionary<int, DllNode>(capacity);

        _head      = new DllNode();
        _tail      = new DllNode();
        _head.Next = _tail;
        _tail.Prev = _head;
    }

    // ── Get: O(1) ────────────────────────────────────────────────────────
    public int Get(int key)
    {
        if (!_cache.TryGetValue(key, out DllNode node))
            return -1; // key not in cache

        MoveToFront(node); // mark as most recently used
        return node.Value;
    }

    // ── Put: O(1) ────────────────────────────────────────────────────────
    public void Put(int key, int value)
    {
        if (_cache.TryGetValue(key, out DllNode existing))
        {
            // Key exists: update value and move to front
            existing.Value = value;
            MoveToFront(existing);
        }
        else
        {
            // New key: evict LRU if at capacity
            if (_cache.Count == _capacity)
            {
                DllNode lru = _tail.Prev; // node just before tail = LRU
                RemoveNode(lru);
                _cache.Remove(lru.Key);
            }

            // Insert new node at front (MRU position)
            var newNode = new DllNode(key, value);
            AddToFront(newNode);
            _cache[key] = newNode;
        }
    }

    // ── Helpers ──────────────────────────────────────────────────────────

    private void RemoveNode(DllNode node)
    {
        // Wire node's neighbors together, bypassing node
        node.Prev.Next = node.Next;
        node.Next.Prev = node.Prev;
    }

    private void AddToFront(DllNode node)
    {
        // Insert node between _head and _head.Next
        node.Next      = _head.Next;
        node.Prev      = _head;
        _head.Next.Prev = node;
        _head.Next     = node;
    }

    private void MoveToFront(DllNode node)
    {
        RemoveNode(node); // unlink from current position
        AddToFront(node); // re-link at head
    }
}
```

### Follow-up Questions

- LFU Cache (Least Frequently Used)? (more complex — need frequency buckets + doubly linked list per bucket)
- Thread-safe LRU Cache? (use `lock` or `ReaderWriterLockSlim`)
- Can you implement with `LinkedList<T>` built-in? (yes, but it is slower due to node allocation overhead on every AddFirst/Remove)
- How does `OrderedDictionary` compare? (not O(1) for all operations in all .NET versions)

---

## 6. Top K Frequent Elements

### Problem Statement

Given an integer array `nums` and integer `k`, return the `k` most frequent elements. The answer can be in any order.

### Examples

```
Input:  nums = [1,1,1,2,2,3], k = 2
Output: [1, 2]

Input:  nums = [1], k = 1
Output: [1]
```

### Approach

**Approach 1 — Sort by Frequency: O(n log n)**
Count frequencies, sort the dictionary by value descending, take top k.

**Approach 2 — Min-Heap: O(n log k)**
Count frequencies with a Dictionary. Use a min-heap (PriorityQueue) of size k. For each unique element, push to heap. If heap size exceeds k, pop the minimum (least frequent). The heap always holds the k most frequent elements.

**Approach 3 — Bucket Sort: O(n)**
Since the maximum frequency is at most n, create n+1 buckets where `bucket[freq]` contains numbers with that frequency. Iterate from the highest frequency bucket down to collect k elements.

### Complexity

| Approach | Time | Space |
|---|---|---|
| Sort by frequency | O(n log n) | O(n) |
| Min-Heap (optimal balance) | O(n log k) | O(n + k) |
| Bucket Sort (optimal time) | O(n) | O(n) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Top K Frequent Elements.
/// Approach A: Min-Heap O(n log k)
/// Approach B: Bucket Sort O(n)
/// </summary>
public class Solution06_TopKFrequent
{
    // ── Approach A: Min-Heap — O(n log k) ───────────────────────────────
    public int[] TopKFrequentHeap(int[] nums, int k)
    {
        // Step 1: Count frequencies
        var freq = new Dictionary<int, int>();
        foreach (int num in nums)
        {
            freq[num] = freq.GetValueOrDefault(num, 0) + 1;
        }

        // Step 2: Min-heap ordered by frequency (PriorityQueue dequeues MINIMUM first)
        // PriorityQueue<TElement, TPriority> — requires .NET 6+
        var minHeap = new PriorityQueue<int, int>();

        foreach (var (num, count) in freq)
        {
            minHeap.Enqueue(num, count); // enqueue with frequency as priority

            // Evict the least frequent if heap exceeds k
            if (minHeap.Count > k)
                minHeap.Dequeue(); // removes lowest-frequency element
        }

        // Step 3: Extract all k elements
        int[] result = new int[k];
        for (int i = k - 1; i >= 0; i--)
            result[i] = minHeap.Dequeue();

        return result;
    }

    // ── Approach B: Bucket Sort — O(n) ──────────────────────────────────
    public int[] TopKFrequentBucketSort(int[] nums, int k)
    {
        // Step 1: Count frequencies
        var freq = new Dictionary<int, int>();
        foreach (int num in nums)
            freq[num] = freq.GetValueOrDefault(num, 0) + 1;

        // Step 2: Buckets indexed by frequency (max frequency = nums.Length)
        var buckets = new List<int>[nums.Length + 1];
        foreach (var (num, count) in freq)
        {
            buckets[count] ??= new List<int>(); // null-coalescing assignment (.NET 8+)
            buckets[count].Add(num);
        }

        // Step 3: Collect top k from highest frequency downward
        var result = new List<int>();
        for (int f = buckets.Length - 1; f >= 0 && result.Count < k; f--)
        {
            if (buckets[f] != null)
                result.AddRange(buckets[f]);
        }

        return result.GetRange(0, k).ToArray();
    }
}
```

### Follow-up Questions

- What if k equals the number of unique elements? (return all, any approach works)
- What if you need the top k frequent words (strings)? (same approach; for equal frequency, sort alphabetically — adjust the heap comparator)
- K Closest Points to Origin? (same min-heap pattern, priority = distance²)

---

## 7. Binary Search

### Problem Statement

Three variants:
1. **Standard:** Find target in sorted array. Return index or -1.
2. **First/Last Occurrence:** Find the first (or last) index of target in array with duplicates.
3. **Rotated Sorted Array:** Search in `[4,5,6,7,0,1,2]`.

### Examples

```
Standard:
  Input: [1,3,5,7,9,11], target=7  → 3

First Occurrence:
  Input: [1,2,2,2,3], target=2     → 1

Last Occurrence:
  Input: [1,2,2,2,3], target=2     → 3

Rotated:
  Input: [4,5,6,7,0,1,2], target=0 → 4
  Input: [4,5,6,7,0,1,2], target=3 → -1
```

### Approach

**Core idea:** Maintain `left` and `right` pointers. Compute `mid = left + (right - left) / 2` (avoids integer overflow vs `(left + right) / 2`). Narrow the search space by half each iteration.

**First occurrence:** When `nums[mid] == target`, record the index but continue searching LEFT (`right = mid - 1`).

**Last occurrence:** When `nums[mid] == target`, record the index but continue searching RIGHT (`left = mid + 1`).

**Rotated array:** One half is always sorted. Determine which half, then check if the target falls in the sorted range.

### Complexity

| Variant | Time | Space |
|---|---|---|
| All variants | O(log n) | O(1) |

### C# Solution

```csharp
using System;

/// <summary>
/// Binary Search — standard, first/last occurrence, and rotated sorted array.
/// All variants run in O(log n) time, O(1) space.
/// </summary>
public class Solution07_BinarySearch
{
    // ── Variant 1: Standard Binary Search ───────────────────────────────
    public int Search(int[] nums, int target)
    {
        int left = 0, right = nums.Length - 1;

        while (left <= right)
        {
            // Use left + (right - left) / 2 to prevent integer overflow
            int mid = left + (right - left) / 2;

            if      (nums[mid] == target) return mid;
            else if (nums[mid] <  target) left  = mid + 1; // target in right half
            else                          right = mid - 1; // target in left half
        }

        return -1; // not found
    }

    // ── Variant 2a: First Occurrence ────────────────────────────────────
    public int FindFirst(int[] nums, int target)
    {
        int left = 0, right = nums.Length - 1, result = -1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target)
            {
                result = mid;    // record candidate
                right  = mid - 1; // keep searching LEFT for earlier occurrence
            }
            else if (nums[mid] < target) left  = mid + 1;
            else                          right = mid - 1;
        }

        return result;
    }

    // ── Variant 2b: Last Occurrence ─────────────────────────────────────
    public int FindLast(int[] nums, int target)
    {
        int left = 0, right = nums.Length - 1, result = -1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target)
            {
                result = mid;   // record candidate
                left   = mid + 1; // keep searching RIGHT for later occurrence
            }
            else if (nums[mid] < target) left  = mid + 1;
            else                          right = mid - 1;
        }

        return result;
    }

    // ── Variant 3: Search in Rotated Sorted Array ────────────────────────
    // e.g. [4,5,6,7,0,1,2] — one half is always fully sorted
    public int SearchRotated(int[] nums, int target)
    {
        int left = 0, right = nums.Length - 1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) return mid;

            // Determine which half is sorted
            if (nums[left] <= nums[mid]) // left half is sorted
            {
                if (target >= nums[left] && target < nums[mid])
                    right = mid - 1; // target in sorted left half
                else
                    left = mid + 1;  // target must be in right half
            }
            else // right half is sorted
            {
                if (target > nums[mid] && target <= nums[right])
                    left = mid + 1;  // target in sorted right half
                else
                    right = mid - 1; // target must be in left half
            }
        }

        return -1;
    }
}
```

### Follow-up Questions

- Find minimum in rotated sorted array? (binary search — if `nums[mid] > nums[right]`, min is in right half)
- Search a 2D matrix? (treat as 1D — index `i/cols`, `i%cols`)
- Find peak element? (binary search — if `nums[mid] < nums[mid+1]`, peak is on the right)

---

## 8. BFS — Breadth-First Search

### Problem Statement

Three scenarios:
1. **Level-order tree traversal** — return list of node values level by level.
2. **Shortest path in grid** — find minimum steps from top-left to bottom-right (0 = open, 1 = wall).
3. **Count connected components** — number of connected groups in an undirected graph.

### Examples

```
Level-order tree [3,9,20,null,null,15,7]:
  Output: [[3],[9,20],[15,7]]

Grid shortest path:
  [[0,0,0],[0,1,0],[0,0,0]] → 5 steps

Connected components, n=5, edges=[[0,1],[1,2],[3,4]]:
  Output: 2 (components: {0,1,2} and {3,4})
```

### Approach

BFS explores nodes **level by level** using a Queue (FIFO). This guarantees the **shortest path** in unweighted graphs.

For level-order: snapshot `queue.Count` at the start of each iteration — that's exactly the number of nodes on the current level.

For shortest path: BFS naturally finds the minimum distance. Mark cells as visited before enqueuing (not after) to prevent re-processing.

### Complexity

| Problem | Time | Space |
|---|---|---|
| Level-order | O(n) | O(n) |
| Shortest path | O(rows × cols) | O(rows × cols) |
| Connected components | O(V + E) | O(V) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

// Shared tree node definition (also used in Problems 9 and 10)
public class TreeNode
{
    public int      Val;
    public TreeNode Left;
    public TreeNode Right;

    public TreeNode(int val = 0, TreeNode left = null, TreeNode right = null)
    {
        Val   = val;
        Left  = left;
        Right = right;
    }
}

/// <summary>
/// BFS — level-order traversal, shortest path in grid, connected components.
/// </summary>
public class Solution08_BFS
{
    // ── Problem A: Level-Order Tree Traversal ────────────────────────────
    public IList<IList<int>> LevelOrder(TreeNode root)
    {
        var result = new List<IList<int>>();
        if (root == null) return result;

        var queue = new Queue<TreeNode>();
        queue.Enqueue(root);

        while (queue.Count > 0)
        {
            int levelSize = queue.Count; // all nodes on this level
            var levelVals = new List<int>();

            for (int i = 0; i < levelSize; i++)
            {
                TreeNode node = queue.Dequeue();
                levelVals.Add(node.Val);

                if (node.Left  != null) queue.Enqueue(node.Left);
                if (node.Right != null) queue.Enqueue(node.Right);
            }

            result.Add(levelVals);
        }

        return result;
    }

    // ── Problem B: Shortest Path in Grid ────────────────────────────────
    // 0 = open cell, 1 = wall. Move in 4 directions.
    public int ShortestPath(int[][] grid)
    {
        int rows = grid.Length, cols = grid[0].Length;

        if (grid[0][0] == 1 || grid[rows - 1][cols - 1] == 1)
            return -1; // start or end is blocked

        int[][] dirs = { new[] { 0, 1 }, new[] { 0, -1 }, new[] { 1, 0 }, new[] { -1, 0 } };

        var visited = new bool[rows, cols];
        visited[0, 0] = true;

        // Queue stores (row, col, distance)
        var queue = new Queue<(int r, int c, int dist)>();
        queue.Enqueue((0, 0, 1));

        while (queue.Count > 0)
        {
            var (r, c, dist) = queue.Dequeue();

            if (r == rows - 1 && c == cols - 1) return dist; // reached destination

            foreach (int[] dir in dirs)
            {
                int nr = r + dir[0], nc = c + dir[1];

                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                    && !visited[nr, nc] && grid[nr][nc] == 0)
                {
                    visited[nr, nc] = true; // mark before enqueue to avoid duplicates
                    queue.Enqueue((nr, nc, dist + 1));
                }
            }
        }

        return -1; // no path exists
    }

    // ── Problem C: Count Connected Components ────────────────────────────
    public int CountComponents(int n, int[][] edges)
    {
        // Build adjacency list for undirected graph
        var adj = new List<int>[n];
        for (int i = 0; i < n; i++) adj[i] = new List<int>();

        foreach (int[] edge in edges)
        {
            adj[edge[0]].Add(edge[1]);
            adj[edge[1]].Add(edge[0]); // undirected
        }

        var  visited    = new bool[n];
        int  components = 0;

        for (int start = 0; start < n; start++)
        {
            if (visited[start]) continue; // already explored

            components++;                // found a new component
            var queue = new Queue<int>();
            queue.Enqueue(start);
            visited[start] = true;

            while (queue.Count > 0)
            {
                int node = queue.Dequeue();
                foreach (int neighbor in adj[node])
                {
                    if (!visited[neighbor])
                    {
                        visited[neighbor] = true;
                        queue.Enqueue(neighbor);
                    }
                }
            }
        }

        return components;
    }
}
```

### Follow-up Questions

- BFS vs DFS — when to prefer BFS? (shortest path in unweighted graph, level-by-level processing)
- Bidirectional BFS? (search from both source and target simultaneously — reduces average search space from O(b^d) to O(b^(d/2)))
- Word Ladder problem? (BFS where each word is a node; edges connect words differing by one letter)

---

## 9. DFS — Depth-First Search

### Problem Statement

Three scenarios:
1. **Tree DFS** — recursive and iterative pre-order traversal.
2. **Cycle detection** — detect cycle in a directed graph.
3. **Number of Islands** — count connected groups of `'1'` in a 2D grid.

### Examples

```
Number of Islands:
  Input:  [["1","1","0"],["1","0","0"],["0","0","1"]]
  Output: 2

Cycle Detection, n=3, edges=[[0,1],[1,2],[2,0]]:
  Output: true (0→1→2→0)
```

### Approach

DFS uses a **Stack** (explicit or call stack via recursion) to explore as deep as possible before backtracking.

**Cycle detection (directed graph):** Use a 3-color scheme:
- `0` = White (unvisited)
- `1` = Gray (on the current DFS path — if you visit gray, it's a cycle)
- `2` = Black (fully explored — safe)

**Number of Islands:** For each unvisited `'1'`, increment count and use DFS to "sink" the entire island (mark all connected `'1'`s as visited by changing to `'0'`).

### Complexity

| Problem | Time | Space |
|---|---|---|
| Tree DFS | O(n) | O(h) height |
| Cycle detection | O(V + E) | O(V) |
| Number of Islands | O(rows × cols) | O(rows × cols) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// DFS — recursive tree traversal, cycle detection, number of islands.
/// </summary>
public class Solution09_DFS
{
    // ── Problem A: Tree DFS Recursive ────────────────────────────────────
    public List<int> DfsTreeRecursive(TreeNode root)
    {
        var result = new List<int>();
        Explore(root, result);
        return result;

        // Local function for recursion
        static void Explore(TreeNode node, List<int> acc)
        {
            if (node == null) return;
            acc.Add(node.Val);          // pre-order: process before children
            Explore(node.Left,  acc);
            Explore(node.Right, acc);
        }
    }

    // ── Problem A: Tree DFS Iterative ────────────────────────────────────
    public List<int> DfsTreeIterative(TreeNode root)
    {
        var result = new List<int>();
        if (root == null) return result;

        var stack = new Stack<TreeNode>();
        stack.Push(root);

        while (stack.Count > 0)
        {
            TreeNode node = stack.Pop();
            result.Add(node.Val);

            // Push right first so left is processed first (LIFO order)
            if (node.Right != null) stack.Push(node.Right);
            if (node.Left  != null) stack.Push(node.Left);
        }

        return result;
    }

    // ── Problem B: Cycle Detection in Directed Graph ─────────────────────
    public bool HasCycle(int n, int[][] edges)
    {
        var adj = new List<int>[n];
        for (int i = 0; i < n; i++) adj[i] = new List<int>();
        foreach (int[] e in edges) adj[e[0]].Add(e[1]); // directed edge

        // 0=unvisited, 1=on current path (gray), 2=fully explored (black)
        int[] color = new int[n];

        bool Dfs(int node)
        {
            color[node] = 1; // gray: currently visiting

            foreach (int neighbor in adj[node])
            {
                if (color[neighbor] == 1) return true;            // back edge = cycle
                if (color[neighbor] == 0 && Dfs(neighbor)) return true; // recurse
            }

            color[node] = 2; // black: done exploring
            return false;
        }

        // Try DFS from every unvisited node (graph may be disconnected)
        for (int i = 0; i < n; i++)
            if (color[i] == 0 && Dfs(i))
                return true;

        return false;
    }

    // ── Problem C: Number of Islands ─────────────────────────────────────
    // '1' = land, '0' = water. Count connected regions of land.
    public int NumIslands(char[][] grid)
    {
        int rows    = grid.Length;
        int cols    = grid[0].Length;
        int islands = 0;

        void Sink(int r, int c)
        {
            // Base case: out of bounds or already water
            if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1')
                return;

            grid[r][c] = '0'; // mark as visited (sink the cell)

            // Explore all 4 cardinal directions
            Sink(r + 1, c);
            Sink(r - 1, c);
            Sink(r, c + 1);
            Sink(r, c - 1);
        }

        for (int r = 0; r < rows; r++)
        {
            for (int c = 0; c < cols; c++)
            {
                if (grid[r][c] == '1')
                {
                    islands++;    // found an island
                    Sink(r, c);  // sink the entire island so it's not counted again
                }
            }
        }

        return islands;
    }
}
```

### Follow-up Questions

- DFS vs BFS for shortest path? (DFS does NOT guarantee shortest path; use BFS for that)
- Topological Sort? (DFS-based — post-order collection; or BFS Kahn's algorithm with in-degree)
- Can Number of Islands be solved with Union-Find? (yes — union adjacent land cells, count distinct components)

---

## 10. Tree Traversal

### Problem Statement

Given a binary tree, implement all four traversals, both recursively and iteratively:
- **InOrder**: Left → Root → Right (produces sorted output for BST)
- **PreOrder**: Root → Left → Right (useful for serialization)
- **PostOrder**: Left → Right → Root (useful for deletion / bottom-up problems)
- **Level-Order**: Level by level (BFS)

### Examples

```
Tree:
        1
       / \
      2   3
     / \
    4   5

InOrder:    [4, 2, 5, 1, 3]
PreOrder:   [1, 2, 4, 5, 3]
PostOrder:  [4, 5, 2, 3, 1]
LevelOrder: [[1], [2, 3], [4, 5]]
```

### Approach

Recursive implementations are straightforward — just change the position of the "process" statement (before/between/after recursive calls).

Iterative InOrder is the trickiest: push nodes while going left, pop to process, then go right.

Iterative PostOrder uses a trick: reverse of "root → right → left" gives "left → right → root".

### Complexity

| Traversal | Time | Space |
|---|---|---|
| All traversals | O(n) | O(h) — h = tree height |
| Level-order | O(n) | O(w) — w = max width |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Tree Traversal — all four traversals, recursive and iterative.
/// Uses the TreeNode class defined in Problem 8.
/// </summary>
public class Solution10_TreeTraversal
{
    // ────────────────────────────────────────────────────────────────────
    // IN-ORDER: Left → Root → Right
    // ────────────────────────────────────────────────────────────────────

    public List<int> InOrderRecursive(TreeNode root)
    {
        var result = new List<int>();
        void Traverse(TreeNode node)
        {
            if (node == null) return;
            Traverse(node.Left);      // recurse left
            result.Add(node.Val);     // process root
            Traverse(node.Right);     // recurse right
        }
        Traverse(root);
        return result;
    }

    public List<int> InOrderIterative(TreeNode root)
    {
        var result  = new List<int>();
        var stack   = new Stack<TreeNode>();
        TreeNode cur = root;

        while (cur != null || stack.Count > 0)
        {
            // Drill down to the leftmost node
            while (cur != null)
            {
                stack.Push(cur);
                cur = cur.Left;
            }

            // Pop and process; then move to right subtree
            cur = stack.Pop();
            result.Add(cur.Val);
            cur = cur.Right;
        }

        return result;
    }

    // ────────────────────────────────────────────────────────────────────
    // PRE-ORDER: Root → Left → Right
    // ────────────────────────────────────────────────────────────────────

    public List<int> PreOrderRecursive(TreeNode root)
    {
        var result = new List<int>();
        void Traverse(TreeNode node)
        {
            if (node == null) return;
            result.Add(node.Val);     // process root FIRST
            Traverse(node.Left);
            Traverse(node.Right);
        }
        Traverse(root);
        return result;
    }

    public List<int> PreOrderIterative(TreeNode root)
    {
        var result = new List<int>();
        if (root == null) return result;

        var stack = new Stack<TreeNode>();
        stack.Push(root);

        while (stack.Count > 0)
        {
            TreeNode node = stack.Pop();
            result.Add(node.Val);             // process immediately on pop

            // Push right before left so left is processed first (LIFO)
            if (node.Right != null) stack.Push(node.Right);
            if (node.Left  != null) stack.Push(node.Left);
        }

        return result;
    }

    // ────────────────────────────────────────────────────────────────────
    // POST-ORDER: Left → Right → Root
    // ────────────────────────────────────────────────────────────────────

    public List<int> PostOrderRecursive(TreeNode root)
    {
        var result = new List<int>();
        void Traverse(TreeNode node)
        {
            if (node == null) return;
            Traverse(node.Left);
            Traverse(node.Right);
            result.Add(node.Val);     // process root LAST
        }
        Traverse(root);
        return result;
    }

    // Trick: PostOrder = reverse of PreOrder with Left/Right swapped
    // Process: Root → Right → Left, then reverse the list → Left → Right → Root
    public List<int> PostOrderIterative(TreeNode root)
    {
        var result = new List<int>();
        if (root == null) return result;

        var stack = new Stack<TreeNode>();
        stack.Push(root);

        while (stack.Count > 0)
        {
            TreeNode node = stack.Pop();
            result.Insert(0, node.Val); // prepend (builds list in reverse)

            // Push LEFT before RIGHT so RIGHT is processed next (opposite of PreOrder)
            if (node.Left  != null) stack.Push(node.Left);
            if (node.Right != null) stack.Push(node.Right);
        }

        return result;
    }

    // ────────────────────────────────────────────────────────────────────
    // LEVEL-ORDER (BFS)
    // ────────────────────────────────────────────────────────────────────

    public IList<IList<int>> LevelOrder(TreeNode root)
    {
        var result = new List<IList<int>>();
        if (root == null) return result;

        var queue = new Queue<TreeNode>();
        queue.Enqueue(root);

        while (queue.Count > 0)
        {
            int size  = queue.Count; // nodes at this specific level
            var level = new List<int>();

            for (int i = 0; i < size; i++)
            {
                TreeNode node = queue.Dequeue();
                level.Add(node.Val);

                if (node.Left  != null) queue.Enqueue(node.Left);
                if (node.Right != null) queue.Enqueue(node.Right);
            }

            result.Add(level);
        }

        return result;
    }
}
```

### Follow-up Questions

- Validate BST? (InOrder traversal — check that values are strictly increasing)
- Binary Tree Maximum Path Sum? (PostOrder DFS — return max gain from each subtree)
- Serialize/Deserialize Binary Tree? (PreOrder traversal with null markers)

---

## 11. Valid Parentheses

### Problem Statement

Given string `s` containing `(`, `)`, `{`, `}`, `[`, `]`, determine if the string is valid. A string is valid if every open bracket is closed in the correct order.

### Examples

```
"()"         → true
"()[]{}"     → true
"(]"         → false
"([)]"       → false
"{[]}"       → true
""           → true
```

### Approach

Use a **Stack**:
- Push every **opening** bracket.
- For every **closing** bracket: if the stack is empty or the top doesn't match the expected opener, return false. Otherwise pop.
- At the end, the stack must be empty (all openers were closed).

```
"({[]})"
→ push '(' : stack = ['(']
→ push '{' : stack = ['(', '{']
→ push '[' : stack = ['(', '{', '[']
→ ']': pop '[' — matches ✓
→ '}': pop '{' — matches ✓
→ ')': pop '(' — matches ✓
→ stack empty → true
```

### Complexity

| | Time | Space |
|---|---|---|
| Stack approach | O(n) | O(n) |

### C# Solution

```csharp
using System.Collections.Generic;

/// <summary>
/// Valid Parentheses — stack-based O(n) solution.
/// </summary>
public class Solution11_ValidParentheses
{
    public bool IsValid(string s)
    {
        var stack = new Stack<char>();

        foreach (char c in s)
        {
            if (c == '(' || c == '[' || c == '{')
            {
                stack.Push(c); // always push opening brackets
            }
            else
            {
                // Closing bracket: stack must not be empty and top must match
                if (stack.Count == 0) return false;

                char top = stack.Pop();

                if (c == ')' && top != '(') return false;
                if (c == ']' && top != '[') return false;
                if (c == '}' && top != '{') return false;
            }
        }

        return stack.Count == 0; // all openers must have been closed
    }

    // Cleaner alternative: use a dictionary for matching
    public bool IsValidCleaner(string s)
    {
        var matching = new Dictionary<char, char>
        {
            [')'] = '(',
            [']'] = '[',
            ['}'] = '{'
        };

        var stack = new Stack<char>();

        foreach (char c in s)
        {
            if (!matching.ContainsKey(c)) // it's an opening bracket
            {
                stack.Push(c);
            }
            else // it's a closing bracket
            {
                if (stack.Count == 0 || stack.Pop() != matching[c])
                    return false;
            }
        }

        return stack.Count == 0;
    }
}
```

### Follow-up Questions

- Minimum number of moves to make parentheses valid? (greedy — track open count)
- Longest valid parentheses substring? (DP or stack with index tracking)
- Generate all valid parentheses combinations? (backtracking/DFS)

---

## 12. Maximum Subarray — Kadane's Algorithm

### Problem Statement

Given integer array `nums`, find the contiguous subarray with the **largest sum** and return that sum.

### Examples

```
Input:  [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6      (subarray [4, -1, 2, 1])

Input:  [1]
Output: 1

Input:  [5, 4, -1, 7, 8]
Output: 23
```

### Approach

**Kadane's Algorithm:** At each position, the maximum subarray ending *here* is either:
- Extend the previous subarray: `currentSum + nums[i]`, OR
- Start fresh from `nums[i]` (if the previous subarray sum was negative — it drags us down)

Formally: `currentSum = max(nums[i], currentSum + nums[i])`

Track the global maximum throughout.

```
[-2, 1, -3, 4, -1, 2, 1, -5, 4]

i=0: cur=-2, max=-2
i=1: cur=max(1, -2+1)=1,    max=1
i=2: cur=max(-3, 1-3)=-2,   max=1
i=3: cur=max(4, -2+4)=4,    max=4
i=4: cur=max(-1, 4-1)=3,    max=4
i=5: cur=max(2, 3+2)=5,     max=5
i=6: cur=max(1, 5+1)=6,     max=6  ← answer
i=7: cur=max(-5, 6-5)=1,    max=6
i=8: cur=max(4, 1+4)=5,     max=6
```

### Complexity

| | Time | Space |
|---|---|---|
| Kadane's | O(n) | O(1) |

### C# Solution

```csharp
using System;

/// <summary>
/// Maximum Subarray — Kadane's Algorithm O(n) time, O(1) space.
/// </summary>
public class Solution12_MaxSubarray
{
    public int MaxSubArray(int[] nums)
    {
        int currentSum = nums[0]; // max subarray sum ending at position i
        int maxSum     = nums[0]; // global maximum seen so far

        for (int i = 1; i < nums.Length; i++)
        {
            // Either extend the current subarray or start a new one here
            currentSum = Math.Max(nums[i], currentSum + nums[i]);

            // Update global maximum
            maxSum = Math.Max(maxSum, currentSum);
        }

        return maxSum;
    }

    // Extended version: also return the subarray's start and end indices
    public (int MaxSum, int Start, int End) MaxSubArrayWithIndices(int[] nums)
    {
        int currentSum = nums[0], maxSum = nums[0];
        int start = 0, end = 0, tempStart = 0;

        for (int i = 1; i < nums.Length; i++)
        {
            if (nums[i] > currentSum + nums[i])
            {
                // Starting a new subarray here is better
                currentSum = nums[i];
                tempStart  = i;
            }
            else
            {
                currentSum += nums[i];
            }

            if (currentSum > maxSum)
            {
                maxSum = currentSum;
                start  = tempStart;
                end    = i;
            }
        }

        return (maxSum, start, end);
    }
}
```

### Follow-up Questions

- Maximum product subarray? (track both max and min at each position — negatives can become positives when multiplied)
- Circular subarray maximum sum? (max of: standard Kadane, total - min subarray)
- What if you need at least two elements in the subarray? (slight modification to the base case)

---

## 13. Coin Change (Dynamic Programming)

### Problem Statement

Given an array of coin denominations and an `amount`, find the **minimum number of coins** to make that amount. Return -1 if it's not possible.

### Examples

```
Input:  coins = [1, 2, 5], amount = 11
Output: 3    (5 + 5 + 1)

Input:  coins = [2], amount = 3
Output: -1   (impossible)

Input:  coins = [1], amount = 0
Output: 0
```

### Approach

**Bottom-up DP (Tabulation):**
- Create `dp[]` of size `amount + 1`, initialized to `amount + 1` (a sentinel "impossible" value).
- `dp[0] = 0` — zero coins needed for amount 0.
- For each amount `i` from 1 to `amount`, and for each coin: if `coin <= i`, then `dp[i] = min(dp[i], dp[i - coin] + 1)`.
- Intuition: "the minimum coins for amount `i` = minimum over all valid coins of (minimum coins for `i - coin` + 1 more coin)."

**DP Table Walkthrough (`coins=[1,2,5], amount=11`):**

```
amount: 0  1  2  3  4  5  6  7  8  9  10  11
dp:     0  1  1  2  2  1  2  2  3  3   2   3
```

- `dp[11] = 3` (5 + 5 + 1)

### Complexity

| | Time | Space |
|---|---|---|
| Bottom-up DP | O(n × amount) — n = #coins | O(amount) |

### C# Solution

```csharp
using System;

/// <summary>
/// Coin Change — bottom-up dynamic programming.
/// O(n * amount) time, O(amount) space.
/// </summary>
public class Solution13_CoinChange
{
    public int CoinChange(int[] coins, int amount)
    {
        // dp[i] = minimum coins needed to make exactly amount i
        int[] dp = new int[amount + 1];
        Array.Fill(dp, amount + 1); // sentinel: larger than any valid answer
        dp[0] = 0;                   // base case: 0 coins for amount 0

        for (int i = 1; i <= amount; i++)
        {
            foreach (int coin in coins)
            {
                if (coin <= i)
                {
                    // Use this coin + optimal solution for remaining (i - coin)
                    dp[i] = Math.Min(dp[i], dp[i - coin] + 1);
                }
            }
        }

        // If dp[amount] is still sentinel value, no solution exists
        return dp[amount] > amount ? -1 : dp[amount];
    }

    // Top-down with memoization (alternative approach)
    public int CoinChangeMemo(int[] coins, int amount)
    {
        int[] memo = new int[amount + 1];
        Array.Fill(memo, -2); // -2 = not computed, -1 = impossible

        int Dp(int rem)
        {
            if (rem == 0) return 0;
            if (rem < 0)  return -1;
            if (memo[rem] != -2) return memo[rem];

            int minCoins = int.MaxValue;
            foreach (int coin in coins)
            {
                int sub = Dp(rem - coin);
                if (sub >= 0 && sub < minCoins)
                    minCoins = sub + 1;
            }

            memo[rem] = (minCoins == int.MaxValue) ? -1 : minCoins;
            return memo[rem];
        }

        return Dp(amount);
    }
}
```

### Follow-up Questions

- Count the number of combinations (not minimum)? (change the recurrence to sum instead of min)
- Unbounded Knapsack vs 0/1 Knapsack? (coin change is unbounded — coins can be reused; loop order matters for 0/1)
- What if each coin has a quantity limit? (bounded knapsack — add another dimension)

---

## 14. Climbing Stairs (DP / Fibonacci)

### Problem Statement

You are climbing a staircase with `n` steps. You can climb 1 or 2 steps at a time. How many distinct ways can you reach the top?

### Examples

```
n=1 → 1   ([1])
n=2 → 2   ([1,1], [2])
n=3 → 3   ([1,1,1], [1,2], [2,1])
n=4 → 5   ([1,1,1,1],[1,1,2],[1,2,1],[2,1,1],[2,2])
n=5 → 8
```

### Approach

To reach step `n`, you must come from step `n-1` (take 1 step) or step `n-2` (take 2 steps). So:

```
ways(n) = ways(n-1) + ways(n-2)
```

This is exactly the Fibonacci recurrence!

- **Naive recursion:** O(2^n) — exponential due to repeated subproblems.
- **Memoization (top-down):** O(n) time, O(n) space.
- **Tabulation (bottom-up):** O(n) time, O(n) space.
- **Space-optimized:** O(n) time, **O(1) space** — only keep last two values.

### Complexity

| Approach | Time | Space |
|---|---|---|
| Naive recursion | O(2^n) | O(n) stack |
| Memoization | O(n) | O(n) |
| Tabulation | O(n) | O(n) |
| Space-optimized | O(n) | O(1) |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Climbing Stairs — three DP approaches.
/// Space-optimized is the preferred answer in interviews.
/// </summary>
public class Solution14_ClimbingStairs
{
    // ── Approach 1: Memoization (Top-down DP) ────────────────────────────
    public int ClimbStairsMemo(int n)
    {
        var memo = new Dictionary<int, int>();

        int Dp(int i)
        {
            if (i <= 1) return 1;            // base cases: 1 way for 0 or 1 step
            if (memo.ContainsKey(i)) return memo[i];

            memo[i] = Dp(i - 1) + Dp(i - 2); // can arrive from i-1 or i-2
            return memo[i];
        }

        return Dp(n);
    }

    // ── Approach 2: Tabulation (Bottom-up DP) ────────────────────────────
    public int ClimbStairsTabulation(int n)
    {
        if (n <= 1) return 1;

        int[] dp = new int[n + 1];
        dp[0] = 1; // 1 way to stand at step 0 (do nothing)
        dp[1] = 1; // 1 way to reach step 1

        for (int i = 2; i <= n; i++)
            dp[i] = dp[i - 1] + dp[i - 2]; // Fibonacci recurrence

        return dp[n];
    }

    // ── Approach 3: Space-Optimized — O(1) space ─────────────────────────
    public int ClimbStairsOptimal(int n)
    {
        if (n <= 1) return 1;

        int prev2 = 1; // dp[i-2]
        int prev1 = 1; // dp[i-1]

        for (int i = 2; i <= n; i++)
        {
            int curr = prev1 + prev2; // dp[i]
            prev2    = prev1;         // slide window
            prev1    = curr;
        }

        return prev1; // dp[n]
    }
}

/*
n=5 tabulation:
dp: [1, 1, 2, 3, 5, 8]
     0  1  2  3  4  5
Answer = 8 ✓ (Fibonacci: F(n+1))
*/
```

### Follow-up Questions

- What if you can climb 1, 2, or 3 steps? (`dp[i] = dp[i-1] + dp[i-2] + dp[i-3]`, 3 prev variables)
- House Robber? (same Fibonacci-like pattern — `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`)
- Min cost climbing stairs? (pay a cost to step on each stair; `dp[i] = cost[i] + min(dp[i-1], dp[i-2])`)

---

## 15. Find All Anagrams in a String

### Problem Statement

Given strings `s` and `p`, return a list of start indices of all `p`'s anagrams in `s`.

### Examples

```
Input:  s = "cbaebabacd", p = "abc"
Output: [0, 6]
  "cba" (idx 0) and "bac" (idx 6) are anagrams of "abc"

Input:  s = "abab", p = "ab"
Output: [0, 1, 2]
```

### Approach

**Sliding Window + Frequency Array:**
- Use an integer array of size 26 (for lowercase letters) to track character frequencies.
- Track `matches` — how many of the 26 character slots have matching frequencies between the window and `p`.
- Slide the window one character at a time:
  - Add the new right character (update frequency, adjust `matches`).
  - Remove the old left character (update frequency, adjust `matches`).
- When `matches == 26`, the window is an anagram.

This avoids recomputing the entire frequency comparison on every slide — `matches` is updated incrementally in O(1).

### Complexity

| | Time | Space |
|---|---|---|
| Sliding window | O(n) | O(1) — only 26-length arrays |

### C# Solution

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Find All Anagrams in a String — sliding window with frequency arrays.
/// O(n) time, O(1) space (fixed 26-char alphabet).
/// </summary>
public class Solution15_FindAnagrams
{
    public IList<int> FindAnagrams(string s, string p)
    {
        var result = new List<int>();
        if (s.Length < p.Length) return result;

        int[] pCount = new int[26]; // frequency of each char in p
        int[] wCount = new int[26]; // frequency of each char in current window

        // Initialize counts for pattern and first window
        for (int i = 0; i < p.Length; i++)
        {
            pCount[p[i] - 'a']++;
            wCount[s[i] - 'a']++;
        }

        // Count how many character slots currently match between window and p
        int matches = 0;
        for (int i = 0; i < 26; i++)
            if (pCount[i] == wCount[i]) matches++;

        // Slide the window across s
        for (int right = p.Length; right < s.Length; right++)
        {
            // Check if the previous window (before sliding) was an anagram
            if (matches == 26)
                result.Add(right - p.Length);

            // ── Add new character entering from the right ────────────────
            int charIn = s[right] - 'a';
            wCount[charIn]++;
            if      (wCount[charIn] == pCount[charIn])     matches++; // now matches
            else if (wCount[charIn] == pCount[charIn] + 1) matches--; // just broke match

            // ── Remove character leaving from the left ───────────────────
            int charOut = s[right - p.Length] - 'a';
            wCount[charOut]--;
            if      (wCount[charOut] == pCount[charOut])     matches++; // now matches
            else if (wCount[charOut] == pCount[charOut] - 1) matches--; // just broke match
        }

        // Check the last window
        if (matches == 26)
            result.Add(s.Length - p.Length);

        return result;
    }
}
```

### Follow-up Questions

- Permutation in String? (same problem — just return true/false instead of collecting indices)
- Minimum Window Substring? (sliding window, but track how many distinct chars are satisfied)
- Anagram groups (Group Anagrams)? (sort each word as key, group by key — O(n × k log k))

---

## Bonus: Algorithm Patterns Summary

> Quick reference: choose the right pattern before you start coding.

| Pattern | Use When | Data Structure | Example Problems |
|---|---|---|---|
| **HashMap / HashSet** | Need O(1) lookup, frequency count, duplicate detection | `Dictionary<K,V>`, `HashSet<T>` | Two Sum, Anagram Groups, Longest Substring |
| **Two Pointers** | Sorted array, find pair/triplet, palindrome, partition | Two `int` indices | Two Sum (sorted), 3Sum, Container With Most Water |
| **Sliding Window** | Subarray/substring with constraint, moving window | `left`/`right` pointers + window state | Longest Substring, Find Anagrams, Min Window |
| **Binary Search** | Sorted collection, find boundary/peak, O(log n) needed | `left`/`right` on array index | Classic search, Rotated Array, Find Peak |
| **Stack** | Matching pairs, next greater/smaller element, DFS iterative | `Stack<T>` | Valid Parentheses, Daily Temperatures, DFS |
| **Queue / BFS** | Shortest path (unweighted), level-by-level, FIFO processing | `Queue<T>` | BFS, Shortest Path, Level-Order |
| **Heap / Priority Queue** | K-th largest/smallest, streaming median, top-K | `PriorityQueue<E,P>` | Top K Frequent, K Closest Points, Median Finder |
| **Recursion + Memoization** | Tree/graph problems, overlapping subproblems | Recursive + `Dictionary` | Tree DFS, DFS with cache |
| **Dynamic Programming** | Optimal substructure + overlapping subproblems | `int[]` or `int[,]` DP table | Coin Change, Climbing Stairs, Longest Subseq |
| **Union-Find** | Connectivity, grouping, cycle detection (undirected) | `int[] parent, rank` | Number of Islands, Redundant Connection |
| **Monotonic Stack** | Next greater/smaller element, span problems | `Stack<int>` (indices) | Daily Temperatures, Largest Rectangle |

### Decision Framework

```
Is the input sorted (or can it be)?
  Yes → Binary Search O(log n) or Two Pointers O(n)

Do you need O(1) lookup?
  Yes → Dictionary / HashSet

Is the problem about a subarray/substring?
  Fixed size → Sliding Window
  Variable size with constraint → Sliding Window + shrink left

Is the problem on a tree or graph?
  Shortest path / level-by-level → BFS
  Explore all paths / detect cycle → DFS
  Both → think about which guarantees you need

Does the problem have choices that build on smaller choices (optimal substructure)?
  Overlapping subproblems? → Dynamic Programming
  No overlapping? → Greedy

Do you need K-th or top-K?
  → Heap (PriorityQueue)
```

---

## Bonus: C# Specific Tips

### Collections Performance

| Use | When |
|---|---|
| `int[]` | Fixed-size, cache-friendly, fastest iteration |
| `List<int>` | Dynamic size needed, convenient LINQ |
| `Dictionary<K,V>` | O(1) average lookup, insert, delete |
| `HashSet<T>` | O(1) membership test, deduplication |
| `Stack<T>` | LIFO — DFS, matching, undo |
| `Queue<T>` | FIFO — BFS, sliding window |
| `PriorityQueue<T,P>` | Min-heap (.NET 6+) — top-K, Dijkstra |
| `LinkedList<T>` | O(1) insert/remove at known node — rarely needed |
| `SortedSet<T>` | Sorted membership, O(log n) ops |
| `SortedDictionary<K,V>` | Sorted keys, O(log n) ops |

### Common C# Patterns in Interviews

```csharp
// Dictionary with default value — avoids KeyNotFoundException
freq[num] = freq.GetValueOrDefault(num, 0) + 1;

// Array.Sort with lambda
Array.Sort(intervals, (a, b) => a[0].CompareTo(b[0]));

// String to char frequency array
int[] count = new int[26];
foreach (char c in s) count[c - 'a']++;

// Index from end (C# 8+)
int last = arr[^1];     // same as arr[arr.Length - 1]
int[] slice = arr[1..]; // range operator

// Null-coalescing assignment (C# 8+)
buckets[freq] ??= new List<int>();

// Tuples for queue state
var queue = new Queue<(int row, int col, int dist)>();
queue.Enqueue((0, 0, 0));
var (r, c, d) = queue.Dequeue();

// StringBuilder for O(n) string building in loops
var sb = new StringBuilder();
foreach (char c in chars) sb.Append(c);
string result = sb.ToString();

// PriorityQueue (min-heap) — .NET 6+
var minHeap = new PriorityQueue<int, int>(); // <element, priority>
minHeap.Enqueue(value, priority);
int top = minHeap.Dequeue(); // removes minimum priority

// Array.Fill — initialize DP arrays
int[] dp = new int[n + 1];
Array.Fill(dp, int.MaxValue);
```

### Things That Surprise Interviewers (Good Way)

- Using `int.TryParse` and `TryGetValue` instead of checking `ContainsKey` first (avoids double lookup).
- Knowing `List<int>.Capacity` vs `Count` and pre-allocating: `new List<int>(n)`.
- Understanding `string` is immutable — string concatenation in a loop is O(n²). Always use `StringBuilder`.
- Using `Array.Sort` with `Comparison<T>` delegate instead of implementing `IComparer<T>` for one-off sorts.
- Recognizing when `SortedSet` or `SortedDictionary` saves you a sort step.
- Null-coalescing operators (`??`, `??=`) for cleaner initialization code.
- Span<T> and Memory<T> for zero-allocation slicing (advanced — mention if relevant).