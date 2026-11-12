# 📌 Binary search and divide-and-conquer
*November 12, 2026 · Daily Dev Insight*

## 🧠 Overview

Binary search isn't just an algorithm—it's a mindset. At its core, it embodies the divide-and-conquer paradigm: break a problem into smaller pieces, solve recursively, and combine the results. While most developers can recite "split the array in half, check the middle," the real power lies in recognizing when a problem space can be systematically halved. That leap from O(n) to O(log n) isn't just faster; it's the difference between something that scales and something that doesn't.

The beauty of divide-and-conquer extends far beyond searching sorted arrays. Think about debugging a production issue across 100,000 commits (git bisect), finding the right version of a dependency, or determining the optimal configuration value through binary search optimization. These real-world applications share a common thread: a monotonic search space where you can eliminate half your options with each decision.

What separates junior from senior engineers isn't knowing binary search exists—it's recognizing when a linear scan can be replaced with a logarithmic strategy. Can you sort first and then search repeatedly? Is your problem space actually ordered, even if not obviously? These questions unlock algorithmic improvements that make products genuinely better for users.

## 💡 Key Concepts

- **Precondition: Sorted or Monotonic Space** – Binary search requires order. Whether it's a sorted array, a monotonic function, or a searchable decision tree, you need to be able to make meaningful comparisons that eliminate half the possibilities.

- **The Invariant: Answer Always in Remaining Range** – Your loop/recursion must maintain the property that if an answer exists, it's always within your current boundaries. This is what makes the algorithm correct.

- **Off-by-one Errors Are Your Enemy** – The trickiest part of binary search is handling boundaries. Should you use `mid = (left + right) // 2` or `+ 1`? Is it `while left < right` or `left <= right`? These details matter intensely and depend on your exact use case.

- **Divide-and-Conquer vs Decrease-and-Conquer** – True divide-and-conquer (like merge sort) solves both halves and combines results. Binary search is technically "decrease-and-conquer"—you only recurse into one half. Both are powerful patterns worth recognizing.

- **Beyond Arrays: Search on Answer Spaces** – Advanced applications use binary search to find optimal numerical answers (min/max problems) by searching the space of possible solutions rather than array indices.

## 🐍 Python Example

```python
def binary_search(arr, target):
    """Classic binary search with clear boundary handling."""
    left, right = 0, len(arr) - 1
    
    while left <= right:
        # Avoid overflow: mid = (left + right) // 2 can overflow in other languages
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1  # Target is in right half
        else:
            right = mid - 1  # Target is in left half
    
    return -1  # Not found


def find_first_occurrence(arr, target):
    """Find the leftmost occurrence of target (handles duplicates)."""
    left, right = 0, len(arr) - 1
    result = -1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if arr[mid] == target:
            result = mid  # Found it, but keep searching left
            right = mid - 1
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return result


def min_capacity_to_ship_packages(weights, days):
    """Binary search on answer space: minimum ship capacity needed."""
    def can_ship_in_days(capacity):
        current_load, days_needed = 0, 1
        for weight in weights:
            if current_load + weight > capacity:
                days_needed += 1
                current_load = weight
            else:
                current_load += weight
        return days_needed <= days
    
    # Search space: [max(weights), sum(weights)]
    left, right = max(weights), sum(weights)
    
    while left < right:
        mid = left + (right - left) // 2
        if can_ship_in_days(mid):
            right = mid  # Try smaller capacity
        else:
            left = mid + 1  # Need larger capacity
    
    return left


# Demo
print(binary_search([1, 3, 5, 7, 9, 11], 7))  # 3
print(find_first_occurrence([1, 2, 2, 2, 3, 4], 2))  # 1
print(min_capacity_to_ship_packages([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 5))  # 15
```

## 🟨 JavaScript Example

```javascript
/**
 * Binary search with modern JavaScript patterns
 */
function binarySearch(arr, target) {
    let left = 0;
    let right = arr.length - 1;
    
    while (left <= right) {
        const mid = Math.floor(left + (right - left) / 2);
        
        if (arr[mid] === target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return -1;
}

/**
 * Binary search on a rotated sorted array - common interview question
 * e.g., [4, 5, 6, 7, 0, 1, 2] was originally [0, 1, 2, 4, 5, 6, 7]
 */
function searchRotated(nums, target) {
    let left = 0;
    let right = nums.length - 1;
    
    while (left <= right) {
        const mid = Math.floor(left + (right - left) / 2);
        
        if (nums[mid] === target) return mid;
        
        // Determine which half is properly sorted
        if (nums[left] <= nums[mid]) {
            // Left half is sorted
            if (nums[left] <= target && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {
            // Right half is sorted
            if (nums[mid] < target && target <= nums[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    
    return -1;
}

// Demo
console.log(binarySearch([1, 3, 5, 7, 9, 11], 7));  // 3
console.log(searchRotated([4, 5, 6, 7, 0, 1, 2], 0));  // 4
console.log(searchRotated([4, 5, 6, 7, 0, 1, 2], 3));  // -1
```

## ⚖️ When To Use / When To Avoid

**✅ Use Binary Search When:**
- Data is sorted or can be sorted once and searched multiple times
- You need O(log n) lookup performance and can't use a hash table
- Searching for a threshold value in a monotonic function
- Working with large datasets where linear search is too slow
- Implementing features like autocomplete with prefix matching (on sorted data)

**❌ Avoid Binary Search When:**
- Data is unsorted and sorting cost outweighs linear search (one-time searches)
- Using a hash map/set gives O(1) lookup and is available
- Dataset is small enough that linear search is simpler and fast enough
- Data changes frequently (maintaining sorted order is expensive)
- The search space isn't actually monotonic or ordered

## 