# Coding, DSA & Python — Pattern-Based Interview Prep

> **Target**: Knows basic Python syntax. New to DSA.
> **Goal**: Recognize patterns, not memorize solutions. Write optimal code confidently.

## the platform-Specific Coding Questions (From Real Interviews)

**Source**: CodeJeet, Taro, AmbitionBox — confirmed the platform LeetCode problems:

| Problem | Difficulty | Topics | Pattern |
|---------|-----------|--------|---------|
| Check if Array Is Sorted and Rotated | Easy | Array | Array traversal |
| Linked List in Binary Tree | Medium | Linked List, Tree, DFS | Tree + LinkedList |
| First Missing Positive | Hard | Array, Hash Table | Hash map |

Additionally from past an implementation engineer interviews:
- Basic logic building puzzles
- Simple coding in language of your choice (Java/JS/Python)
- String manipulation
- "Approach to a problem" — they care about your THOUGHT PROCESS more than the exact answer
- Coding SPEED matters — practice implementing solutions quickly

**Practice these specific problems before the interview.**

---

## How to Use This Guide

Each pattern has:
1. **When to use** — what signals tell you to reach for this pattern
2. **Brute force first** — the obvious solution, then improve it
3. **Optimal approach** — with line-by-line explanation
4. **Complexity** — Big O analysis
5. **Python code** — runnable, with comments
6. **Variations** — same pattern, different problems

**Golden rule**: Learn the PATTERN, not the problem. When you recognize "this is a two-pointer problem," you can solve any variant.

---

# BIG O — The Language of Efficiency

## What Big O Measures

How runtime grows as input size grows. NOT exact time, but the RATE of growth.

```python
# O(1) — Constant time
# Runtime does NOT change with input size
def get_first(arr):
    return arr[0]  # always 1 operation

# O(n) — Linear time
# 10x input = 10x operations
def find_max(arr):
    max_val = arr[0]
    for x in arr:  # runs n times
        if x > max_val:
            max_val = x
    return max_val

# O(n²) — Quadratic time
# 10x input = 100x operations
def has_duplicates(arr):
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):  # nested loop
            if arr[i] == arr[j]:
                return True
    return False

# O(log n) — Logarithmic time
# 10x input = 1 extra operation
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
# With 1000 elements: ~10 checks
# With 1,000,000 elements: ~20 checks
# That's log₂(n) — incredibly efficient
```

## Cheat Sheet

| Big O | Name | Example | 100 elements | 10,000 elements |
|-------|------|---------|-------------|-----------------|
| O(1) | Constant | Array lookup by index | 1 op | 1 op |
| O(log n) | Logarithmic | Binary search | ~7 ops | ~14 ops |
| O(n) | Linear | Single loop | 100 ops | 10,000 ops |
| O(n log n) | Linearithmic | Sort (merge sort) | ~700 ops | ~140,000 ops |
| O(n²) | Quadratic | Nested loops | 10,000 ops | 100,000,000 ops |
| O(2^n) | Exponential | Fibonacci naive | Can't compute | Never finishes |

**Space complexity**: same notation but for memory. "O(n) space" = uses memory proportional to input.

---

# PATTERN 1: HASH MAP (Frequency Counter)

**Signal**: "Find duplicates," "count occurrences," "check if exists," "complement"

## Valid Anagram

```python
from collections import Counter

def is_anagram(s: str, t: str) -> bool:
    """
    Anagrams have the SAME character counts.
    "listen" → l:1, i:1, s:1, t:1, e:1, n:1
    "silent" → s:1, i:1, l:1, e:1, n:1, t:1  (same!)
    """
    # Brute force: sort both strings, compare O(n log n)
    # return sorted(s) == sorted(t)
    
    # Optimal: count characters, compare counts O(n)
    if len(s) != len(t):
        return False
    
    # Counter creates a dict: {char: count}
    return Counter(s) == Counter(t)

    # OR: manual count with array of 26 (for lowercase letters only)
    # counts = [0] * 26
    # for c in s:
    #     counts[ord(c) - ord('a')] += 1
    # for c in t:
    #     counts[ord(c) - ord('a')] -= 1
    # return all(c == 0 for c in counts)
```

**Variations**: Group Anagrams (use sorted string as hash key), Ransom Note

## First Non-Repeating Character

```python
from collections import Counter

def first_non_repeating(s: str) -> int:
    """
    Find first character that appears exactly once.
    "leetcode" → l is first non-repeating → index 0
    """
    count = Counter(s)  # O(n): count all characters
    
    for i, c in enumerate(s):  # O(n): find first with count=1
        if count[c] == 1:
            return i
    return -1
# Time: O(n), Space: O(1) (at most 26/52/256 distinct chars)
```

## Contains Duplicate

```python
def contains_duplicate(nums: list[int]) -> bool:
    """
    Does any value appear more than once?
    [1,2,3,1] → True (1 appears twice)
    """
    seen = set()
    for num in nums:
        if num in seen:  # O(1) lookup
            return True
        seen.add(num)
    return False
# Time: O(n), Space: O(n)

# ALTERNATIVE: set length comparison
# return len(nums) != len(set(nums))
```

## Two Sum

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    """
    Find TWO numbers that add up to target. Return their indices.
    [2, 7, 11, 15], target = 9 → [0, 1] (2 + 7 = 9)
    """
    # Brute force: check every pair O(n²)
    # for i in range(len(nums)):
    #     for j in range(i+1, len(nums)):
    #         if nums[i] + nums[j] == target: return [i, j]
    
    # Optimal: store each number's "needed complement" O(n)
    seen = {}  # value → index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i  # store current number for future complements
    return []
# Time: O(n), Space: O(n)
```

## Group Anagrams

```python
from collections import defaultdict

def group_anagrams(strs: list[str]) -> list[list[str]]:
    """
    Group words that are anagrams of each other.
    ["eat","tea","tan","ate","nat","bat"]
    → [["eat","tea","ate"], ["tan","nat"], ["bat"]]
    """
    groups = defaultdict(list)
    
    for s in strs:
        # Sorted string is the same for anagrams
        # "eat" → sorted → "aet"
        # "tea" → sorted → "aet" (same!)
        key = ''.join(sorted(s))
        groups[key].append(s)
    
    return list(groups.values())
# Time: O(n * k log k) where k = max word length
# Space: O(n * k)
```

## Subarray Sum Equals K

```python
from collections import defaultdict

def subarray_sum(nums: list[int], k: int) -> int:
    """
    Count subarrays whose sum equals K.
    [1, 1, 1], k = 2 → 2 ([1,1] appears twice at different starts)
    """
    # KEY INSIGHT: prefix sum + hash map
    # If prefix_sum[j] - prefix_sum[i] = k, then subarray i+1...j sums to k
    
    count = 0
    prefix_sum = 0
    sum_freq = defaultdict(int)
    sum_freq[0] = 1  # empty subarray has sum 0
    
    for num in nums:
        prefix_sum += num
        # If (prefix_sum - k) existed before, those subarrays sum to k
        count += sum_freq[prefix_sum - k]
        sum_freq[prefix_sum] += 1
    
    return count
# Time: O(n), Space: O(n)
```

---

# PATTERN 2: TWO POINTERS

**Signal**: "Sorted array," "find pair," "palindrome," "in-place modification"

## Valid Palindrome

```python
def is_palindrome(s: str) -> bool:
    """
    Is string same forwards and backwards (ignoring non-alphanumeric)?
    "A man, a plan, a canal: Panama" → True
    """
    left, right = 0, len(s) - 1
    
    while left < right:
        # Skip non-alphanumeric characters
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        
        if s[left].lower() != s[right].lower():
            return False
        
        left += 1
        right -= 1
    
    return True
# Time: O(n), Space: O(1)
```

## Reverse String

```python
def reverse_string(s: list[str]) -> None:
    """Reverse string IN PLACE (no extra memory)."""
    left, right = 0, len(s) - 1
    while left < right:
        s[left], s[right] = s[right], s[left]  # swap
        left += 1
        right -= 1
# Time: O(n), Space: O(1)
```

## Move Zeros

```python
def move_zeros(nums: list[int]) -> None:
    """
    Move all 0's to the END, preserving relative order.
    [0, 1, 0, 3, 12] → [1, 3, 12, 0, 0]
    """
    # "Snowball" technique: non_zero pointer
    non_zero = 0
    
    # Move all non-zero elements forward
    for i in range(len(nums)):
        if nums[i] != 0:
            nums[non_zero], nums[i] = nums[i], nums[non_zero]
            non_zero += 1
    
    # The rest are already zeros (moved to end during swaps)
# Time: O(n), Space: O(1)
```

## Two Sum II — Sorted

```python
def two_sum_sorted(numbers: list[int], target: int) -> list[int]:
    """
    Sorted array. Find pair summing to target. Return 1-indexed indices.
    [2, 7, 11, 15], target = 9 → [1, 2]
    """
    left, right = 0, len(numbers) - 1
    
    while left < right:
        current = numbers[left] + numbers[right]
        if current == target:
            return [left + 1, right + 1]  # 1-indexed
        elif current < target:
            left += 1  # need bigger sum
        else:
            right -= 1  # need smaller sum
    return []
# Time: O(n), Space: O(1)
```

## 3Sum

```python
def three_sum(nums: list[int]) -> list[list[int]]:
    """
    Find all unique triplets that sum to zero.
    [-1, 0, 1, 2, -1, -4] → [[-1, -1, 2], [-1, 0, 1]]
    """
    nums.sort()
    result = []
    
    for i in range(len(nums) - 2):
        # Skip duplicate first elements
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        
        # Now it's Two Sum II for the rest
        left, right = i + 1, len(nums) - 1
        target = -nums[i]  # need nums[i] + X + Y = 0, so X + Y = -nums[i]
        
        while left < right:
            total = nums[left] + nums[right]
            if total == target:
                result.append([nums[i], nums[left], nums[right]])
                
                # Skip duplicates
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                
                left += 1
                right -= 1
            elif total < target:
                left += 1
            else:
                right -= 1
    
    return result
# Time: O(n²), Space: O(1) excluding output
```

---

# PATTERN 3: SLIDING WINDOW

**Signal**: "Contiguous subarray," "longest substring with K constraint"

## Max Sum Subarray of Size K

```python
def max_sum_subarray(nums: list[int], k: int) -> int:
    """
    Find max sum of ANY contiguous subarray of size K.
    [2, 1, 5, 1, 3, 2], k = 3 → 9 (subarray [5, 1, 3])
    """
    window_sum = sum(nums[:k])  # first window
    max_sum = window_sum
    
    for i in range(k, len(nums)):
        # Slide window: remove left, add right
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)
    
    return max_sum
# Time: O(n), Space: O(1)
```

## Longest Substring Without Repeating Characters

```python
def longest_substring(s: str) -> int:
    """
    Longest substring with ALL unique characters.
    "abcabcbb" → 3 ("abc")
    """
    char_index = {}  # char → last seen index
    left = 0
    max_len = 0
    
    for right, char in enumerate(s):
        # If char already in window, move left past it
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1
        
        char_index[char] = right
        max_len = max(max_len, right - left + 1)
    
    return max_len
# Time: O(n), Space: O(min(n, alphabet_size))
```

## Sliding Window Maximum

```python
from collections import deque

def max_sliding_window(nums: list[int], k: int) -> list[int]:
    """
    Maximum value in EVERY sliding window of size K.
    [1,3,-1,-3,5,3,6,7], k = 3 → [3,3,5,5,6,7]
    """
    result = []
    dq = deque()  # stores INDICES of elements in decreasing order
    
    for i, num in enumerate(nums):
        # Remove elements outside current window
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        
        # Remove smaller elements (they'll never be max)
        while dq and nums[dq[-1]] < num:
            dq.pop()
        
        dq.append(i)
        
        # First element in deque is always max
        if i >= k - 1:
            result.append(nums[dq[0]])
    
    return result
# Time: O(n), Space: O(k) — each element added/removed at most once
```

---

# PATTERN 4: STACK

**Signal**: "Nested structures," "matching pairs," "parentheses," "monotonic"

## Valid Parentheses

```python
def is_valid(s: str) -> bool:
    """
    Do brackets close properly?
    "()[]{}" → True, "([)]" → False
    """
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    
    for char in s:
        if char in pairs:  # closing bracket
            if not stack or stack[-1] != pairs[char]:
                return False
            stack.pop()
        else:  # opening bracket
            stack.append(char)
    
    return len(stack) == 0  # all brackets must be closed
# Time: O(n), Space: O(n)
```

---

# PATTERN 5: LINKED LIST

**Signal**: "Linked list manipulation," "cycle detection"

## Reverse Linked List

```python
def reverse_list(head: ListNode) -> ListNode:
    """
    1 → 2 → 3 → None
    None ← 1 ← 2 ← 3
    """
    prev = None
    curr = head
    
    while curr:
        next_node = curr.next  # save next
        curr.next = prev       # reverse pointer
        prev = curr            # move prev forward
        curr = next_node       # move curr forward
    
    return prev  # new head
# Time: O(n), Space: O(1)

# Recursive:
def reverse_list_recursive(head: ListNode) -> ListNode:
    if not head or not head.next:
        return head
    
    new_head = reverse_list_recursive(head.next)
    head.next.next = head  # reverse: node.next.next = node
    head.next = None       # old forward = None
    
    return new_head
```

## Detect Cycle

```python
def has_cycle(head: ListNode) -> bool:
    """
    Floyd's Tortoise and Hare.
    Slow moves 1 step, fast moves 2 steps.
    If they meet: there's a cycle.
    """
    slow = fast = head
    
    while fast and fast.next:
        slow = slow.next       # 1 step
        fast = fast.next.next  # 2 steps
        if slow == fast:
            return True
    
    return False
# Time: O(n), Space: O(1)
```

---

# PATTERN 6: BINARY SEARCH

**Signal**: "Sorted array," "find in O(log n)"

## Binary Search

```python
def binary_search(nums: list[int], target: int) -> int:
    """
    Find target in sorted array. Return index or -1.
    [1, 3, 5, 7, 9], target = 5 → 2
    """
    left, right = 0, len(nums) - 1
    
    while left <= right:
        mid = (left + right) // 2
        
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1  # search right half
        else:
            right = mid - 1  # search left half
    
    return -1
# Time: O(log n), Space: O(1)
```

## Search in Rotated Sorted Array

```python
def search_rotated(nums: list[int], target: int) -> int:
    """
    [4, 5, 6, 7, 0, 1, 2] (sorted but rotated), target = 0 → 4
    """
    left, right = 0, len(nums) - 1
    
    while left <= right:
        mid = (left + right) // 2
        
        if nums[mid] == target:
            return mid
        
        # Left half is sorted
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1  # target in left half
            else:
                left = mid + 1   # target in right half
        # Right half is sorted
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1   # target in right half
            else:
                right = mid - 1  # target in left half
    
    return -1
# Time: O(log n), Space: O(1)
```

---

# PATTERN 7: BINARY TREE

**Signal**: "Tree traversal," "tree depth," "level order"

## Max Depth of Binary Tree

```python
def max_depth(root: TreeNode) -> int:
    """Maximum depth (root-to-leaf longest path)."""
    if not root:
        return 0
    
    left = max_depth(root.left)
    right = max_depth(root.right)
    
    return 1 + max(left, right)
# Time: O(n), Space: O(h) where h = height
```

## Level Order Traversal

```python
from collections import deque

def level_order(root: TreeNode) -> list[list[int]]:
    """
    BFS: process level by level.
        3
       / \
      9  20
         / \
        15  7
    → [[3], [9, 20], [15, 7]]
    """
    if not root:
        return []
    
    result = []
    queue = deque([root])
    
    while queue:
        level_size = len(queue)
        level = []
        
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        
        result.append(level)
    
    return result
# Time: O(n), Space: O(n)
```

---

# PATTERN 8: DYNAMIC PROGRAMMING

**Signal**: "Overlapping subproblems," "optimal substructure," "count ways"

## Fibonacci — The DP Template

```python
# Approach 1: Naive recursion — O(2^n)
def fib_naive(n):
    if n <= 1:
        return n
    return fib_naive(n-1) + fib_naive(n-2)  # recomputes same values!

# Approach 2: Memoization (top-down DP) — O(n)
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]

# Approach 3: Tabulation (bottom-up DP) — O(n)
def fib_tab(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# Approach 4: Space-optimized — O(n) time, O(1) space
def fib_opt(n):
    if n <= 1:
        return n
    prev2, prev1 = 0, 1
    for _ in range(2, n + 1):
        curr = prev1 + prev2
        prev2 = prev1
        prev1 = curr
    return prev1
```

## Climbing Stairs

```python
def climb_stairs(n: int) -> int:
    """
    Climb n stairs. Each step: 1 or 2 stairs. How many distinct ways?
    n = 3 → 3 ways: (1,1,1), (1,2), (2,1)
    
    KEY INSIGHT: This is EXACTLY Fibonacci!
    Ways to reach step n = ways to reach n-1 + ways to reach n-2
    """
    if n <= 2:
        return n
    
    prev2, prev1 = 1, 2  # dp[1], dp[2]
    for _ in range(3, n + 1):
        curr = prev1 + prev2
        prev2 = prev1
        prev1 = curr
    
    return prev1
# Time: O(n), Space: O(1)
```

## Coin Change

```python
def coin_change(coins: list[int], amount: int) -> int:
    """
    Minimum coins needed to make amount. Return -1 if impossible.
    coins = [1, 2, 5], amount = 11 → 3 (5 + 5 + 1)
    """
    # dp[i] = minimum coins to make amount i
    # Initialize with "impossible" value
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0  # 0 coins to make amount 0
    
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                # Try using this coin + optimal for remaining
                dp[i] = min(dp[i], dp[i - coin] + 1)
    
    return dp[amount] if dp[amount] != float('inf') else -1
# Time: O(amount × len(coins)), Space: O(amount)
```

---

# PATTERN 9: KADANE'S ALGORITHM

**Signal**: "Maximum subarray," "best time," "local vs global optimum"

## Maximum Subarray

```python
def max_subarray(nums: list[int]) -> int:
    """
    Largest sum of ANY contiguous subarray.
    [-2,1,-3,4,-1,2,1,-5,4] → 6 (subarray [4,-1,2,1])
    """
    # Kadane's insight: at each position, decide:
    # "Is it better to start fresh, or extend the previous subarray?"
    
    max_ending_here = nums[0]  # best sum ENDING at current position
    max_so_far = nums[0]        # best sum seen anywhere
    
    for i in range(1, len(nums)):
        # Either extend previous subarray or start new one
        max_ending_here = max(nums[i], max_ending_here + nums[i])
        max_so_far = max(max_so_far, max_ending_here)
    
    return max_so_far
# Time: O(n), Space: O(1)
```

## Best Time to Buy/Sell Stock

```python
def max_profit(prices: list[int]) -> int:
    """
    Buy once, sell once. Max profit.
    [7, 1, 5, 3, 6, 4] → 5 (buy at 1, sell at 6)
    """
    min_price = float('inf')
    max_profit = 0
    
    for price in prices:
        if price < min_price:
            min_price = price  # new best day to BUY
        else:
            max_profit = max(max_profit, price - min_price)  # best profit if sell TODAY
    
    return max_profit
# Time: O(n), Space: O(1)
```

---

# PATTERN 10: PREFIX PRODUCT

**Signal**: "Product of array except self," "no division allowed"

## Product of Array Except Self

```python
def product_except_self(nums: list[int]) -> list[int]:
    """
    For each position: product of ALL OTHER elements WITHOUT division.
    [1, 2, 3, 4] → [24, 12, 8, 6]
    
    Left pass:  [1,   1,   2,   6]   ← product of everything to the LEFT
    Right pass: [24,  12,  4,   1]   ← product of everything to the RIGHT
    Result:     [24,  12,  8,   6]   ← left[i] * right[i]
    """
    n = len(nums)
    result = [1] * n
    
    # Left pass: result[i] = product of elements to left of i
    left_product = 1
    for i in range(n):
        result[i] = left_product
        left_product *= nums[i]
    
    # Right pass: multiply by product of elements to right
    right_product = 1
    for i in range(n - 1, -1, -1):
        result[i] *= right_product
        right_product *= nums[i]
    
    return result
# Time: O(n), Space: O(1) excluding output
```

---

# PATTERN 11: BACKTRACKING

**Signal**: "All combinations," "all subsets," "all permutations"

## Subsets

```python
def subsets(nums: list[int]) -> list[list[int]]:
    """
    ALL possible subsets (power set).
    [1, 2, 3] → [[], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]]
    """
    result = []
    
    def backtrack(start: int, current: list[int]):
        # Add current subset (decision made so far)
        result.append(current[:])  # copy!
        
        # Try adding each remaining element
        for i in range(start, len(nums)):
            current.append(nums[i])       # choose
            backtrack(i + 1, current)     # explore
            current.pop()                  # unchoose (backtrack)
    
    backtrack(0, [])
    return result
# Time: O(2^n), Space: O(n) for recursion stack
```

---

# PATTERN 12: HEAP

**Signal**: "Top K," "kth largest/smallest," "merge K sorted"

## Top K Frequent Elements

```python
from collections import Counter
import heapq

def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """
    K most frequent elements.
    [1,1,1,2,2,3], k = 2 → [1, 2]
    """
    # Count frequencies
    count = Counter(nums)  # {1: 3, 2: 2, 3: 1}
    
    # Use min-heap of size k
    # (frequency, element) — Python heap is min-heap
    heap = []
    for num, freq in count.items():
        heapq.heappush(heap, (freq, num))
        if len(heap) > k:
            heapq.heappop(heap)  # remove least frequent
    
    return [num for freq, num in heap]
# Time: O(n log k), Space: O(n)

# ALTERNATIVE: Quickselect O(n) average, O(n²) worst
```

---

# PYTHON BUILT-INS TO MASTER

```python
# === COLLECTIONS ===

# Counter — frequency counting
from collections import Counter
c = Counter("aabbbcccdddeee")
print(c.most_common(2))  # [('e', 3), ('a', 2)]
print(c['b'])  # 3
print(c['z'])  # 0 (no KeyError!)

# defaultdict — auto-create missing keys
from collections import defaultdict
group = defaultdict(list)
group['numbers'].append(1)  # no KeyError — auto-creates empty list
group['numbers'].append(2)
print(group)  # {'numbers': [1, 2]}

# deque — O(1) pop from both ends
from collections import deque
dq = deque([1, 2, 3])
dq.append(4)       # [1,2,3,4]
dq.appendleft(0)   # [0,1,2,3,4]
dq.pop()           # 4, deque is [0,1,2,3]
dq.popleft()       # 0, deque is [1,2,3]

# === ITERATION ===

# enumerate — index + value
for i, val in enumerate(['a', 'b', 'c']):
    print(i, val)  # 0 a, 1 b, 2 c

# zip — parallel iteration
names = ['Alice', 'Bob', 'Charlie']
scores = [85, 92, 78]
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# sorted with key — custom sort
students = [('Alice', 85), ('Bob', 92), ('Charlie', 78)]
sorted(students, key=lambda x: x[1], reverse=True)
# [('Bob', 92), ('Alice', 85), ('Charlie', 78)]

# === COMPREHENSIONS ===

# List comprehension
squares = [x**2 for x in range(10)]  # [0, 1, 4, 9, ..., 81]
evens = [x for x in range(20) if x % 2 == 0]  # [0, 2, 4, ...]

# Dict comprehension
square_dict = {x: x**2 for x in range(5)}  # {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Set comprehension
unique_lengths = {len(w) for w in ["hello", "world", "hi"]}  # {2, 5}
```

---

# QUICK REFERENCE: Pattern Recognition

| See this in the problem | Use this pattern |
|------------------------|-----------------|
| "Find if exists", "Count occurrences" | **Hash Map** (Counter, set, dict) |
| "Sorted array", "Find pair" | **Two Pointers** |
| "Contiguous subarray", "Longest substring" | **Sliding Window** |
| "Maximum subarray" | **Kadane's Algorithm** |
| "Matching brackets", "Nested" | **Stack** |
| "Linked list", "Detect cycle" | **Fast + Slow Pointer** |
| "Sorted", "Find in O(log n)" | **Binary Search** |
| "Tree", "Level order" | **BFS / DFS** |
| "Count ways", "Minimum coins", "Optimal path" | **Dynamic Programming** |
| "Product except self", "No division" | **Prefix Product** |
| "All subsets", "All combinations" | **Backtracking** |
| "Top K", "Kth largest" | **Heap** |
| "Need to remember past values" | **Hash Map** |
| "In-place modification" | **Two Pointers** |
